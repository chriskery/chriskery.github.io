---
title: 'Golang: pflag包'
date: 2023-12-06 21:49:50
tags:
- golang
- pflag
categories:
- golang
---
## 简介
[pflag](https://github.com/spf13/pflag)是一个Go标准`flag`包的开箱即用的代替，实现了`POSIX/GNU`风格的flags。pflag的作用是帮助开发者定义命令行参数选项，命令行参数选项在实际开发中很常用，特别是在写工具的时候。

pflag的使用非常广泛，近些年大火的Kubernetes的大量组件的都是用的pflag代替Go原有的flag包。例如Kubelet：
```golang

// NewKubeletCommand creates a *cobra.Command object with default parameters
func NewKubeletCommand() *cobra.Command {
	cleanFlagSet := pflag.NewFlagSet(componentKubelet, pflag.ContinueOnError)
	cleanFlagSet.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)
	kubeletFlags := options.NewKubeletFlags()
}
```

## 安装

```shell
go get github.com/spf13/pflag
```

## 定义参数
`pflag`完全兼容Go原生的flag包，因此只需简单的设置pflag，所有旧的代码都无需进行更改。
```golang
import flag "github.com/spf13/pflag"
```

以下是一个更全的使用Demo:

```golang
package main

import flag "github.com/spf13/pflag"
import (
    "fmt"
    "strings"
)

// 定义命令行参数对应的变量
var cliName = flag.StringP("name", "n", "nick", "Input Your Name")
var cliAge = flag.IntP("age", "a",22, "Input Your Age")
var cliGender = flag.StringP("gender", "g","male", "Input Your Gender")
var cliOK = flag.BoolP("ok", "o", false, "Input Are You OK")
var cliDes = flag.StringP("des-detail", "d", "", "Input Description")
var cliOldFlag = flag.StringP("badflag", "b", "just for test", "Input badflag")

func main() {
    
    // 为 age 参数设置 NoOptDefVal
    flag.Lookup("age").NoOptDefVal = "25"

    // 把 badflag 参数标记为即将废弃的，请用户使用 des-detail 参数
    flag.CommandLine.MarkDeprecated("badflag", "please use --des-detail instead")
    // 把 badflag 参数的 shorthand 标记为即将废弃的，请用户使用 des-detail 的 shorthand 参数
    flag.CommandLine.MarkShorthandDeprecated("badflag", "please use -d instead")

    // 在帮助文档中隐藏参数 gender
    flag.CommandLine.MarkHidden("badflag")

    // 把用户传递的命令行参数解析为对应变量的值
    flag.Parse()

    fmt.Println("name=", *cliName)
    fmt.Println("age=", *cliAge)
    fmt.Println("gender=", *cliGender)
    fmt.Println("ok=", *cliOK)
    fmt.Println("des=", *cliDes)
}
```
与`flag`包不同，在pflag中，`-`和`--`代表的是不同的含义。`-`代表一系列flags的缩写，`--`代表完整的选项。所有的缩写都必须为boolean flags或者是具有默认值的参数。


定义好所有的flags后，需要调用`flag.Parse()`把命令行参数解析到定义好的flags里面。

## 使用
### 语法
pflag命令行参数支持三种语法：
```shell
--flag    // boolean flags, or flags with no option default values
--flag x  // only on flags without a default value
--flag=x
```

- --flag写法只支持boolean或者没有可选默认值的flag
- --flag x写法只支持没有默认值的flag

pflag遇到`--`解析将会终止，例如：
```shell
go build main.go
./main -A 1 -- -B 2
```
-A 会被正常解析为1，-B则不会被解析，`--`之后的剩余参数会被保存在flag中，可以通过Args/NArg/Arg等方法访问。


### 标准化选项的名称
`pflag`提供了`NormalizeFunc`可以实现标准化选项的功能。例如我们定义了一个flag为my-flag，用户在使用的时候写成了--my_flag=3，我们希望这也能够正常工作，就通过 pflag 提供的 SetNormalizeFunc 功能轻松的解决这个问题。
```golang
func wordSepNormalizeFunc(f *flag.FlagSet, name string) flag.NormalizedName {
    from := []string{"-", "_"}
    to := "."
    for _, sep := range from {
        name = strings.Replace(name, sep, to, -1)
    }
    return flag.NormalizedName(name)
}
func main() {
    // 设置标准化参数名称的函数
    flag.CommandLine.SetNormalizeFunc(wordSepNormalizeFunc)
}
```

## FlagSet
`FlagSet`类型允许定义独立的标志集，例如在命令行界面中实现子命令。FlagSet的方法类似于命令行标志集的顶级函数。