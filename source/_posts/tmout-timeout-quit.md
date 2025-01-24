---
title: "TMOUT - Shell 超时退出"
date: 2025-01-09 20:01:10 +0800
category: 杂项
tags:
	- UOS
  - TMOUT
---

最近，在 UOS 上通过 Tmux 跑一个小时的测试，当我去查看结果时发现其终端自动结束了，连带着 Tmux 会话也被终结了，导致无法查看到测试的结果，令人十分恼火。

<!-- 最近在使用 UOS 时，总是遇到终端自动退出了，即便是我使用 Tmux 在后台运行，也会被强制退出。
这对于跑测试的情况则-->

<!-- more -->

## TMOUT

原来是 UOS 为了增强系统的安全性，需要在用户输入空闲一段时间后自动断开，该功能通过 `TMOUT` 来实现。我们可以通过下面的命令来查看是否设置了 `TMOUT`：

``` shell
env | grep TMOUT
```
{% codeblock 输出 line_number:false %}
TMOUT=300
{% endcodeblock %}

其单位为秒，因此当我们在 5 分钟内没有响应时，UOS 将主动退出终端程序。

Tmux 会话同样继承了 `TMOUT` 变量，从而导致 Tmux 会话被终止。

## 解决方法

我们通过禁用 `TMOUT` 来解决上述问题，及使用 `export TMOUT=0` 重新导出环境变量。若您的配置采用了 `readonly TMOUT=900 ; export TMOUT`，那么您在当前的 shell 终端中是无法修改此环境变量的，您需要先将该配置中的 `readonly` 删除并重新登录。

## 参考

[1] https://blog.csdn.net/qq1186351245/article/details/124880409
[2] https://www.tenable.com/audits/items/CIS_Red_Hat_EL7_STIG_v2.0.0_STIG.audit:d6974c5d51be2a6f59794773830b5724
[3] https://stackoverflow.com/questions/62605288/tmux-windows-closing-due-to-auto-logout
