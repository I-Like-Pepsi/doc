# Docker镜像

## 命令

```shell
docker images [OPTIONS] [REPOSITORY[:TAG]]
```

## Base镜像

- 不依赖其他镜像，从scratch构建

- 其他镜像可以以他就基础进行扩展

## 镜像的分层结构

Docker支持通过扩展现有镜像，创建新的镜像。

这样做最大的好处就是**共享资源**。Docker Host只需要在磁盘中保存一份Base镜像，同时内存中也只需要加载一份Base镜像。如果多个容器共享一份基础镜像，当某个容器修改了基础镜像内容，比如/etc下的文件，其他容器的/etc文件不会被修改，修改只会限制在单个容器内。

### 可写的容器层

当容器启动时一个新的可写层被加载到镜像的顶部，这一层被称作**容器层**，容器层之下叫做**镜像层**。所有对容器的改动都只会发生在容器层中，只有容器层是可写的，容器层下面的所有镜像层都只是可读的。

镜像层可能会有很多，所有镜像层会组成一个统一的文件系统。如果不同层中有一个相同的路径文件，上层的会覆盖下层的文件，也就是说用户只能访问到上层中的文件，用户看到的是一个叠加之后的文件系统。

- 添加文件。在容器层中创建文件，新文件被添加到容器层中。

- 读取文件。Docker会从上往下依次在各镜像层中查找此文件，然后打开并读入到内存中。

- 修改文件。Docker会从上往下依次在各镜像层中查找此文件，一旦找到将其复制到容器层，然后修改。

- 删除文件。Docker会从上往下依次在各镜像层中查找此文件，找到后会在容器层中记录下此删除操作。

只有修改时才需要复制一份数据，这种特性被称作**Copy-on-Write**。

## 构建镜像

Docker提供了两种构建镜像的方法：docker commit命令和Dockerfile构建文件。

### docker commit

主要包含三个步骤：

- 运行容器

- 修改容器

- 将容器保存为新的镜像

但是docker commit 方式并不推荐使用。

### Dockerfile

Dockerfile是一个文本文件，记录了镜像构建的所有步骤。

根据Dockerfile文件执行docker build命令可以生成镜像。

```shell
docker build [OPTIONS] PATH | URL | -
```

#### 查看镜像分层

可以通过```docker history 镜像名```查看镜像分层。

```shell
[zengchao@bogon docker]$ docker history ubuntu-with-dockerfile
IMAGE          CREATED          CREATED BY                                      SIZE      COMMENT
344680191dde   27 minutes ago   /bin/sh -c echo "hello world" > 1.txt           12B       
2dc39ba059dc   2 weeks ago      /bin/sh -c #(nop)  CMD ["bash"]                 0B        
<missing>      2 weeks ago      /bin/sh -c #(nop) ADD file:a7268f82a86219801…   77.8MB    
```

#### 镜像的缓存特征

Docker会缓存已有镜像的镜像层，构建新镜像时，如果某镜像层已经存在，就直接使用，无须重新创建。

如果我们希望在构建镜像时不使用缓存，可以在docker build命令中加上```--no-cache```参数。

#### Dockerfile构建过程

- 从base镜像运行一个容器

- 执行一条命令，对容器做修改

- 执行类似docker commit的操作，生成一个新的镜像层。

- Docker再基于刚刚提交的镜像运行一个新容器。

- 重复2～4步，直到Dockerfile中的所有指令执行完毕。

#### Dockerfile常用指令

- **FROM**指定base镜像

- **MAINTAINER**设置镜像作者

- **COPY**将文件从build context复制到镜像

- **ADD**与COPY类似，从build context复制文件到镜像。不同的是，如果src是归档文件（tar、zip、tgz、xz等），文件会被自动解压到dest。

- **ENV**设置环境变量，环境变量可被后面的指令使用。

- **EXPOSE**指定容器中的进程会监听某个端口，Docker可以将该端口暴露出来。

- **VOLUME**将文件或目录声明为volume。

- **WORKDIR**为后面的RUN、CMD、ENTRYPOINT、ADD或COPY指令设置镜像中的当前工作目录。

- **RUN**在容器中运行指定的命令。

- **CMD**容器启动时运行指定的命令。Dockerfile中可以有多个CMD指令，但只有最后一个生效。CMD可以被docker run之后的参数替换。

- **ENTRYPOINT**设置容器启动时运行的命令。Dockerfile中可以有多个ENTRYPOINT指令，但只有最后一个生效。CMD或docker run之后的参数会被当作参数传递给ENTRYPOINT。

## RUN VS CMD vs ENTRYPOINT

- RUN：执行命令并创建新的镜像层，RUN经常用于安装软件包。

- CMD：设置容器启动后默认执行的命令及其参数，但CMD能够被docker run后面跟的命令行参数替换。

- ENTRYPOINT：配置容器启动时运行的命令。

### Shell和Exec格式

我们可用两种方式指定RUN、CMD和ENTRYPOINT要运行的命令：Shell格式和Exec格式，二者在使用上有细微的区别。

#### Shell格式

```shell
    <instruction> <command>
```

例如：

```shell
    RUN apt-get install python3
    CMD echo "Hello world"
    ENTRYPOINT echo "Hello world"
```

当指令执行时，shell格式底层会调用 /bin/sh -c [command]。

#### Exec格式

```shell
    <instruction> ["executable", "param1", "param2", ...]
```

例如：

```shell
    RUN ["apt-get", "install", "python3"]
    CMD ["/bin/echo", "Hello world"]
    ENTRYPOINT ["/bin/echo", "Hello world"]
```

当指令执行时，会直接调用 [command]，不会被shell解析。

CMD和ENTRYPOINT推荐使用Exec格式，因为指令可读性更强，更容易理解。RUN则两种格式都可以。

### RUN

RUN指令通常用于安装应用和软件包。RUN在当前镜像的顶部执行命令，并创建新的镜像层。

### CMD

CMD指令允许用户指定容器的默认执行的命令。此命令会在容器启动且docker run没有指定其他命令时运行。

- 此命令会在容器启动且docker run没有指定其他命令时运行。

- 如果Dockerfile中有多个CMD指令，只有最后一个CMD有效。

### ENTRYPOINT

ENTRYPOINT指令可让容器以应用程序或者服务的形式运行。ENTRYPOINT不会被忽略，一定会被执行，即使运行docker run时指定了其他命令。
