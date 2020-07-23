---
title: 关于kodi声音变小的问题
date: 2020-07-22 21:18:10
tags:
- kodi

categories:
- pc
---

由于debian下使用kodi不是官方推荐的，所以更新发行版到ubuntu 20.04 lts版。
但是kodi经过HDMI连接电视后声音非常小，把电视上音量调到最大，勉强可以听清。

## 1、排除电视自身的问题
使用电视自带的系统播放视频，声音正常，可以排除电视的问题。

## 2、kodi设置的问题
google搜索相关的问题得到以下信息。

```
https://discourse.coreelec.org/t/solved-sound-volume-via-hdmi-suddenly-very-low/6045/4

A couple of things that might help.

One, plug a keyboard in and press the + key a few times. kodi’s volume indicator should popup to centre of the screen. It’s possible that kodi’s internal sound level ended up turned down some how. Use + and - keys to adjust volume level if this is the case.

Two, check the audio device that you are using, Settings/System/Audio. Try changing it to one that mentions hdmi at the end of it. If it is not already on that one.

The +/- did the trick. Tbh, I didn’t even know Kodi had that key shortcuts.
In all the years I’ve been using Kodi on several platforms, I’ve always used CEC.

For some reason the volume was at one third. Putting it back to 100% and volume was back to normal (for me).

Thanks for the replies!
```

解决方法是连接键盘，按+/-键，出现kodi音量调节项，设置到最大音量恢复正常。
但是还有点问题，每次电视关掉再打开声音又变小，必须重启kodi才恢复正常。


