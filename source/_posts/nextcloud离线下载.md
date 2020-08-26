---
title: nextcloud离线下载
date: 2020-8-25 22:01:20
tags:
- nextcloud 
categories:
- linux
---
 
## install nextcloud

## install ocdownloader

这里设置正确的路径否则报无法找到视频地址错误
```
controller/ytdownloader.php
$this->YTDLBinary = '/usr/local/bin/youtube-dl'; // default path
```

## install aria2

参考https://gist.github.com/GAS85/79849bfd09613067a2ac0c1a711120a6

注意要想ocdownloader使用aria2，需要设置不加密方式，还是不太安全

## install youtube-dl
```
apt install python-pip
pip install youtube-dl
```

