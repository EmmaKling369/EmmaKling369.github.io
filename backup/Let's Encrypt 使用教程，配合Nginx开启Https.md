## 一、安装Certbot客户端
```
yum install certbot
```

## 获取证书
```
certbot certonly --webroot -w /var/www/example -d example.com -d www.example.com
```
>这个命令会为 example.com 和 www.example.com 这两个域名生成一个证书，使用 --webroot 模式会在 /var/www/example 中创建 .well-known 文件夹，这个文件夹里面包含了一些验证文件，certbot 会通过访问 example.com/.well-known/acme-challenge 来验证你的域名是否绑定的这个服务器。这个命令在大多数情况下都可以满足需求，

但是有些时候我们的一些服务并没有根目录，例如一些微服务，这时候使用 --webroot 就走不通了。certbot 还有另外一种模式 --standalone ， 这种模式不需要指定网站根目录，他会自动启用服务器的443端口，来验证域名的归属。我们有其他服务（例如nginx）占用了443端口，就必须先停止这些服务，在证书生成完毕后，再启用。
```
certbot certonly --standalone -d example.com -d www.example.com
```
证书生成完毕后，我们可以在 /etc/letsencrypt/live/ 目录下看到对应域名的文件夹，里面存放了指向证书的一些快捷方式。

这时候我们的第一生成证书已经完成了，接下来就是配置我们的web服务器，启用HTTPS。

## Nginx 配置启用 HTTPS
```
server {
    listen       443    ssl; # 监听443端口并开启ssl
    server_name  vm.onelol.top; # 要用的域名

    access_log  /var/log/nginx/host.access.log  main; # 日志路径
    ssl_certificate /etc/letsencrypt/live/vm.onelol.top/fullchain.pem; # 证书公钥
    ssl_certificate_key /etc/letsencrypt/live/vm.onelol.top/privkey.pem; # 证书私钥
    location / {
    #    root   /usr/share/nginx/html;
    #    index  index.html index.htm;
        proxy_pass            http://127.0.0.1:38123; # 反向代理地址
        proxy_set_header      Host $http_host; 
        proxy_set_header      Upgrade $http_upgrade;
        proxy_http_version 1.1;
        proxy_set_header X_FORWARDED_PROTO https;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

```

## 自动更新 SSL 证书
配置完这些过后，我们的工作还没有完成。 Let’s Encrypt 提供的证书只有90天的有效期，我们必须在证书到期之前，重新获取这些证书，certbot 给我们提供了一个很方便的命令，那就是 certbot renew。 通过这个命令，他会自动检查系统内的证书，并且自动更新这些证书。 我们可以运行这个命令测试一下
```
certbot renew --dry-run
```
这是因为我的api.diamondfsd.com生成证书的时候使用的是 --standalone 模式，验证域名的时候，需要启用443端口，这个错误的意思就是要启用的端口已经被占用了。 这时候我必须把nginx先关掉，才可以成功。果然，我先运行 service nginx stop 运行这个命令，就没有报错了，所有的证书都刷新成功。

证书是90天才过期，我们只需要在过期之前执行更新操作就可以了。 这件事情就可以直接交给定时任务来完成。linux 系统上有 cron 可以来搞定这件事情。 我新建了一个文件 certbot-auto-renew-cron， 这个是一个 cron 计划，这段内容的意思就是 每隔 两个月的 凌晨 2:15 执行 更新操作。

```
15 2 * */2 * certbot renew --dry-run --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx"
```
--pre-hook 这个参数表示执行更新操作之前要做的事情，因为我有 --standalone 模式的证书，所以需要 停止 nginx 服务，解除端口占用。 --post-hook 这个参数表示执行更新操作完成后要做的事情，这里就恢复 nginx 服务的启用

最后我们用 crontab 来启动这个定时任务

crontab certbot-auto-renew-cron
至此，整个网站升级到HTTPS就完成了。 总结一下我们需要做什么

获取Let’s Encrypt 免费证书
配置Nginx开启HTTPS
定时刷新证书
