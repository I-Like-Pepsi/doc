# Docker开启远程访问

## docker.service配置文件

一般情况下docker.service配置文件位置在```/lib/systemd/system/docker.service```。可能不同版本会有稍许不同。

## 修改启动参数

打开配置文件，找到```ExecStart```选项，增加启动参数```-H tcp://0.0.0.0```，然后保存退出即可。增加该启动参数代表所有的远程IP都能访问Docker。

## 重启Docker服务

```shell
systemctl daemon-reload
service docker restart
```

## 测试

在浏览器中输入:```${主机地址}:2375/version```会输出内容。如果不行请注意端口是否打开。
