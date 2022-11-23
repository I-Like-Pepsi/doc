# Docker安装Mysql

## 拉取镜像

```shell
docker pull mysql:5.7
```

## 运行

```shell
docker run -p 3306:3306 --name mysql -v /home/zengchao/mysql/conf:/etc/mysql/conf.d -v /home/zengchao/mysql/logs:/logs -v /home/zengchao/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=@z763757299c@ -d mysql:5.7
```

- `-p`指定的是端口映射

- `-v`挂载主机目录到docker容器

- `-e`设置参数。此处是设置的root用户的密码

- `-d`后台运行。

## Docker Hub地址

```url
https://hub.docker.com/_/mysql
```


