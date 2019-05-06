---
title: Java-SVN分支切换和合并
date: 2018-05-15 11:02:14
categories: Java
---

# 创建分支

## 打开菜单

右键单击项目文件，选择如图所示的以下菜单选项：

![IMAGE](IDEA-SVN分支切换和合并/DF4101C753F3EC9F71337FE0BD4B5B63.jpg)

<!-- more -->

## 选择分支文件的初始代码和分支的位置

这里分成两个大的部分，Copy From 和 Copy To，Copy From 用于设置分支初始源码的 URL，其中 Working Copy 表示分支的初始源码来自本地，Repository Copy 表示分支源码来自一个 SVN 仓库，这里一般默认的是 SVN 上的 URL。Copy To 表示分支创建的 URL 路径，其中 Branch or Tag 分成 Base URL 和 Name (分支名称) 两个部分，合在一起就是分支的路径，Any Location 则是直接指定分支的 URL。

![IMAGE](IDEA-SVN分支切换和合并/85356AEA40D4657FF3097F96C4A65C06.jpg)

## 创建分支

在第二步中，分支初始源码选择 Repository Copy，使用当前项目在 SVN 中的源码作为分支初始源码，填写分支名字为 tdx-branch，点击确定后，刷新 Cornerstone (一款 Mac 下的 SVN 客户端)，就可以看见创建的分支文件了，如图所示：

![IMAGE](IDEA-SVN分支切换和合并/3CD47440F416BCDA7DBC5D3748909D66.jpg)

# 切换分支

**注意：切换分支前一定记得先提交**，在 VCS 菜单中，选择更新项目，如图所示：

![IMAGE](IDEA-SVN分支切换和合并/56239F22B457ED67CFF65B86F85A8D8F.jpg)

勾选 Update/Switch to specific url 选项，选择分支文件，注意 URL 不要弄错了，然后点击确认就可以切换分支了。

# 查看当前使用的分支

点击底部状态栏 Version Control -> Subversion Working Copies Information，如下图所示：

![IMAGE](IDEA-SVN分支切换和合并/CE2828E4DBF962156A1718464B9D094A.jpg)

# 合并分支

首先，菜单栏中依次选择 VCS -> Integtate Project，如下图所示：

<img src="IDEA-SVN分支切换和合并/651C374560639826AB53CE3B1D69689C.jpg" width="300" height="150">

然后，选择需要合并的主干项目 URL(对应 Sourc 1) 和分支项目 URL (对应 Source 2)，如下图所示：

![IMAGE](IDEA-SVN分支切换和合并/4B660AEC332D6AA82AB86384C69B2CE3.jpg)

分支合并到主干：切换到主干 -> 合并分支操作 -> Source 1 填主干项目 URL，Source 2 填分支项目 URL
主干同步到分支：切换到分支 -> 合并分支操作 -> Sorceu 1 填分支项目 URL，Source 2 填主干项目 URL