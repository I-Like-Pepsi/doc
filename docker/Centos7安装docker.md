# Centos7安装docker

## 卸载旧版本

```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## 安装方法

该方法是通过设置[Docker’s repositories](https://docs.docker.com/engine/install/centos/#install-using-the-repository)并从中安装，这也是官网推荐的安装方法。

### 设置repository

```shell
sudo yum install -y yum-utils
sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

### 安装Docker Engine

```shell
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

### 启动Docker

上面步骤执行完成之后，Docker就安装成功了。可以通过下面命令启动Docker

```shell
sudo systemctl start docker
```

启动之后我们可以使用hello-world来测试

```shell
sudo docker run hello-world
```


