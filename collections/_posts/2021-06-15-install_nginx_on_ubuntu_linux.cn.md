---
lang: cn
layout: post
title: "在Ubuntu 20.04LTS 安装Nginx"
date: 2021-06-15
author: "[码农森哥](https://twitter.com/#)"
---

Nginx 发音 “engine x” ,是一个开源软件，高性能 HTTP 和反向代理服务器，用来在互联网上处理一些大型网站。它可以被用作独立网站服务器，负载均衡，内容缓存和针对 HTTP 和非 HTTP 的反向代理服务器。
和 Apache相比，Nginx 可以处理大量的并发连接，并且每个连接占用一个很小的内存。
接下来将如何在 Ubuntu 20.04上安装和管理 Nginx。


安装命令
sudo apt update
sudo apt install nginx

一旦安装完成，Nginx 将会自动被启动。你可以运行下面的命令来验证它：
sudo systemctl status nginx


输出类似下面这样：
● nginx.service - A high performance web server and a reverse proxy server
   Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-04-29 06:43:26 UTC; 8s ago
     Docs: man:nginx(8)
  Process: 3091 ExecStart=/usr/sbin/nginx -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
  Process: 3080 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=0/SUCCESS)
 Main PID: 3095 (nginx)
    Tasks: 2 (limit: 507)
   CGroup: /system.slice/nginx.service
           ├─3095 nginx: master process /usr/sbin/nginx -g daemon on; master_process on;
           └─3097 nginx: worker process



四、测试安装
想要测试你的新 Nginx 安装，在你的浏览器中打开http://YOUR_IP，你应该可以看到默认的 Nginx 加载页面，像下面这样：


五、管理Nginx服务
您可以以与其他任何systemd服务相同的方式管理Nginx服务。
现在您的Web服务器已启动并运行，让我们来回顾一些基本的管理命令。

要停止您的Web服务器，请键入：

sudo systemctl stop nginx
停止时要启动Web服务器，请输入：

sudo systemctl start nginx
要停止并再次启动服务，请键入：

sudo systemctl restart nginx
如果您只是简单地进行配置更改，Nginx通常可以重新加载而不会丢失连接。 为此，请输入：

sudo systemctl reload nginx
默认情况下，Nginx配置为在服务器引导时自动启动。 如果这不是您想要的，可以通过输入以下命令来禁用此行为：

sudo systemctl disable nginx
要重新启用服务以在启动时启动，您可以键入：
sudo systemctl enable nginx


六、Nginx 配置文件结构以及最佳实践
所有的 Nginx 配置文件都在/etc/nginx/目录下。
主要的 Nginx 配置文件是/etc/nginx/nginx.conf。
为每个域名创建一个独立的配置文件，便于维护服务器。你可以按照需要定义任意多的 block 文件。
Nginx 服务器配置文件被储存在/etc/nginx/sites-available目录下。在/etc/nginx/sites-enabled目录下的配置文件都将被 Nginx 使用。
最佳推荐是使用标准的命名方式。例如，如果你的域名是mydomain.com，那么配置文件应该被命名为/etc/nginx/sites-available/mydomain.com.conf
如果你在域名服务器配置块中有可重用的配置段，把这些配置段摘出来，做成一小段可重用的配置。
Nginx 日志文件(access.log 和 error.log)定位在/var/log/nginx/目录下。推荐为每个服务器配置块，配置一个不同的access和error。
你可以将你的网站根目录设置在任何你想要的地方。最常用的网站根目录位置包括：

/home/<user_name>/<site_name>
/var/www/<site_name>
/var/www/html/<site_name>
/opt/<site_name>




现在基本上就可以开机启动了，常用的命令如下：

sudo service nginx {start|stop|restart|reload|force-reload|status|configtest|rotate|upgrade}


结论
恭喜，您已在Ubuntu 18.04服务器上成功安装了Nginx。现在，您准备开始部署应用程序并将Nginx用作Web或代理服务器。如今，安全证书是所有网站的必备功能，要使用免费的Let's Encrypt SSL证书保护您的网站，您可以按照此指南在Ubuntu 18.04上使用Let's Encrypt保护Nginx 。

如果您打算在服务器上托管多个域，则可以查看​​本教程，并了解如何创建Nginx服务器块。





