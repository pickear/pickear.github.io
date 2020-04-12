title: python2和python3共存
author: Dylan
tags:
  - pip
  - python
categories:
  - 编程语言
date: 2018-10-12 10:55:00
---
### 前言
&emsp;&emsp;python3发布了已经将近10年，由于python3和python2的不兼容，导致python2切换到python3受阻重重。python2和python3两个版本平时发展,但python2的时间表已经出来。虽然python3发布了很长时间，但现在用python2的还是比较多，并且很多系统自带的python版本还是python2。当然,python3是未来。

&emsp;&emsp;由于在这种情型下，很多时候我们的计算机需要两个版本共存。下面介绍两个版本共享时需要注意和解决的一些问题。

1. 运行程序

&emsp;&emsp;我们通常执行python程序，就用$ python xxxx.py。默认情况下，如果系统同时存在python2和python3,那么通过python这个命令执行python脚本，使用的是python2执行。如果想用python3去执行一个python脚本，那么需要$ python3 xxxx.py。

2. 安装pip

首先介绍一下给python2安装pip，具体文档可以看 [installing](https://pip.pypa.io/en/stable/installing/) 。

```shell
$ curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py
$ sudo python get-pip.py
```
从上面看到，使用的是python命令。所以，pip是安装在python2下面的。如果同时要安装在python3下:

```shell
$ sudo python3 get-pip.py
```

3. 使用pip安装模块

要想安装python的依懒模块，可以通过:

```shell
$ sudo pip install xxxx
```
默认，依懒模块是安装在了python2下。如果在python3下直接引用该模块，执行脚本会抛出找不到该模块的错误。如果要将该模块安装在python3下，就需要:

```shell
$ sudo python3 -m pip install xxxx
```

4. 更新pip
更新python2下的pip:

```shell
$ sudo pip install -U pip
```
更新python3下的pip:

```shell
$sudo python3 pip install -U pip
```