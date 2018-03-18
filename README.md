# openresty/nginx 实践

nginx的两个很重要的优势就是反向代理和负载均衡

### 前言：

本文nginx采用目录模板如下：

+ nginx 配置文件

  将你的服务的 Nginx 配置文件目录：*/etc/nginx/sites-available/*

  目录下，然后通过软连接链到 */etc/nginx/sites-enabled/* 

+ SSL 证书

  使用 letsencrypt 签发的证书，目录位置：*/etc/letsencrypt/* 

  ssl_dhparam 目录位置：*/etc/letsencrypt/* 

  若为其他机构签发的证书可在*/etc* 下新建一个文件夹存放

+ 静态文件部署

  ```
  /mnt/var/www/<your-name>/<your-project-name>/
  ```

+ 项目部署

  ```
  /var/www/<your-name>/<your-project-name>
  ```

+ Nginx日志

  ```
  /mnt/log/nginx/<your-project-name>/<env>/
  ```

### 1. 基础知识

**反向代理**（Reverse Proxy）方式是指用代理服务器来接受 internet 上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给 internet 上请求连接的客户端，此时代理服务器对外就表现为一个反向代理服务器。

举个例子，一个用户访问 <http://www.example.com/readme>，但是 www.example.com 上并不存在 readme 页面，它是偷偷从另外一台服务器上取回来，然后作为自己的内容返回给用户。但是用户并不知情这个过程。对用户来说，就像是直接从 www.example.com 获取 readme 页面一样。这里所提到的 www.example.com 这个域名对应的服务器就设置了反向代理功能。

反向代理服务器，对于客户端而言它就像是原始服务器，并且客户端不需要进行任何特别的设置。客户端向反向代理的命名空间(name-space)中的内容发送普通请求，接着反向代理将判断向何处(原始服务器)转交请求，并将获得的内容返回给客户端，就像这些内容原本就是它自己的一样。如下图所示：

![proxy](https://moonbingbing.gitbooks.io/openresty-best-practices/content/images/proxy.png)

### 2. 实践

假如你有多台服务器（下文中分别用代号1， 2， 3， 4表示），那么你可以只暴露1号服务器，通过在1号服务器上架设反向代理，映射到2、3、4号服务器上，同时在2、3、4号服务器上设置防火墙让2、3、4号服务器只允许通过1号服务器访问

### 3. 配置

服务器的nginx.conf文件：

```nginx

#user  nobody;
worker_processes  2;

#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid        logs/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       mime.types;
    default_type  application/octet-stream;

    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    #access_log  logs/access.log  main;
    sendfile        on;
    #tcp_nopush     on;
    #keepalive_timeout  0;
    keepalive_timeout  65;
    #gzip  on;

    include /etc/nginx/sites-enabled/*;
}
```



反向代理服务器的nginx配置文件模板：

```nginx
  upstream monitor_server {
      server <server1-host>:<port>; 
      server <server2-host>:<port>;   # 可通过nginx负载均衡 
      keepalive 2000;
  }

  server {
      listen 80;
      server_name hostname;

      # redirect all http to https
      return 301 https://$host$request_uri;
  }

  server {
      listen 443 ssl;
      server_name hostname;

      ssl_certificate /etc/letsencrypt/live/hostname/fullchain.pem;
      ssl_certificate_key /etc/letsencrypt/live/hostname/privkey.pem;
      # disable SSLv2
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

      # ciphers' order matters
      ssl_ciphers "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA384:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!aNULL";

      # the Elliptic curve key used for the ECDHE cipher.
      ssl_ecdh_curve secp384r1;

      # use command line
      # openssl dhparam -out dhparam.pem 2048
      # to generate Diffie Hellman Ephemeral Parameters
      ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
        # let the server choose the cipher
      ssl_prefer_server_ciphers on;

      # turn on the OCSP Stapling and verify
      ssl_stapling on;
      ssl_stapling_verify on;

      # http compression method is not secure in https
      # opens you up to vulnerabilities like BREACH, CRIME
      gzip off;

      location ^~ /.well-known/acme-challenge/ {
          default_type "text/plain";
          root /mnt/var/www/<your-name>/hostname;
      }
    
      location / {
          proxy_redirect off;
          proxy_set_header Host $host;
          proxy_set_header X-Real-IP $remote_addr;
          proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
          proxy_pass http://monitor_server;
          proxy_set_header X-Forwarded-Proto $scheme;
      }

      access_log /mnt/log/nginx/hostname/access.log;
      error_log /mnt/log/nginx/hostname/error.log;
  }
```

原始服务器配置模板：

```nginx
 server {
       listen <port>;
       server_name hostname;
       location / {
           root /var/www/<your-name>/<project name>;
           index index.html;
       }
   
       access_log /var/log/nginx/hostname/access.log;
       error_log /var/log/nginx/hostname/error.log;
   
  }
```

### 4. 使用Let's Encrypt配置SSL证书

##### 1. 安装 Certbot

*Let's Encrypt* 证书生成不需要手动进行，官方推荐 [certbot](https://certbot.eff.org/) 这套自动化工具来实现。

- Nginx on CentOS/RHEL 7

  Certbot is packaged in EPEL (Extra Packages for Enterprise Linux). To use Certbot, you must first [enable the EPEL repository](https://fedoraproject.org/wiki/EPEL#How_can_I_use_these_extra_packages.3F). On RHEL or Oracle Linux, you must also enable the optional channel.

  > Note:
  >
  > If you are using RHEL on EC2, you can enable the optional channel by running: 
  >
  > ```shell
  > $ yum -y install yum-utils
  > $ yum-config-manager --enable rhui-REGION-rhel-server-extras rhui-REGION-rhel-server-optional
  > ```

  After doing this, you can install Certbot by running:

  ```shell
  $ sudo yum install certbot-nginx
  ```

- Nginx on Ubuntu 16.04 (xenial)

  On Ubuntu systems, the Certbot team maintains a PPA. Once you add it to your list of repositories all you'll need to do is apt-get the following packages.

  ```shell
  $ sudo apt-get update

  $ sudo apt-get install software-properties-common

  $ sudo add-apt-repository ppa:certbot/certbot

  $ sudo apt-get update

  $ sudo apt-get install python-certbot-nginx 
  ```

  Certbot's DNS plugins which can be used to automate obtaining a wildcard certificate from Let's Encrypt's ACMEv2 server are not available for your OS yet. This should change soon but if you don't want to wait, you can use these plugins now by running Certbot in Docker instead of using the instructions on this page. 

##### 2. 生成SSL证书

- 编辑配置文件：

  ```shell
  $ sudo vim /etc/letsencrypt/configs/hostname
  ```

  ```
  # 写你的域名和邮箱
  domains = hostname
  rsa-key-size = 2048
  email = your-email
  text = True

  # 把下面的路径修改为 hostname 的目录位置
  authenticator = webroot
  webroot-path = /mnt/var/www/<your-name>/<hostname>
  ```

  只需将 hostname 修改为你的域名即可，certbot 会自动在 `/mnt/var/www/<your-name>/<hostname>` 下面创建一个隐藏文件 `.well-known/acme-challenge` ，通过请求这个文件来验证 `hostname` 确实属于你。外网服务器访问 `http://hostname/.well-known/acme-challenge` ，如果访问成功则验证OK。

- 配置Nginx 进行 webroot 验证

  eg: 在`/etc/nginx/sites-available` 目录下 编辑 temp 文件

  ```nginx
  server {
     listen 80;
     server_name hostname;
   
     location ~ /.well-known {
         root /mnt/var/www/<your-name>/<hostname>;
         default_type "text/plain";
     }
  }
  ```

  设置软连接：

  ```shell
  $ cd /etc/nginx/sites-enabled     # 必须!!!
  $ sudo ln -s ../sites-available/temp temp
  $ sudo openresty -s reload        
  ```

- 生成SSL证书

  ```shell
  $ sudo certbot -c /etc/letsencrypt/configs/hostname certonly

  ## 片刻之后，看到下面内容就是成功了
  IMPORTANT NOTES:
   - Congratulations! Your certificate and chain have been saved at /etc/letsencrypt/live/hostname/fullchain.pem.
  ```

  *之后删除 之前的 temp 软连接* 

##### 3. 部署 https 反向代理

- nginx 配置文件

  在`/etc/nginx/sites-available` 目录下 编辑 hostname 文件

    模板如下：

  ```nginx
    upstream monitor_server {
        server <server-host>:<port>; 
        keepalive 2000;
    }

    server {
        listen 80;
        server_name hostname;

        # redirect all http to https
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_name hostname;

        ssl_certificate /etc/letsencrypt/live/hostname/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/hostname/privkey.pem;
        # disable SSLv2
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;

        # ciphers' order matters
        ssl_ciphers "ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES256-SHA384:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!aNULL";

        # the Elliptic curve key used for the ECDHE cipher.
        ssl_ecdh_curve secp384r1;

        # use command line
        # openssl dhparam -out dhparam.pem 2048
        # to generate Diffie Hellman Ephemeral Parameters
        ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
          # let the server choose the cipher
        ssl_prefer_server_ciphers on;

        # turn on the OCSP Stapling and verify
        ssl_stapling on;
        ssl_stapling_verify on;

        # http compression method is not secure in https
        # opens you up to vulnerabilities like BREACH, CRIME
        gzip off;

        location ^~ /.well-known/acme-challenge/ {
            default_type "text/plain";
            root /mnt/var/www/<your-name>/hostname;
        }
        location / {
          ...
        }

        access_log /mnt/log/nginx/hostname/access.log;
        error_log /mnt/log/nginx/hostname/error.log;
    }
  ```

  > 注:
  >
  > ​      如需支持HTTP2，可将http server第一行修改为 listen 443 ssl http2; 作用是启用 Nginx 的 ngx_http_v2_module 模块支持 HTTP2，Nginx 版本需要高于 1.9.5，且编译时需要设置 --with-http_v2_module。
  >
  > ssl_certificate 和 ssl_certificate_key ，分别对应 fullchain.pem 和 privkey.pem，这2个文件是之前就生成好的证书和密钥。
  >
  > ssl_dhparam 通过下面命令生成：
  >
  > ```shell
  > $ sudo openssl dhparam -out /etc/letsencrypt/ssl-dhparams.pem 2048
  > ```
  >
  > 之后
  >
  > ```shell
  > $ cd /etc/nginx/sites-enabled     # 必须!!!
  > $ sudo ln -s ../sites-available/hostname hostname 
  > $ sudo openresty -s reload
  > ```

##### 4. 设置SSL证书自动更新 

```shell
$ sudo vim /etc/systemd/system/letsencrypt.service
```

```
[Unit]
Description=Let's Encrypt renewal

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet --agree-tos
ExecStartPost=/bin/systemctl reload nginx.service
```

然后增加一个 systemd timer 来触发这个服务：

```shell
$ sudo vim /etc/systemd/system/letsencrypt.timer
```

```
[Unit]
Description=Monthly renewal of Let's Encrypt's certificates

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

启用服务，开启 timer：

```
$ sudo systemctl enable letsencrypt.timer
$ sudo systemctl start letsencrypt.timer
```

上面两条命令执行完毕后，你可以通过 `systemctl list-timers` 列出所有 systemd 定时服务。当中可以找到 `letsencrypt.timer` 并看到运行时间是明天的凌晨12点。

##### 5. 在线工具测试SSL 安全性

[Qualys SSL Labs](https://www.ssllabs.com/ssltest/index.html) 提供了全面的 SSL 安全性测试，填写你的网站域名，给自己的 HTTPS 配置打个分。