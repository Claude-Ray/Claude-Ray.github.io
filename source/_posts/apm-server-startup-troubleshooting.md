---
title: 如何解决 Debian 系 Elastic apm-server 7.x 启动失败
date: 2019-09-13 11:55:57
tags: [APM,Elastic-APM,apm-server,Debian]
categories: Linux
---

本来是几个月前在 Ubuntu 部署 Elastic apm-server 遇到的问题，当时处理起来没遇到特别的卡点，就只是把解决过程丢到 Evernote 了。最近发现还有人在重复踩这个坑，因此我把笔记整理之后搬到这里作一个极简的分享。

<!--more-->

# apm-server 安装

实际步骤就不需要我复述了，官方提供现成的 deb 安装包。除了查看官方文档，更推荐使用 Kibana APM 看板自带的指南。

指南的 url 路径大概是 http://localhost:5601/app/kibana#/home/tutorial/apm?_g=()

不仅有安装引导，还提供按钮协助检查 apm-server 的服务状态。

![Kibana apm-server tutorial](/image/apm-server-startup-troubleshooting/kibana-apm-server-tutorial.png)

# 启动异常
在 debian 系发行版安装 apm-server 后，执行 `service apm-server start` 报告失败，且切换到 `systemctl` 也无效。

`service apm-server status`报错如下：

```sh
$ service apm-server status                                                                                          
● apm-server.service - Elastic APM Server                                                                               
   Loaded: loaded (/lib/systemd/system/apm-server.service; enabled; vendor preset: enabled)                             
   Active: failed (Result: exit-code) since Tue 2019-04-16 14:44:42 CST; 3s ago                                         
     Docs: https://www.elastic.co/solutions/apm                                                                         
  Process: 4783 ExecStart=/usr/share/apm-server/bin/apm-server $BEAT_LOG_OPTS $BEAT_CONFIG_OPTS $BEAT_PATH_OPTS (code=ex
 Main PID: 4783 (code=exited, status=1/FAILURE)                                                                         
                                                                                                                        
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Service hold-off time over, scheduling restart.                     
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Scheduled restart job, restart counter is at 5.                     
4 月 16 14:44:42 ray systemd[1]: Stopped Elastic APM Server.                                                             
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Start request repeated too quickly.                                 
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Failed with result 'exit-code'.                                     
4 月 16 14:44:42 ray systemd[1]: Failed to start Elastic APM Server. 
```

# 检查日志
首先使用 `journalctl` 查看 systemd 的日志，如下
```sh
$ journalctl -u apm-server.service
```

打印日志

```sh
-- Logs begin at Wed 2019-04-10 09:30:25 CST, end at Tue 2019-04-16 14:44:42 CST. --                                    
4 月 16 13:43:23 ray systemd[1]: Started Elastic APM Server.                                                             
4 月 16 13:43:23 ray apm-server[2487]: Exiting: error loading config file: config file ("/etc/apm-server/apm-server.yml")
4 月 16 13:43:23 ray systemd[1]: apm-server.service: Main process exited, code=exited, status=1/FAILURE                  
4 月 16 13:43:23 ray systemd[1]: apm-server.service: Failed with result 'exit-code'.                                     
4 月 16 13:43:23 ray systemd[1]: apm-server.service: Service hold-off time over, scheduling restart.                     
4 月 16 13:43:23 ray systemd[1]: apm-server.service: Scheduled restart job, restart counter is at 1.                     
4 月 16 13:43:23 ray systemd[1]: Stopped Elastic APM Server.                                                             
4 月 16 13:43:23 ray systemd[1]: Started Elastic APM Server.
# ... ，笔者注释，省略中间的多次重启信息
4 月 16 14:44:42 ray apm-server[4783]: Exiting: error loading config file: config file ("/etc/apm-server/apm-server.yml")
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Main process exited, code=exited, status=1/FAILURE
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Failed with result 'exit-code'.
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Service hold-off time over, scheduling restart.
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Scheduled restart job, restart counter is at 5.
4 月 16 14:44:42 ray systemd[1]: Stopped Elastic APM Server.
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Start request repeated too quickly.
4 月 16 14:44:42 ray systemd[1]: apm-server.service: Failed with result 'exit-code'.
4 月 16 14:44:42 ray systemd[1]: Failed to start Elastic APM Server.
```

这样找出真正的启动错误是 `Exiting: error loading config file: config file ("/etc/apm-server/apm-server.yml")`

# 解决方法

配置文件异常，采用 `apm-server export config` 进一步观察。提示如下：

```sh
error initializing beat: error loading config file: config file ("/etc/apm-server/apm-server.yml") must be owned by the beat user (uid=1000) or root
```

[github issues](https://github.com/elastic/apm-server/issues/2001) 上找到了类似的问题，但没有给出推荐的处理方案，所以决定自己动手解决。

`ls -l` 观察 `/etc/apm-server/` 的信息，发现除了 apm-server.yml 之外，owner 都是 root

```sh
$ ls -l /etc/apm-server
total 148K
drwxr-xr-x   2 root       root       4.0K 4 月  16 14:11 .
drwxr-xr-x 142 root       root        12K 4 月  16 14:11 ..
-rw-------   1 apm-server apm-server  33K 4 月   6 05:48 apm-server.yml
-rw-r--r--   1 root       root        94K 4 月   6 05:48 fields.yml
```

那么统一将权限变更到 root 吧！

```sh
$ sudo chown root:root /etc/apm-server/apm-server.yml
```
改之后测试

```sh
$ sudo apm-server test config
Config OK
```

再尝试启动则提示成功。

