---
lang: cn
layout: post
title: "Linux上远程运行不间断Python服务(ssh断开不退出)"
date: 2021-07-23
author: "[码农森哥](https://twitter.com/senge26430360)"
---

通过ssh登录服务器运行一个python脚本，想让它24小时不间断运行。可是一旦退出ssh，整个程序就断了。这是由于ssh的session特性——它本身就是一个session，连接上开启session，断开ssh连接则关闭session，关闭时所有你在这个session里运行的东西都会被中断。

nohup一般都是Linux系统自带的，使用极其简单：

命令1（记录所有日志）： 
```
nohup python -u senge_server.py > test.log 2>&1 & 
```

命令2（只记录错误日志）：
```
nohup python -u senge_server.py >/dev/null 2>error.log 2>&1 & 
```

命令3（不记录任何日志）：
```
nohup python -u senge_server.py >/dev/null 2>&1 & 
```


## 命令解释：
nohup 不挂断的意思
python senge_server.py 服务的启动脚本

命令运行后显示进程号，还有使用了nohup之后，很多人就这样不管了，其实这样有可能在当前账户非正常退出或者结束的时候，命令还是自己结束了。所以在使用nohup命令后台运行命令之后，需要使用exit正常退出当前账户，这样才能保证命令一直在后台运行。


## 其它相关命令
查看 nohub 命令下运行的所有后台进程：jobs

查看后台运行的所有进程：ps -aux

查看后台运行的所有python 进程：

```
ps aux |grep python
```
或者
```
ps -ef | grep python
```

