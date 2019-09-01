---
title: thrift转NodeJS mac安装
date: 2019-07-01 22:21:37
tags: NodeJs
---

这次有个需求，后端没有提供http接口，需要前端直接通过node读取thrift接口。于是先配置安装了一下环境，但是其中虽然通过wiki找到了配置方法，但是还是有一些缺漏的地方，包括互联网上其他的答案也是不是特别全面，特意记录保留。

### 一、安装依赖

```shell
> brew install boost
> brew install Bison
> brew install libtool
> brew install automake
# 如果系统没有安装openssl
> brew install openssl
> brew install pkg-config
> cp /usr/local/Cellar/pkg-config/0.29.2/share/aclocal/pkg.m4 ./aclocal/
```

### 二、下载 thrift安装包
下载地址：https://thrift.apache.org/download
我安装的是0.11.0 下载地址：http://note.youdao.com/noteshare?id=da2bb9f2163c5243341ca64dc6a4ee9e
解压thrift-0.11.0.tar.gz   

```shell
> cd thrift-0.11.0
> cp /usr/local/Cellar/pkg-config/0.29.2/share/aclocal/pkg.m4 ./aclocal/
> ./bootstrap.sh
> ./configure LDFLAGS='-L/usr/local/opt/openssl/lib' CPPFLAGS='-I/usr/local/opt/openssl/include' --without-php  --without-python --prefix=/usr/local/ --with-boost=/usr/local --with-libevent=/usr/local
```

看日志是否安装有问题，我安装时有错报
提示错误大概是 bison 版本必须是2.5或 更新版本

报错走第三步

### 三、解决错误
```shell
> brew unlink bison
> brew install bison  或 brew reinstall bison
> brew link bison --force
```
如果brew link bison --force
报错

Warning: Refusing to link macOS-provided software: bison
If you need to have bison first in your PATH run:
echo 'export PATH="/usr/local/opt/bison/bin:$PATH"' >> ~/.zshrc

For compilers to find bison you may need to set:
export LDFLAGS="-L/usr/local/opt/bison/lib"

执行：
```shell
> echo 'export PATH="/usr/local/opt/bison/bin:$PATH"' >> ~/.zshrc
> export LDFLAGS="-L/usr/local/opt/bison/lib"
```

然后执行 bison -V 看bison版本是不是2.3

如果是：

 继续执行 source ~/.zshrc

bison版本就不是2.3， 在我的机器上是3.1

### 四、安装thrift
```shell
> make   #时间会有点长。如果有错（好像是提示找不到makefile）参考第三步
> make install
```

### 五、thrift文件转node文件

```shell
如果我们有定义好的 my_file.thrift 文件，那么可以在该文件所在目录下执行：

/Users/macbook/thrift/compiler/cpp/thrift -r --gen js:node machine.thrift

就可以看到生成的相应JS文件了。
```