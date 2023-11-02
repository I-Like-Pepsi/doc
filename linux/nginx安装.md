## Nginx安装
### 依赖安装
- gcc安装
  ```
  yum install gcc-c++
  ```
- PCRE库安装
  ```
  yum install -y pcre pcre-devel
  ```
- zlib 压缩和解压缩
  ```
  yum install -y zlib zlib-devel
  ```
- 安装 SSL
  ```
  yum install -y openssl openssl-devel
  ```
### 配置安装命令

在配置安装命令之前，创建目录```mkdir /var/temp/nginx -p```

```
./configure \
    --prefix=/usr/local/nginx \
    --pid-path=/var/run/nginx/nginx.pid \
    --lock-path=/var/lock/nginx.lock \
    --error-log-path=/var/log/nginx/error.log \
    --http-log-path=/var/log/nginx/access.log \
    --with-http_gzip_static_module \
    --http-client-body-temp-path=/var/temp/nginx/client \
    --http-proxy-temp-path=/var/temp/nginx/proxy \
    --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
    --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
    --http-scgi-temp-path=/var/temp/nginx/scgi
```
编译安装：
```
make && make install
```
配置解释：
```
–prefix 指定nginx安装目录

–pid-path 指向nginx的pid

–lock-path 锁定安装文件，防止被恶意篡改或误操作

–error-log 错误日志

–http-log-path http日志

–with-http_gzip_static_module 启用gzip模块，在线实时压缩输出数据流

–http-client-body-temp-path 设定客户端请求的临时目录

–http-proxy-temp-path 设定http代理临时目录

–http-fastcgi-temp-path 设定fastcgi临时目录

–http-uwsgi-temp-path 设定uwsgi临时目录

–http-scgi-temp-path 设定scgi临时目录
```
