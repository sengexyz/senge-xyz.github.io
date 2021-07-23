---
lang: en
layout: post
title: "Linux 服务器Python后台运行服务(ssh断开不退出)"
date: 2021-07-12
author: "[森哥](https://twitter.com/#)"
---

通过ssh登录服务器运行一个python脚本，想让它24小时不间断运行。可是一旦我退出ssh，整个程序就断了。这是由于ssh的session特性——它本身就是一个session，连接上开启session，断开ssh连接则关闭session，关闭时所有你在这个session里运行的东西都会被中断。


网上一般说到不间断任务，一般也都会先提到这个，可以说是常规方案。
nohup一般都是Linux系统自带的，使用极其简单：



命令1（记录所有日志）： nohup python -u tornado_server.py > test.log 2>&1 &
命令2（只记录错误日志）：nohup python -u tornado_server.py >/dev/null 2>error.log 2>&1 &
命令3（不记录任何日志）：nohup python -u tornado_server.py >/dev/null 2>&1 &


、命令解释：
nohup 不挂断的意思
python tornado_server.py tornado服务的启动脚本

-u 代表程序不启用缓存，也就是把输出直接放到log中，没这个参数的话，log文件的生成会有延迟

p.log 将输出日志保存到这个log中

2>1 2与> 结合代表错误重定向，而1则代表错误重定向到一个文件1，而不代表标准输出；

2>&1 换成2>&1，&与1结合就代表标准输出了，就变成错误重定向到标准输出.

& 最后一个& ，代表该命令在后台执行

PS: 命令运行后显示进程号，还有使用了nohup之后，很多人就这样不管了，其实这样有可能在当前账户非正常退出或者结束的时候，命令还是自己结束了。所以在使用nohup命令后台运行命令之后，需要使用exit正常退出当前账户，这样才能保证命令一直在后台运行。

[1]   2880
1
二、 其它相关命令
查看 nohub 命令下运行的所有后台进程：jobs

查看后台运行的所有进程：ps -aux

查看后台运行的所有python 进程：

ps aux |grep python
#或者
ps -ef | grep python
1
2
3
停止进程 kill -9 [进程id]



