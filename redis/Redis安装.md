# Redis安装

官方提供的多种安装方式，我这里采用源码安装的方式进行安装。

## 下载源文件

我们可以通过[redis/redis-hashes: Redis tarball SHA1 hashes (github.com)](https://github.com/redis/redis-hashes)该页面找到自己需要安装的安装包进行下载，我这里安装的是版本是**5.0.12**这个版本。

```uri
http://download.redis.io/releases/redis-5.0.12.tar.gz
```

## 编译redis

首先我们通过下面的命令将刚才下载的安装包进行解压：

```shell
tar -xzvf redis-5.0.12.tar.gz
```

然后进入刚刚解压的目录：

```shell
cd redis-5.0.12
```

然后进行make即可。

```shell
make
```

大多数时候执行**make**的时候没有问题，但是可能你会遇到下面两个错误：

- /bin/sh: cc: command not found

这个问题是因为你环境里面没有gcc，解决办法也很简单，直接安装gcc即可。我这里是Centos7环境，执行以下命令即可：

```shell
yum -y install gcc gcc-c++ libstdc++-devel
```

- zmalloc.h:50:31: fatal error: jemalloc/jemalloc.h: No such file or directory

这也是另外一个常见的问题，解决方法很简单，在make后面增加指定参数即可：

```shell
make MALLOC=libc
```

上面两个问题解决完成之后接下来执行安装即可。

```shell
make install
```

安装完成之后可以通过下面命令尝试启动redis服务

```shell
redis-server
```

如果执行成功则说明redis安装成功了。
