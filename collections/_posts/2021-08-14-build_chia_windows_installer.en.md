---
lang: en
layout: post
title: "Windows编译生成Chia安装包"
date: 2021-08-14
author: "[码农森哥](https://twitter.com/senge26430360)"
---

很有很多挖矿的兄弟来询问如何在Windows下编译Chia的安装包。对于一些基于Chia的分叉币，本教程也适用。森哥现在总结如下：

### 预备工具
1. powershell 工具
2. python 3.7 或以上
3. Visual Studio 2017 或以上
4. Visual Studio Code (可选)
5. 电子签名（可选）

### 清除以往编译文件

如果有以往的编译文件或目录存在，会影响编译。需要先清除历史目录文件

在chia-blockchain文件加下，输入以下命令进行清除：

```batch
rmdir venv /s /q
rmdir build_scripts\build /s /q
rmdir build_scripts\dist /s /q
rmdir build_scripts\win_build /s /q
rmdir chia-blockchain-gui\build /s /q
rmdir chia-blockchain-gui\daemon /s /q
rmdir chia-blockchain-gui\node_modules /s /q
rmdir chia-blockchain-gui\Chia-win32-x64 /s /q
rmdir chia-blockchain-gui\release-builds /s /q
```

### 将 Editor.exe工具添加入路径

确认Visual Studio 2017或以上已经安装，本例中使用 Community 2017版本，根据版本的不同，请自行修改路径。

在Powershell中输入如下命令：

```powershell
$env:Path += ";C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\VC\Tools\MSVC\14.16.27023\bin\Hostx64\x64\" 
```

### 开始编译
在提示符下运行：

```powershell
build_scripts\build_windows.ps1
```


### 编译完成
编译完成完成后，可以在```chia-blockchain-gui\release-builds\windows-installer```目录中找到安装包。






