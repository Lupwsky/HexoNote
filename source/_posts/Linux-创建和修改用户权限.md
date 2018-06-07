---
title: Linux-创建和删修改用户权限
date: 2018-06-06 23:39:14
categories: Linux
---

以下操作的服务器是在阿里云服务器上有实际的操作成功。

# 创建用户

创建用户，并在 home 目录下创建 lupw 目录。

```text
useradd -d /home/lupw -m lupw
```

配置用户的登录密码，passwd [用户名]。

```text
passwd lupw
```

# 修改用户权限

默认新创建的用户是无法使用 sudo 权限的，需要进行配置，没有 sudo 权限时执行 sudo 指令时会提示：[用户名] is not in the sudoers file。

```text
[lupw@iZwz95w6z2icuvzwoxbna3Z ~]$ sudo mkdir jdk
[sudo] password for lupw:
lupw is not in the sudoers file.  This incident will be reported.
```

<!-- more -->

在 root 权限下更改 /etc/sudoers 文件，在 root ALL = (ALL) ALL 下面添加 [用户名] ALL = (ALL) ALL 即可，在修改之前需要修改文件的权限。

```text
chmod u+w /etc/sudoers
vim /etc/sudoers
```

添加指定用户的权限并保存文件。

```text
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
lupw    ALL=(ALL)       ALL
```

将文件的写权限改回。

```text
chmod u-w /etc/sudoers
```