---
title: "FiberHome HG2543C1 桥接"
date: 2020-06-12T08:39:23+00:00
tags: [Networking]
---

内容**完全**来自[这篇 Blog](http://shigc.top/2019/10/17/article4/)，这里留给自己备份用。

# 启用 Telnet

访问如下 URL 开启 FiberHome HG2543C1 的 telnet

`http://$IP:8080/cgi-bin/telnetenable.cgi?telnetenable=1`

telnet 登录 FiberHome HG2543C1，用户 `root`，密码是默认 Wi-Fi 密码拼接默认 `useradmin` 密码，即 `$DEFAULT_WIFI_PASSWORD$DEFAULT_USERADMIN_PASSWORD`

# 获取 `telecomadmin` 密码

用户 `telecomadmin` 的密码位于 `/flash/cfg/agentconf/factory.conf` 的 `TelecomPasswd` 配置项

# 获取 PPPoE 帐号和密码

PPPoE 帐号密码位于 `/flash/cfg/app_conf/pppoe/chap-secrets`

# 通过 `telecomadmin` 帐号设置桥接

将连接 `2_INTERNET_B_VID_*` 的连接模式改为桥接即可
