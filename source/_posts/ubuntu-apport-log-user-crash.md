---
title: "Ubuntu 使用 Apport 记录用户程序崩溃信息"
date: 2024-06-08 22:29:59 +0800
category: 杂项
tags:
  - Ubuntu
---

在使用 Ubuntu 系统时，您可能会遇到应用程序崩溃或系统故障。为了帮助开发者快速发现和解决这些问题，Ubuntu 提供了一个名为 Apport 的错误报告工具。本文主要介绍一下如何使用 Apport 捕获用户程序的崩溃信息。

<!-- more -->

## 什么是 Apport

Apport 是一个用于捕捉应用程序崩溃信息并生成详细错误报告的工具。它不仅可以记录崩溃时的堆栈跟踪，还能收集与错误相关的系统和应用程序信息。这些报告可以自动发送到开发者团队，以便更快地修复问题。

## 如何禁用/启用 Apport

在 Ubuntu 12.04 LTS 及更高版本中，Apport 默认是启用的。如果您需要手动启用或禁用 Apport，可以通过修改 `/etc/default/apport` 文件来实现：该文件中的 `enable=1` 表示启用，`enable=0` 则表示禁用。修改之后我们需要通过 `sudo systemctl restart apport` 来重启 Apport 服务。

当某个应用程序崩溃后，Apport 会在 `/var/crash/` 目录下生成一个 `.crash` 文件。

## Apport 示例

这里我们通过源码编译安装 NeoVim 之后，使用 LazyVim 时，发现在执行 `LazyHealth` 的时候会出现崩溃的情况，但是由于 Apport 不会对非打包程序登记崩溃信息，因此我们采用默认配置无法查看程序崩溃时的堆栈信息。

这时我们需要对其进行配置告知 Apport 针对用户编译的程序同样登记崩溃信息，如下所示：

``` bash
$ mkdir ~/.config/apport
$ cat <<-EOF > ~/.config/apport/settings
[main]
unpackaged = true
EOF
```

当程序再次崩溃时，我们可以在 `/var/crash/` 目录下看到如下所示的崩溃日志记录：

``` bash
$ ls /var/crash/*
/var/crash/_usr_local_bin_nvim.1000.crash
```

随后，我们可以借助工具 `apport-unpack` 对其进行展开，并获取到程序运行时的 coredump 相关信息：

``` bash
$ apport-unpack /var/crash/_usr_local_bin_nvim.1000.crash nvim-crash
$ cd nvim-crash/
$ ls -l
total 35092
-rw-rw-r-- 1 japin japin        5 Jun  8 23:05 Architecture
-rw-rw-r-- 1 japin japin 35848192 Jun  8 23:05 CoreDump
-rw-rw-r-- 1 japin japin       24 Jun  8 23:05 Date
-rw-rw-r-- 1 japin japin       12 Jun  8 23:05 DistroRelease
-rw-rw-r-- 1 japin japin       19 Jun  8 23:05 ExecutablePath
-rw-rw-r-- 1 japin japin       10 Jun  8 23:05 ExecutableTimestamp
-rw-rw-r-- 1 japin japin        2 Jun  8 23:05 _HooksRun
-rw-rw-r-- 1 japin japin        5 Jun  8 23:05 ProblemType
-rw-rw-r-- 1 japin japin       12 Jun  8 23:05 ProcCmdline
-rw-rw-r-- 1 japin japin       26 Jun  8 23:05 ProcCwd
-rw-rw-r-- 1 japin japin      111 Jun  8 23:05 ProcEnviron
-rw-rw-r-- 1 japin japin    21277 Jun  8 23:05 ProcMaps
-rw-rw-r-- 1 japin japin     1533 Jun  8 23:05 ProcStatus
-rw-rw-r-- 1 japin japin        2 Jun  8 23:05 Signal
-rw-rw-r-- 1 japin japin        7 Jun  8 23:05 SignalName
-rw-rw-r-- 1 japin japin       29 Jun  8 23:05 Uname
-rw-rw-r-- 1 japin japin       40 Jun  8 23:05 UserGroups
```

最后，结合 GDB 我们就可以知道程序崩溃时的堆栈信息。

``` bash
$ gdb `cat ExecutablePath` CoreDump
#0  frame2win (frp=frp@entry=0x0) at /home/japin/Tools/neovim/src/nvim/window.c:3473
#1  0x00005c76f570f3e6 in winframe_find_altwin (win=win@entry=0x5c76f78184e0, dirp=dirp@entry=0x7ffcda0a259c, tp=tp@entry=0x5c76f779c930, altfr=altfr@entry=0x7ffcda0a2520)
    at /home/japin/Tools/neovim/src/nvim/window.c:3219
#2  0x00005c76f5711165 in winframe_remove (win=win@entry=0x5c76f78184e0, dirp=0x7ffcda0a259c, tp=tp@entry=0x5c76f779c930, unflat_altfr=unflat_altfr@entry=0x0)
    at /home/japin/Tools/neovim/src/nvim/window.c:3156
#3  0x00005c76f5713d2b in win_free_mem (win=win@entry=0x5c76f78184e0, dirp=dirp@entry=0x7ffcda0a259c, tp=tp@entry=0x5c76f779c930) at /home/japin/Tools/neovim/src/nvim/window.c:3080
#4  0x00005c76f5713f89 in win_close_othertab (win=0x5c76f78184e0, free_buf=free_buf@entry=0, tp=0x5c76f779c930) at /home/japin/Tools/neovim/src/nvim/window.c:3058
#5  0x00005c76f5717639 in close_windows (buf=0x5c76f7769a60, keep_curwin=false) at /home/japin/Tools/neovim/src/nvim/window.c:2515
#6  0x00005c76f54d6254 in do_buffer (action=action@entry=4, start=start@entry=1, dir=dir@entry=1, count=<optimized out>, count@entry=22, forceit=forceit@entry=0)
    at /home/japin/Tools/neovim/src/nvim/buffer.c:1401
#7  0x00005c76f54d6658 in do_bufdel (command=4, arg=<optimized out>, addr_count=1, start_bnr=<optimized out>, end_bnr=22, forceit=0)
    at /home/japin/Tools/neovim/src/nvim/buffer.c:1075
#8  0x00005c76f556bc2c in ex_bunload (eap=0x7ffcda0a2e40) at /home/japin/Tools/neovim/src/nvim/ex_docmd.c:4467
#9  0x00005c76f55630ad in execute_cmd0 (retv=retv@entry=0x7ffcda0a2764, eap=eap@entry=0x7ffcda0a2e40, errormsg=errormsg@entry=0x7ffcda0a2768, preview=preview@entry=false)
    at /home/japin/Tools/neovim/src/nvim/ex_docmd.c:1706
#10 0x00005c76f5563714 in execute_cmd (eap=0x7ffcda0a2e40, cmdinfo=<optimized out>, preview=<optimized out>) at /home/japin/Tools/neovim/src/nvim/ex_docmd.c:1791
#11 0x00005c76f54a1688 in nvim_cmd (channel_id=9223372036854775809, cmd=<optimized out>, opts=0x7ffcda0a30d7, arena=0x7ffcda0a30f0, err=0x7ffcda0a30e0)
    at /home/japin/Tools/neovim/src/nvim/api/command.c:646
[...]
```

## 参考

[1] https://wiki.ubuntu.com/Apport
[2] [How to change apport default behaviour for non-packaged application crashes?](https://stackoverflow.com/questions/14204961/)
