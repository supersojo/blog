---
title: linux下systemd service的编写
date: 2020-10-25 21:01:20
tags:
- systemd
categories:
- linux
---
 
## 编写service
```
[Unit]
After=network.service

[Service]
Type=forking
ExecStart=/usr/local/bin/mystart.sh

[Install]
WantedBy=default.target
```
设置service类型为forking。

## 编写脚本
```
#!/bin/bash
echo 'my start script start to run'


echo 'my start script run done'

exit 0
```
由于设置了service类型为forking，需要在我们的脚本的最后执行exit退出systemd
为我们fork的进程。

