---
lang: cn
layout: post
title: "为Prerender配置Nginx中间件文件"
date: 2022-02-07
author: "[码农森哥](https://twitter.com/sengexyz)"
---

Linux Ubuntu操作系统中，Nginx的默认配置文件在 

```
/etc/nginx/nginx.conf
```


## 修改文件
在文件中添加如下内容

```
http {
server {
        listen       80;#默认端口是80，如果端口没被占用可以不用修改
        server_name  www.*****.com ****.com;
        root        /opt/coinUnitWebH5/dist;#vue项目的打包后的dist
        index index.html;
        location / {
            #try_files $uri $uri/ @prerender;#需要指向下面的@否则会出现vue的路由在nginx中刷新出现404
            try_files $uri @prerender;
	    index  index.html index.htm;
        }
        #对应上面的@，主要原因是路由的路径资源并不是一个真实的路径，所以无法找到具体的文件
        #因此需要rewrite到index.html中，然后交给路由在处理请求资源
	location @prerender {
        	proxy_set_header X-Prerender-Token 官网上注册得到的token;
        	set $prerender 0;
        	if ($http_user_agent ~* "googlebot|bingbot|yandex|baiduspider|twitterbot|facebookexternalhit|rogerbot|linkedinbot|embedly|quora link preview|showyoubot|outbrain|pinterest\/0\.|pinterestbot|slackbot|vkShare|W3C_Validator|whatsapp") {
           		set $prerender 1;
       		}
        	if ($args ~ "_escaped_fragment_") {
            		set $prerender 1;
        	}
        	if ($http_user_agent ~ "Prerender") {
            		set $prerender 0;
        	}
       		if ($uri ~* "\.(js|css|xml|less|png|jpg|jpeg|gif|pdf|doc|txt|ico|rss|zip|mp3|rar|exe|wmv|doc|avi|ppt|mpg|mpeg|tif|wav|mov|psd|ai|xls|mp4|m4a|swf|dat|dmg|iso|flv|m4v|torrent|ttf|woff|svg|eot)") {
            		set $prerender 0;
        	}
        	#resolve using Google's DNS server to force DNS resolution and prevent caching of IPs
        	resolver  8.8.8.8;
        	if ($prerender = 1) {
            		#setting prerender as a variable forces DNS resolution since nginx caches IPs and doesnt play well with load balancing
            		#set $prerender "service.prerender.io";
            		set $prerender "127.0.0.1:3000";
			          rewrite .* /$scheme://$host$request_uri? break;
            		proxy_pass http://$prerender;
        	}
        	if ($prerender = 0) {
            		#rewrite .* /index.html break;
			          rewrite ^.*$ /index.html last;
        	}
    	}
    }
}
```

## 检查nginx并重启

```
nginx -t
service nginx restart
```


## 通过forever运行prerender
```
forever start -l prerender.log --spinSleepTime 5000 --minUptime 5000 prerender/server.js
```



