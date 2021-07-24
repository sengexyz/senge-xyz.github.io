---
lang: cn
layout: post
title: "使用 Let’s Encrypt 给 Nginx网站加密（SSL）"
date: 2021-06-30
author: "[码农森哥](https://twitter.com/senge26430360)"
---

Let’s Encrypt 是Certificate Authority机构（CA）的一个加密证书。它可以帮助我们免费获得和安装TLS/SSL证书，获得这个证书，就可以在我们的服务器上启动HTTPS加密。Let’s Encrypt 提供一个软件Certbot来简化安装步骤，可以让大多数安装步骤已经全自动化进行。Certbot已经可以在 Apache 和 Nginx 上全自动安装了。

## 安装Certbot
首先我们得现在服务器上安装 Certbot

安装Certbot 及 Nginx 插件 apt：

```
sudo apt install certbot python3-certbot-nginx
```

选择 Y 后开始安装，安装完毕我们就可以使用 Certbot 了。

接下来我们要验证一下 Certbot 是否能为Nginx 自动配置 SSL。


## 获取 SSL 证书
Certbot 提供了获得 SSL 的方法。Nginx Plugins（插件）帮助我们在必要的时候重新加载配置文件。让我们来开启这个插件。

```
sudo certbot --nginx -d example.com -d www.example.com
```

certbot与--nginx 插件一起运行，-d用于指定我们希望获得证书的域名。

如果我们是第一次运行，那么 Certbot 将提示我们输入邮箱并阅读服务条款。


```
Invalid email address: .
Enter email address (used for urgent renewal and security notices)

If you really want to skip this, you can run the client with
--register-unsafely-without-email but make sure you then backup your account key
from /etc/letsencrypt/accounts

 (Enter 'c' to cancel): nginx@kalasearch.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please read the Terms of Service at
https://letsencrypt.org/documents/LE-SA-v1.2-November-15-2017.pdf. You must
agree in order to register with the ACME server at
https://acme-v02.api.letsencrypt.org/directory
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(A)gree/(C)ancel:
```

如果这是您第一次运行certbot，将提示您输入电子邮件地址并同意服务条款。

完成此操作后，certbot开始与 Let's Encrypt 服务器通信，然后开始验证我们是否是这个域名的真正拥有者。

如果成功，certbot 会继续询问我们如何配置HTTPS。

```
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```


根据我们的需求来进行选择，然后回车。配置文件就会更新，Nginx 也会重新加载。Certbot 会显示一条消息，告诉我们整个过程已经完成，证书存储在服务器的什么位置上：

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.com/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.com/privkey.pem
   Your cert will expire on 2020-11-01. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

我们的证书已经下载并已经加载成功。我们可以使用 https:// 来进行访问。如果访问成功并且浏览器地址栏前面已经出现带锁的标志，那么说明我们的网站已经获得A级保护。


## 验证 Certbot 自动续订
我们获得的加密证书只有 90 天有效期，但不用担心，Certbot会帮我们解决续订问题，它会每天运行两次systemd监测程序来检查域名证书是否快到期。如果域名证书在近 30 天到期，它会自动续订这些域名的证书。

我们可以输入以下命令来检查systemctl的状态：

```
sudo systemctl status certbot.timer
```

```
senge@git-server:~$ sudo systemctl status certbot.timer

● certbot.timer - Run certbot twice daily
     Loaded: loaded (/lib/systemd/system/certbot.timer; enabled; vendor preset: enabled)
     Active: active (waiting) since Sat 2020-08-01 03:37:47 UTC; 46min ago
    Trigger: Sat 2020-08-01 12:54:45 UTC; 8h left
   Triggers: ● certbot.service

Aug 01 03:37:47 chuan-server systemd[1]: Started Run certbot twice daily.
```


要测试域名证书的续订过程，我们可以输入以下命令：

```
sudo certbot renew --dry-run
```

如果输出结果没有任何错误，则表明一切就绪。如果在未来，续订证书发生问题，那么也不用担心，Let's Encrypt 会通过前面几步中我们留的邮箱联系我们，通知我们域名证书即将过期。


## 总结
本文介绍如何安装 Let's Encrypt 的客户端 certbot。以及如何使用certbot 为我们的域名下载了 SSL 证书。以及学习了如何设置 certbot 让它自动帮我们更新证书。如果你对使用 Certbot 还有疑问，可以给森哥留言，也可以查看它的[官方文档](https://certbot.eff.org/docs/)。


