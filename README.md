# DockPHPWeb

使用 Docker 搭建 Nginx、PHP、Mysql 集成环境

[记录说明](https://wordupr.com/post/R2MC7VF)

## 使用

### 1.更新系统 （Debian/Ubuntu）

```
apt update
apt upgrade
apt install screen wget
```

### 2.安装 Docker

   - [Docker 官方安装文档](https://docs.docker.com/engine/install/ubuntu/)


### 3. clone 项目

```
git clone https://github.com/wordupr/docker-php-web.git
cd docker-php-web
```

### 4.构建镜像

```
screen -S build #如果断线执行 screen -r build 可以恢复

docker compose build
```

### 5.修改 docker-compose.xml 中 Mysql 的 root 密码

### 6.启动容器

```
docker compose up -d
```

### 7.安装 phpMyadmin
为了管理数据方便安装 phpMyadmin，最好 Nginx 限制一下 ip 访问。

在 VPS shell 执行一下命令（非容器内）

1. 打开默认目录
```sh
cd www/html
```

2. 下载最新 phpMyadmin 源码，解压文件
```sh
wget https://files.phpmyadmin.net/phpMyAdmin/5.2.1/phpMyAdmin-5.2.1-all-languages.zip 
unzip phpMyAdmin-5.2.1-all-languages.zip
mv phpMyAdmin-5.2.1-all-languages phpMyAdmin
```

3. 修改配置
```sh
cd phpMyAdmin
cp config.sample.inc.php config.inc.php
nano config.inc.php
```

在 config.inc.php 修改 host 配置为 mysql（docker-compose 里面定义的服务名，docker 会自动解析为 mysql 容器的 IP）

```
$cfg['Servers'][$i]['host'] = 'mysql';
```

修改 cookie 密钥，需要填写32位字符串，推荐使用 [密码生成器](https://1password.com/zh-cn/password-generator/)

```
$cfg['blowfish_secret'] = '32位字符串';
```

### 8.新增站点

1. 新建目录

```sh
mkdir www/example.com
```

2. 配置临时的 HTTP 用于申请 SSL 证书

```sh
nano nginx/conf.d/example.com.conf
```

   内容如下
   
```
server {
    listen 80;
    listen [::]:80;

    server_name example.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        charset utf-8;
        default_type text/html;
        return 200 'Hello';
    }
}
```

3. 更新 Nginx 配置
   
```sh
docker compose exec nginx nginx -s reload
```

4. 申请证书    

```sh
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d example.com
```

5. 删除临时 example.com.conf，配置完整的站点配置
    
```
server {
  listen 80;
  listen [::]:80;
  listen 443 ssl;
  listen [::]:443 ssl;
  ssl_certificate /etc/nginx/ssl/live/example.com/fullchain.pem;
  ssl_certificate_key /etc/nginx/ssl/live/example.com/privkey.pem;
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ecdh_curve X25519:prime256v1:secp384r1:secp521r1;
  ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256;
  ssl_conf_command Ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256;
  ssl_conf_command Options PrioritizeChaCha;
  ssl_prefer_server_ciphers on;
  ssl_session_timeout 10m;
  ssl_session_cache shared:SSL:10m;
  ssl_buffer_size 2k;
  add_header Strict-Transport-Security max-age=15768000;
  ssl_stapling on;
  ssl_stapling_verify on;
  server_name example.com;
  
  index index.html index.htm index.php;
  root /var/www/example.com;
  
  # PHP engine
  location ~ \.php$ {
      fastcgi_pass php82:9000;
      # 404
      try_files                     $fastcgi_script_name =404;
      # default fastcgi_params
      include                       fastcgi_params;
      # fastcgi settings
      fastcgi_index                 index.php;
      fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  }
  
  location /.well-known/acme-challenge/ {
      root /var/www/certbot;
  }

  location ~ .*\.(gif|jpg|jpeg|png|bmp|swf|flv|mp4|ico)$ {
    expires 30d;
    access_log off;
  }
  location ~ .*\.(js|css)?$ {
    expires 7d;
    access_log off;
  }
  location ~ /(\.user\.ini|\.ht|\.git|\.svn|\.project|LICENSE|README\.md) {
    deny all;
  }
}
```

6. 再次更新 Nginx 配置
   
```sh
docker compose exec nginx nginx -s reload
```

7. 配置 crontab 计划任务，每个月月初自动刷新

```
#更新https证书
1 1 1 * * cd /{目录} && docker compose run --rm certbot renew && docker compose exec nginx nginx -s reload
```

8. 重置网站目录权限
```sh
docker compose exec php82 chown -R www-data:www-data /var/www
docker compose exec php82 find /var/www -type d -exec chmod 755 {} \;
docker compose exec php82 find /var/www -type f -exec chmod 644 {} \;
```

