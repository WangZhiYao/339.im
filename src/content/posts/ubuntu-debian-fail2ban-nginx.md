---
title: "Ubuntu 或 Debian 在 Nginx 下使用 fail2ban 阻止恶意扫描"
pubDatetime: 2021-02-23T18:19:03+08:00
slug: ubuntu-debian-fail2ban-nginx
tags: ["Linux", "Ubuntu", "fail2ban", "Nginx"]
description: "在 /etc/fail2ban/filter.d 下新建 nginx-cc.conf"
---


在 `/etc/fail2ban/filter.d` 下新建 `nginx-cc.conf`

```bash
touch /etc/fail2ban/filter.d/nginx-cc.conf
```

输入：

```bash file="nginx-cc.conf"
[Definition]
failregex = ^<HOST> \- \S+ \[\] \".*\" (400|404|444) .+$
ignoreregex =.*(jpg|png)
```

然后在 `/etc/fail2ban/jail.d/` 的 `defaults-debian.conf` 中加入如下几行：

```bash file="defaults-debian.conf"
[nginx-botsearch]
enabled = true
[nginx-cc]
enabled = true
filter  = nginx-cc
logpath = %(nginx_access_log)s
port    = http,https
```

unban：

```bash
fail2ban-client set jailname unbanip ipaddress
```

规则校验：

```bash
fail2ban-regex /var/log/nginx/*.access.log /etc/fail2ban/filter.d/nginx-cc.conf
```
