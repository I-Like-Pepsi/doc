# Docker容器

## 运行容器

docker run是启动容器的方法，可用三种方式指定容器启动时执行的命令：

- CMD命令

- ENTRYPOINT指令

- 在docker run命令行中指定

### 让容器长期运行

因为容器的生命周期依赖于启动时执行的命令，只要该命令不结束，容器也就不会退出。可以加上参数 -d以后台方式启动容器。

### 进入容器的两种方法

我们经常需要进到容器里去做一些工作，比如查看日志、调试、启动其他进程等。有两种方法进入容器：attach和exec。

#### docker attach

通过docker attach可以attach到容器启动命令的终端，可通过Ctrl+p，然后Ctrl+q组合键退出attach终端。

#### docker exec

```shell
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
```

通过docker exec进入相同的容器，执行```exit```退出容器。

#### attach VS exec

- attach直接进入容器启动命令的终端，不会启动新的进程。

- exec则是在容器中打开新的终端，并且可以启动新的进程。

- 如果想直接在终端中查看启动命令的输出，用attach；其他情况使用exec。
