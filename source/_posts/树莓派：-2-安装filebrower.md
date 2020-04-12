title: 树莓派：(2)安装filebrower
author: Dylan
tags:
  - 树莓派
  - filebrowser
categories:
  - 技术杂粹
date: 2018-09-02 18:55:00
---
### 前言
[filebrower](https://filebrowser.github.io/) 是一个轻量的文件管理系统。可以很方便地管理硬盘上的文件。filebrower支持的平台也很多，包括linux,windows,mac。

### 安装
1. 下载
首先，将filebrower的安装包 [下载](https://github.com/filebrowser/filebrowser/releases/download/v1.10.0/linux-arm64-filebrowser.tar.gz)到树莓派上。解压后放在/mnt/hd/filebrowser目录下。

2. 修改配置文件(filebrowser.yaml)

```
port: 6868
##baseURL: /admin
noAuth: false
alternativeReCaptcha: false,
reCaptchaKey: ''
reCaptchaSecret: ''
database: "/mnt/hd/filebrowser/database.db"
log: stderr
plugin: ''
scope: "/mnt/hd/aria2/data"  ##管理某个目录
allowCommands: false
allowEdit: true
allowNew: true
commands:
- git
- svn
```
添加启动脚本start_filebrowser.sh:
```shell
#! /bin/sh
# /etc/init.d/frpc
### BEGIN INIT INFO
# Provides: frpc
# Required-Start:    $network $local_fs $remote_fs
# Required-Stop:     $network $local_fs $remote_fs
# Default-Start:     2 3 4 5
# Default-Stop:      0 1 6
# Short-Description: frpc init script.
# Description: Starts and stops frpc services.
### END INIT INFO
#VAR
RUN="/mnt/hd/filebrowser/filebrowser"
ARIA_PID=$(ps -ef | grep '/mnt/hd/filebrowser/filebrowser' | grep -v grep | awk '{print $2}')
# Carry out specific functions when asked to by the system
case "$1" in
  start)
    echo "starting script filebrowser "
    if [ -z "$ARIA_PID" ]; then
      $RUN  &
      echo "started"
    else
      echo "filebrowser already started"
    fi
    ;;
  stop)
    echo "stopping script filebrowser"
    if [ ! -z "$ARIA_PID" ]; then
      kill $ARIA_PID
    fi
    echo "OK"
    ;;
  restart)
    echo "restarting script filebrowser"
    if [ ! -z "$ARIA_PID" ]; then
      kill $ARIA_PID
    fi
    sleep 3   # TODO:Maybe need to be adjust
    $RUN &
    echo "OK"
    ;;
  status)
    if [ ! -z "$ARIA_PID" ]; then
      echo "the frpc is running with PID = "$ARIA_PID
    else
      echo "no process found for filebrowser"
    fi
    ;;
  *)
    echo "Usage: /mnt/hd/filebrowser/start_filebrowser.sh {start|stop|restart|status}"
    exit 1
    ;;
esac
exit 0
#########################################
```
启动filebrower

```shell
./start_filebrowser.sh start
```
打开浏览器，访问http://ip:6868,默认用户名密码为admin:admin。
![filebrower_login](/images/blog/filebrower_login.png)

![filebrower_index](/images/blog/filebrower_index.png)

### 映射到外网
在这里，我们也用FRP映射到外网。FRP更多内容，请看 [FRP外网穿透](http://tun.gogl.top) 或者 [FRP官方](https://github.com/fatedier/frp)。

1. 配置外网FRP服务端(frps.ini)

```
[common]
bind_addr = 0.0.0.0
bind_port = 5000
log_file = ./logs/frps.log
log_level = debug
log_max_days = 2
vhost_http_port = 5001
vhost_https_port = 5002
dashboard_port = 5003
dashboard_user = admin
dashboard_pwd = xxxxxxx
privilege_mode = false
privilege_allow_ports = 8000-9999
max_pool_count = 30
authentication_timeout = 100

[1281035284]
type = http
auth_token = 111111
bind_addr = 0.0.0.0
listen_port = 8004
custom_domains = www.xxx.com
```

2. 配置内网树莓派FRP客户端服务(frpc.ini)

```
[common]
server_addr = xxx.xxx.xxx  ##服务端ip
server_port = 5000  ##服务端ip
log_level = error
log_file = /mnt/hd/frpc/logs/frpc.log
auth_token = 111111  ##服务商设置的令牌

[1281035284]
local_ip = 127.0.0.1
local_port = 6868
```

3. 使用域名
在域名设置将域名指向外网服务商的ip,配置外网服务器的nginx，添加以下内容:

```
server {
        listen       80;
        server_name  www.xxx.com;
        root /usr/local/nginx/html;
        location / {
            proxy_pass http://127.0.0.1:5001;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header Host $host;
            client_max_body_size 10m; 
            client_body_buffer_size 128k;
            proxy_connect_timeout 90; 
            proxy_send_timeout 90; 
            proxy_read_timeout 90; 
            proxy_buffer_size 4k; 
            proxy_buffers 4 32k; 
            proxy_busy_buffers_size 64k; 
            proxy_temp_file_write_size 64k;
            access_log    /usr/local/application/nginx/logs/file.access.log;
            error_log     /usr/local/application/nginx/logs/file.error.log;
        }
    }
```
注意这里代理的是5001端口。
打开浏览器，访问www.xxx.com。