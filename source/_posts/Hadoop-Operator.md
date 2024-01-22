---
title: Hadoop Operator
date: 2024-01-22 22:01:48
tags:
---
项目地址：https://github.com/chriskery/hadoop-operator

Hadoop是一个框架，允许在计算机集群上使用简单的编程模型对大型数据集进行分布式处理。它被设计为从单个服务器扩展到数千台机器，每台机器都提供本地计算和存储。与依赖硬件提供高可用性不同，该库本身被设计为在应用程序层面检测和处理故障，提供在一组计算机集群上的高可用服务，其中每台计算机都可能发生故障。


Hadoop Operator 的目标是为了用于管理Kubernetes上Apache Hadoop Yarn作业的生命周期，其包括两个方面的内容：
1. 提供声明式的方式创建Hadoop集群
2. 云原生的方式运行传统的Hadoop Yarn作业


## Operator 安装
Hadoop Operator依赖cert manager组件提供的证书创建与更新功能，因此在安装Hadoop Operator之前，需要先安装cert manager，执行以下命令安装Cert Manger ：

```bash

kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml

```

执行以下命令安装Hadoop  Operator：
```bash

kubectl apply -k github.com/chriskery/hadoop-operator/manifests/default

```

## 构建第一个Hadoop Cluster

首先，创建一个YAML文件以定义名为hello-world.yaml的HadoopCluster资源。

然后将下面的片段复制并粘贴到文件中，并保存：


```yaml
apiVersion: kubecluster.org/v1alpha1
kind: HadoopCluster
metadata:
  name: hadoopcluster-sample
spec:
  yarn:
    serviceType: NodePort
```

接下来，通过运行以下命令应用hello-world.yaml：
```bash
> kubectl apply -f hello-world.yaml                                                                                   
hadoopcluster.kubecluster.org/hadoopcluster-sample created
```

现在，我们已经创建了一个Hadoop集群，运行以下命令查看它：
```bash
kubectl get hdcs
```

输出类似于：
```bash
> kubectl get hdc                                             
NAME                   AGE     STATE
hadoopcluster-sample   2m23s   Created
```

Hadoop集群运算符创建Pods以模拟物理Hadoop集群中的节点。运行以下命令查看已创建的Pods：
```bash
> kubectl get pods -o wide
NAME                                   READY   STATUS    RESTARTS   AGE     IP               NODE   NOMINATED NODE   READINESS GATES
hadoopcluster-sample-datanode-0        1/1     Running   0          96s     10.244.0.100     k8s    <none>           <none>
hadoopcluster-sample-namenode          1/1     Running   0          7m25s   10.244.0.96      k8s    <none>           <none>
hadoopcluster-sample-nodemanager-0     1/1     Running   0          5m55s   10.244.0.99      k8s    <none>           <none>
hadoopcluster-sample-resourcemanager   1/1     Running   0          6s      10.244.0.101     k8s    <none>           <none>
```

如您所见，有一个模拟NameNode、DataNode、NodeManager和ResourceManager的Pod。如果需要更多的Pods来运行更多的任务，可以调整相关的Datanodes和Nodemanagers的数量。

现在，让我们尝试登录到一个节点并运行一个MapReduce作业：

```bash
> kubectl exec -it hadoopcluster-sample-resourcemanager /bin/bash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
bash-4.2$ ls
LICENSE-binary  LICENSE.txt  NOTICE-binary  NOTICE.txt  README.txt  bin  etc  include  lib  libexec  licenses-binary  sbin  share
bash-4.2$ cd share/hadoop/mapreduce/
bash-4.2$ ls         
hadoop-mapreduce-client-app-3.3.1.jar     hadoop-mapreduce-client-hs-plugins-3.3.1.jar       hadoop-mapreduce-client-shuffle-3.3.1.jar   lib-examples
hadoop-mapreduce-client-common-3.3.1.jar  hadoop-mapreduce-client-jobclient-3.3.1-tests.jar  hadoop-mapreduce-client-uploader-3.3.1.jar  sources
hadoop-mapreduce-client-core-3.3.1.jar    hadoop-mapreduce-client-jobclient-3.3.1.jar        hadoop-mapreduce-examples-3.3.1.jar
hadoop-mapreduce-client-hs-3.3.1.jar      hadoop-mapreduce-client-nativetask-3.3.1.jar       jdiff
bash-4.2$ hadoop jar hadoop-mapreduce-examples-3.3.1.jar pi 8 1000
Number of Maps  = 8
Samples per Map = 1000
Wrote input for Map #0
Wrote input for Map #1
Wrote input for Map #2
Wrote input for Map #3
Wrote input for Map #4
Wrote input for Map #5
Wrote input for Map #6
Wrote input for Map #7
Starting Job
2024-01-11 08:28:41 INFO  DefaultNoHARMFailoverProxyProvider:64 - Connecting to ResourceManager at hadoopcluster-sample-resourcemanager/10.244.0.101:8032
2024-01-11 08:28:42 INFO  JobResourceUploader:906 - Disabling Erasure Coding for path: /tmp/hadoop-yarn/staging/hadoop/.staging/job_1704961336749_0001
2024-01-11 08:28:42 INFO  FileInputFormat:300 - Total input files to process : 8
2024-01-11 08:28:43 INFO  JobSubmitter:202 - number of splits:8
2024-01-11 08:28:43 INFO  JobSubmitter:298 - Submitting tokens for job: job_1704961336749_0001
2024-01-11 08:28:43 INFO  JobSubmitter:299 - Executing with tokens: []
......
......
Job Finished in 26.166 seconds
Estimated value of Pi is 3.14100000000000000000
bash-4.2$ 
```

## 提交Hadoop作业
![架构](https://github.com/chriskery/hadoop-operator/blob/master/docs/images/architecture.png?raw=true)
Hadoop Operator提供了一个简单的方式直接运行Yarn的作业而无需先创建一个Hadoop Cluster，将下面的片段复制并粘贴到文件中，并保存：
```yaml
apiVersion: kubecluster.org/v1alpha1
kind: HadoopApplication
metadata:
  name: hadoopapplication-sample
spec:
  mainApplicationFile: /opt/hadoop/share/hadoop/mapreduce/hadoop-mapreduce-examples-3.3.1.jar
  arguments: ["pi","20","10000000000"]
  nameNodeDirFormat: true
  executorSpec:
      image: apache/hadoop:3
      replicas: 1
      volumes:
        - name: demos
          hostPath:
            path: /root/demos
      volumeMounts:
        - name: demos
          mountPath: /root/demos
      resources:
        requests:
            cpu: 2
            memory: 4Gi
        limits:
            cpu: 2
            memory: 4Gi
```

提交hadoop作业：
```shell
> kubectl apply -f manifests/samples/hadoop_job.yaml               
hadoopapplication.kubecluster.org/hadoopapplication-sample created
```

查看hadoop作业的状态：
```shell
> kubectl get hda
NAME                       AGE   STATE
hadoopapplication-sample   67s   Running
```

Hadoop Operator会为每个作业创建一个Hadoop Cluster，并创建一个driver Pod提交作业，我们可以通过查看driver Pod的日志查看作业的运行情况：
```shell
·> kubectl get pods                                        
NAME                                       READY   STATUS    RESTARTS   AGE
hadoopapplication-sample-datanode-0        1/1     Running   0          2m45s
hadoopapplication-sample-driver            1/1     Running   0          2m32s
hadoopapplication-sample-namenode          1/1     Running   0          2m47s
hadoopapplication-sample-nodemanager-0     1/1     Running   0          2m42s
hadoopapplication-sample-resourcemanager   1/1     Running   0          2m44s


·> kubectl logs -f hadoopapplication-sample-driver                          
The value of HADOOP_CONF_DIR is: /opt/hadoop/etc/hadoop
Copying configuration files...
Copying configuration files...
cp: cannot create regular file '/hbase/conf': No such file or directory
Environment variable HADOOP_ROLE is not set to a recognized value: driver
Number of Maps  = 20
Samples per Map = 10000000000
Wrote input for Map #0
Wrote input for Map #1
Wrote input for Map #2
Wrote input for Map #3
Wrote input for Map #4
Wrote input for Map #5
Wrote input for Map #6
Wrote input for Map #7
Wrote input for Map #8
Wrote input for Map #9
Wrote input for Map #10
Wrote input for Map #11
Wrote input for Map #12
...
2024-01-22 14:09:26 INFO  Job:1641 - Job job_1705932546606_0001 running in uber mode : false
2024-01-22 14:09:26 INFO  Job:1648 -  map 0% reduce 0%
2024-01-22 14:09:44 INFO  Job:1648 -  map 3% reduce 0%
2024-01-22 14:09:45 INFO  Job:1648 -  map 7% reduce 0%
2024-01-22 14:12:15 INFO  Job:1648 -  map 8% reduce 0%
2024-01-22 14:12:16 INFO  Job:1648 -  map 10% reduce 0%
```


## Web Portal访问
Hadoop Operator会将一个Hadoop 相关的Web服务以NodePort的方式暴露出来，如果想访问Hadoop的web端口对集群和作业进行状态查看，可以访问对应的service：

```bash
> kubectl get svc | grep -i nodePort
hadoopcluster-sample-resourcemanager-nodeport   NodePort    10.107.164.217   <none>        8088:31505/TCP                        18m
```

Open a browser and access the corresponding PhyNodeIp:port and you can see a web portal like this:

![Hadoop-Web-Portal](https://github.com/chriskery/hadoop-cluster-operator/blob/master/docs/images/hadoop-web.png?raw=true)
