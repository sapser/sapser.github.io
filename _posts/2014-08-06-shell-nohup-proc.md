---
layout: post
title: "关闭终端而不终止后端进程的几种方法总结"
date: 2014-08-06 10:02:00 +0800
categories: shell
---


经常在一个终端上将一些长时间运行的进程放在后端跑，但是当终端关闭后，这个后端进程也随之关闭了，这是咋回事呢。

简单来说就是一个ssh连接(开一个终端)会创建一个session，然后在终端上开启的进程都属于这个session，如果关闭终端(关闭ssh连接)，系统会向这个session发送`SIGHUP`信号，终止这个session下的所有进程。

那么解决这个问题的方法也就很明显了，要么是进程忽略`SIGHUP`信号，要么是将进程放到一个新的session中，下面给出目前了解到的几种解决方法：

1、使用`nohup`命令忽略`SIGHUP`信号

```bash
$ nohup cmd & 
```

<br />
2、使用`trap`命令忽略`SIGHUP`信号，通过`kill -l`可以看到`SIGHUP`信号对应数字`1`

```bash
$ trap '' 1 && cmd &
```

<br />
3、使用`setsid`命令让进程跑在新session中，进程自然不会接收到`SIGHUP`信号了

```bash
$ setsid cmd &
```

<br />
4、使用`screen`或`tmux`来让进程跑在新session中，这个就不详解了，具体google这两个工具的用法

