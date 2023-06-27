---
title: 利用 TTRSS 搭建 RSS 服务器
date: 2022-04-08
categories: 解决方案
tags: [RSS]
---

本文主要介绍白嫖AWS的免费云服务器，利用[Tiny Tiny RSS](https://tt-rss.org)来搭建一个RSS服务器的过程。以前我都是使用Inoreader的，但是免费版刷新太慢了，六小时都不给我刷新一次。

现在开始。创建AWS账号、创建实例的过程就跳过了，我选的系统是Ubuntu 20版本，然后就是免费选项选到最后。创建实例之前一定要注意当前的区域，最好在距离近的区域创建实例，这样延迟低一些。

然后需要进行安全组设置。默认只打开了SSH用到的22端口，这肯定是不够的。需要打开HTTP的80端口和HTTPS的443端口。另外，还需要打开181端口，协议用TCP。

接下来，就可以用SSH连接云服务器了。在创建实例的时候，还会让你顺带创建密钥，可以得到一个`pem`文件。下载到这个文件之后，要先对它执行一个命令。这个命令可以修改它的读写执行的权限（改到最少），如果不改的话，SSH的时候会提示权限太`open`，不让你连接。

``` bash
chmod 400 /path/to/pem_file
```

接下来就可以SSH了。

``` bash
ssh -i /path/to/pem_file user_name@server_address
```

`user_name`和`server_address`可以在AWS点击“连接”按钮，直接得到。如果你没改过，那么它提供的默认提示就是对的。

进入到系统之后，直接先更新一下`apt` 。

``` bash
sudo apt update
sudo apt upgrade
```

然后，安装Docker。

``` bash
curl -fsSL get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

当然，有强迫症的话，安装完了可以`rmget-docker.sh`。然后，安装Docker Compose。

``` bash
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

安装命令里带版本号，实在是拉垮，你可以到[这里](https://docs.docker.com/compose/install/)找到最新的安装命令。安装完可以用`docker-compose --version`检查一下。

Tiny Tiny RSS并不是一个基于Docker的项目，但是[Awesome TTRSS](https://ttrss.henry.wang/zh/#关于)给出了这样一个实现，感谢！

我们使用基于Docker Compose的安装方法。

``` bash
mkdir ttrss
cd ttrss
curl -fLo docker-compose.yml https://raw.githubusercontent.com/HenryQW/Awesome-TTRSS/main/docker-compose.yml
```

接下来，我们利用`nano`来修改`docker-compose.yml`。

``` bash
sudo nano docker-compose.yml
```

一共要修改三个地方。

``` yml
version: "3"
services:
  service.rss:
    image: wangqiru/ttrss:latest
    container_name: ttrss
    ports:
      - 181:80
    environment:
      - SELF_URL_PATH=http://localhost:181/ # 第一处修改，把地址改成你的域名
      - DB_PASS=strong_password # 第二处修改，这里写一个非常强的密码，因为你不需要记住它
      - PUID=1000
      - PGID=1000
    volumes:
      - feed-icons:/var/www/feed-icons/
    networks:
      - public_access
      - service_only
      - database_only
    stdin_open: true
    tty: true
    restart: always

  service.mercury: # set Mercury Parser API endpoint to `service.mercury:3000` on TTRSS plugin setting page
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    networks:
      - public_access
      - service_only
    restart: always

  service.opencc: # set OpenCC API endpoint to `service.opencc:3000` on TTRSS plugin setting page
    image: wangqiru/opencc-api-server:latest
    container_name: opencc
    environment:
      - NODE_ENV=production
    networks:
      - service_only
    restart: always

  database.postgres:
    image: postgres:13-alpine
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=strong_password # 第三处修改，改成和第二处一样的密码
    volumes:
      - ~/postgres/data/:/var/lib/postgresql/data # persist postgres data to ~/postgres/data/ on the host
    networks:
      - database_only
    restart: always

  # utility.watchtower:
  #   container_name: watchtower
  #   image: containrrr/watchtower:latest
  #   volumes:
  #     - /var/run/docker.sock:/var/run/docker.sock
  #   environment:
  #     - WATCHTOWER_CLEANUP=true
  #     - WATCHTOWER_POLL_INTERVAL=86400
  #   restart: always

volumes:
  feed-icons:

networks:
  public_access: # Provide the access for ttrss UI
  service_only: # Provide the communication network between services only
    internal: true
  database_only: # Provide the communication between ttrss and database only
    internal: true
```

然后就可以运行了。

``` bash
docker-compose up -d
```

之后想要修改`docker-compose.yml`时，先`docker-compose down`然后修改，改好后重新`docker-compose up -d`即可。注意以上命令都要在`ttrss`文件夹内执行。

然后安装Nginx和Certbot来使用HTTPS。

``` bash
sudo apt install nginx
sudo apt install certbot python3-certbot-nginx
```

然后可以启动Nginx。

``` bash
sudo systemctl start nginx
```

利用Certbot创建证书。

``` bash
sudo certbot --nginx
```

按提示输入你的域名。它还会问你是否要把所有流量都转发到HTTPS，请选择是。接下来修改Nginx配置。

``` bash
nano /etc/nginx/nginx.conf
```

在`http`段找到以下两行：

``` bash
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

把这两行注释掉，然后在这两行下面添加以下内容。请把`ttrssdev.henry.wang`替换成你的域名。

``` plaintext
upstream ttrssdev {
    server 127.0.0.1:181;
}
server {
    listen 80;
    server_name ttrssdev.henry.wang;
    return 301 https://ttrssdev.henry.wang$request_uri;
}
server {
    listen 443 ssl;
    gzip on;
    server_name  ttrssdev.henry.wang;
    ssl_certificate /etc/letsencrypt/live/ttrssdev.henry.wang/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/ttrssdev.henry.wang/privkey.pem;
    access_log /var/log/nginx/ttrssdev_access.log combined;
    error_log  /var/log/nginx/ttrssdev_error.log;
    location / {
        proxy_redirect off;
        proxy_pass http://ttrssdev;
        proxy_set_header  Host                $http_host;
        proxy_set_header  X-Real-IP           $remote_addr;
        proxy_set_header  X-Forwarded-Ssl     on;
        proxy_set_header  X-Forwarded-For     $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto   $scheme;
        proxy_set_header  X-Frame-Options     SAMEORIGIN;
        client_max_body_size        100m;
        client_body_buffer_size     128k;
        proxy_buffer_size           4k;
        proxy_buffers               4 32k;
        proxy_busy_buffers_size     64k;
        proxy_temp_file_write_size  64k;
    }
}
```

最后，重启Nginx。

``` bash
sudo systemctl restart nginx
```

然后访问你的域名即可。记得改掉admin的密码，然后在用户里创建一个普通用户来使用。
