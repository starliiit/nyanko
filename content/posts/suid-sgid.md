---
title: "SUID 与 SGID"
date: 2019-03-23T00:52:52+08:00
draft: false
tags: [linux]
description: ""
---

## 定义

一个可执行文件被作为一个进程执行时，该进程的 UID 和 GID 继承自它 fork 的那个进程。

也就是说，一般情况下，可执行文件运行时的 UID 和 GID 跟文件本身的所有者和所属组无关。

SUID 让可执行文件运行时，进程 UID 等于文件的所有者。
SGID 让可执行文件运行时，进程 GID 等于文件的所属组。
而与启动它的进程的 UID/GID 无关。

## 使用方式

```
sudo chmod u+s <file_name>
sudo chmod g+s <file_name>
```

## 表现形式

一个设置了 SGID 的可执行文件，原来 `x` 位置显示为 `s`。

```
-rwxr-sr-x 1 root alarm 8.2K Mar 23 00:33 print_uid
```

如果所有者/所属组本来就没有 `x` 权限，强行设置了 SUID/SGID 之后，会在 `x` 位置显示大写的 `S`。

```
# before `chmod u+s`
-rw-r--r-x 1 root alarm 8.2K Mar 23 00:33 print_uid

# after `chmod u+s`
-rwSr--r-x 1 root alarm 8.2K Mar 23 00:33 print_uid
```

注意上面的文件，所有者和所属组都无法执行，但是 others 可以执行。
others 执行结果中 SUID 不会生效，这是符合逻辑的。

这里的执行者是 others，如果执行者是所有者呢？

```
-rwSr--r-x 1 sl root 8.2K Mar 23 00:33 print_uid
```

当用户 sl 试图执行该文件时，显示 `permission denied`。因此，如果一个文件本来就不能被所有者/所属组执行，即使设置了 SUID/SGID，也是无法执行的。

## 清零

更改了所有者/所属组之后，SUID/SGID 会被清零。

## C API

- getuid
- geteuid
- getgid
- getegid

REF: https://www.linuxnix.com/suid-set-suid-linuxunix/