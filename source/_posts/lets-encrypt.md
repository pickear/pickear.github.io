title: 'Let''s Encrypt:免费的ssl证书'
author: Dylan
tags:
  - ssl
  - https
categories:
  - 技术杂粹
date: 2018-09-20 15:19:00
---
#### 什么是Let's Encrypt
我们要使用https协议发布我们的应用，就需要SSL证书，将证书配置到nginx或者tomcat里。有很多购买SSL证书的机构，比如阿里云，腾讯云。阿里云也有勉强的SSL证书申请。而Let's Encrypt，是一个免费、开放，自动化的证书颁发机构。

#### 申请证书

1. 下载certbot-auto

```shell
wget https://dl.eff.org/certbot-auto
chmod a+x certbot-auto
```

2. 申请

```shell
# 注xxx.com请根据自己的域名自行更改
./certbot-auto --server https://acme-v02.api.letsencrypt.org/directory -d "*.xxx.com" --manual --preferred-challenges dns-01 certonly
```

注意，申请通配符证书是要经过DNS认证的，按照提示，前往域名后台添加对应的DNS TXT记录。添加之后，不要心急着按回车，先执行dig xxxx.xxx.com txt （如 dig _acme-challenge.gogl.top txt）确认解析记录是否生效，生效之后再回去按回车确认。

![ssl](/images/blog/ssl.png)

到了这一步后，大功告成！！！ 证书存放在/etc/letsencrypt/live/xxx.com/里面
要续期的话，执行certbot-auto renew就可以了

注意，申请通配符证书是要经过DNS认证的，按照提示，前往域名后台添加对应的DNS TXT记录。添加之后，不要心急着按回车，先执行dig xxxx.xxx.com txt （如 dig _acme-challenge.gogl.top txt）确认解析记录是否生效，生效之后再回去按回车确认

3. ngix配置

修改ngix配置文件，添加生成的证书到配置文件里:

```
server {
            listen       80;
            server_name  tun.gogl.top;
            return 301 https://$http_host$request_uri;
    }
   server {
      listen 443;
      server_name  tun.gogl.top;
      ssl on;
      root html;
      index index.html index.htm;
      ssl_certificate   /etc/letsencrypt/live/gogl.top/fullchain.pem;
      ssl_certificate_key  /etc/letsencrypt/live/gogl.top/privkey.pem;
      ssl_trusted_certificate /etc/letsencrypt/live/gogl.top/chain.pem;
      ssl_session_timeout 5m;
      location / {
            proxy_pass http://127.0.0.1:7999;
            proxy_redirect off;
            proxy_set_header X-Real-IP $remote_addr;
            #后端的Web服务器可以通过X-Forwarded-For获取用户真实IP
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            #以下是一些反向代理的配置，可选。
            proxy_set_header Host $host;
            client_max_body_size 10m; #允许客户端请求的最大单文件字节数
            client_body_buffer_size 128k; #缓冲区代理缓冲用户端请求的最大字节数，
            proxy_connect_timeout 90; #nginx跟后端服务器连接超时时间(代理连接超时)
            proxy_send_timeout 90; #后端服务器数据回传时间(代理发送超时)
            proxy_read_timeout 90; #连接成功后，后端服务器响应时间(代理接收超时)
            proxy_buffer_size 4k; #设置代理服务器（nginx）保存用户头信息的缓冲区大小
            proxy_buffers 4 32k; #proxy_buffers缓冲区，网页平均在32k以下的设置
            proxy_busy_buffers_size 64k; #高负荷下缓冲大小（proxy_buffers*2）
            proxy_temp_file_write_size 64k;
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```