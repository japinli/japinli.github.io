---
title: "Ubuntu 笔记本盖住不挂机"
date: 2020-03-08 12:20:23 +0800
category: 杂项
tags:
  - Ubuntu
---

我有一台旧笔记本装有 Ubuntu 18.04 系统，希望把它当作一个服务器，这时我就需要在盖住笔记本时系统不休眠。

Ubuntu 系统中提供了 Login Manager 来管理这些行为，它的配置文件为 /etc/systemd/logind.conf。通过
`man logind.conf` 我们可以看到其详细说明。

其中有一项 `HandleLidSwitch`，它控制了笔记本盖住的行为，默认情况下是 `suspend`，我们可以将其修改为 `ignore` 即可以实现我们的目的。例如，我们修改 /etc/system/logind.conf 文件中的

```
#HandleLidSwitch=suspend
```

行修改为

```
HandleLidSwitch=ignore
```

然后重启 systemd-logind 服务即可。

``` shell
$ sudo service systemd-logind restart
```

## WIFI 配置

当我们使用这种方式合住机盖时，WIFI 连接有时可能不是很稳定，这是因为 WIFI 默认启用了省点模式，可以通过下面的方式关闭 WIFI 的省点模式：

``` shell
$ sed -i 's/wifi.powersave = 3/wifi.powersave = 2/' /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
$ sudo systemctl restart network-manager.service
```

本质上就是将 `/etc/NetworkManager/conf.d/default-wifi-powersave-on.conf` 中的 `wifi.powersave` 由 `3` 修改为 `2`。

* `NM_SETTING_WIRELESS_POWERSAVE_DEFAULT (0)`: use the default value
* `NM_SETTING_WIRELESS_POWERSAVE_IGNORE (1)`: don't touch existing setting
* `NM_SETTING_WIRELESS_POWERSAVE_DISABLE (2)`: disable powersave
* `NM_SETTING_WIRELESS_POWERSAVE_ENABLE (3)`: enable powersave

当 SSH 连接到主机时，如果出现 WIFI 连接无法外网时，可以使用 `nmcli` 命令来连接网络。首先，使用下面的命令查看 WIFI 信息：

``` shell
$ nmcli d wifi
IN-USE  BSSID              SSID           MODE   CHAN  RATE        SIGNAL  BARS  SECURITY
*       D4:4D:9F:FD:44:86  ChinaNet-abPe  Infra  11    130 Mbit/s  89      ▂▄▆█  WPA2 WPA3
```

接着使用 `nmcli d wifi connect ChinaNet-abPe password your-password` 来连接 WIFI 网络。

## 参考

[1] https://gist.github.com/jcberthon/ea8cfe278998968ba7c5a95344bc8b55
