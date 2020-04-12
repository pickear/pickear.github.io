title: 编译golang
author: Dylan
tags:
  - raspberry
  - golang
categories:
  - 编程语言
date: 2018-09-07 15:42:00
---
### 前言
在frp 0.20.0之前的版本，并没有arm64位的版本。树莓派3b已经用上了64位的cpu，并且自己将树莓派装上了arch linux 64位的系统。无奈之下，只能用frp的源代码去编译。frp是golang写的，但golang没有提供官方的arm64的安装程序。在这里，介绍如何编译并安装arm 64位golang。编译环境是amd 64的linux环境。

### 安装编译器
在go1.4(包括go1.4.3)及更早的版本，go的编译器是用C来写的，所以go1.4及之前的版本只要在有gcc的环境下就可以编译。如果想要使用go1.4及之前的版本，拿源码直接编译就可以了。在go1.4之后的版本，编译器改用了用go语言来编写，所以要编译1.4之后的版本，就需要借助go来编译。

获得go编译器环境的方式有很多种，一种是直接下载官方已经编译好的安装文件来安装，一种是通过go1.4版本进行编译所得。

##### 一.通过安装官方编译好的安装程序
首先介绍通过下载官方编译好的安装程序进行安装go环境来获得go的编译环境，从而去编译其他版本的源码。这里的环境是在amd64的linux系统下。

1. 下载
首先下载amd64位的go安装文件

```
https://storage.googleapis.com/golang/go1.8.linux-amd64.tar.gz
```
2. 安装golang

```
tar -zxvf go1.8.linux-amd64.tar.gz -c /usr/local
mv /usr/local/go /usr/local/go1.8
```
到此，golang的编译环境就安装好了。因为这里我们只是想得到一个go的编译器而不是go的运行环境，所以不需要设置GOROOT，GOPATH，验证golang安装:

```shell
$ /usr/local/go1.8/bin/go version
```
4. 设置GOROOT_BOOTSTRAP

```shell
$ export GOROOT_BOOTSTRAP=/usr/local/go1.8
```

##### 二.通过编译go1.4
我们可以直接下载 [go1.4的源码](https://dl.google.com/go/go1.4.3.src.tar.gz)，或者下载 [go1.4-bootstrap-xxxxxxxx.tar.gz](https://dl.google.com/go/go1.4-bootstrap-20171003.tar.gz) 来编译。go1.4-bootstrap包含了go1.4的源码，并且官方会不断更新让其能支持更多新的操作系统。目前最新的bootstrap是go1.4-bootstrap-20171003.tar.gz，从文件名来看，是2017年10月3号更新过的。要获得最新的bootstrap程序，可以在 [Installing Go from source](https://golang.google.cn/doc/install/source) 里找到。

1. 下载

```
https://dl.google.com/go/go1.4-bootstrap-20171003.tar.gz
```

2. 解压编译

```shell
$ tar xvf go1.4-bootstrap-20171003.tar.gz 
$ mv go /usr/local/go
$ mv /usr/local/go /usr/local/go1.4
$ cd /usr/local/go1.4/src
$ ./make.bash
```
编译完之后，可以看到在go1.4/bin目录下生成一个go文件，验证:
```shell
$ /usr/local/go1.4/bin/go version
###显示'go version go1.4-bootstrap-20170531 linux/amd64'说明成功
```
3. 设置GOROOT_BOOTSTRAP

```shell
$ export GOROOT_BOOTSTRAP=/usr/local/go1.4
```


### 源码编译golang

首先，要保证GOROOT_BOOTSTRAP环境变量指向了go的编译器home目录下。
1. 下载源码

```
https://storage.googleapis.com/golang/go1.11.src.tar.gz
```
2. 解压

```shell
$ tar xvf go1.11.src.tar.gz
$ mv go /usr/local
```
3. 本地编译
如果我们是想本地编译本地运行，那么我们可以通过支持src目录下的make.bash脚本完成。

```shell
$ cd /usr/local/go/src
$ ./make.bash
```
编译完之后，在/usr/local/go/bin目录下会生成go文件。配置GOROOT，GOPATH环境变量就可以:

```shell
  export GOROOT=/usr/local/go
  export GOPATH=/root/gowork/go
  export PATH=$PATH:$GOROOT/bin
```
4. 交叉编译
如果我们想将源码编译后在其他环境运行，那么我们可以通过支持src目录下的bootstrap.bash脚本完成,并设置要运行的系统信息。

```shell 
$ GOOS=linux GOARCH=ppc64 ./bootstrap.bash
```
编译好之后，将会生成../../go-${GOOS}-${GOARCH}-bootstrap目录和go-${GOOS}-${GOARCH}-bootstrap.tbz文件。将该编译好的安装程序拷到对应的系统解压，然后配置好GOROOT,GOPATH环境变量即可。

ps:1.4及之前的版本src目录下没有bootstrap.bash。如果想将1.4源码交叉编译，例如将go1.4-bootstrap-20171003.tar.gz交叉编译成指定系统，然后将编译好的程序拷到指定系统下，再在该系统下载新版本的go源码进行本地编译。那么，bootstrap.bash可以在1.4之后版本的src目录下找到，或者 [这里](https://golang.google.cn/src/bootstrap.bash) 可以获得bootstrap.bash脚本。

### 附录

```
	$GOOS	$GOARCH
  android	arm
  darwin	386
  darwin	amd64
  darwin	arm
  darwin	arm64
  dragonfly	amd64
  freebsd	386
  freebsd	amd64
  freebsd	arm
  linux	386
  linux	amd64
  linux	arm
  linux	arm64
  linux	ppc64
  linux	ppc64le
  linux	mips
  linux	mipsle
  linux	mips64
  linux	mips64le
  linux	s390x
  netbsd	386
  netbsd	amd64
  netbsd	arm
  openbsd	386
  openbsd	amd64
  openbsd	arm
  plan9	386
  plan9	amd64
  solaris	amd64
  windows	386
  windows	amd64
```