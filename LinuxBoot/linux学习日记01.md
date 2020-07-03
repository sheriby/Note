---
title: linux学习日记01
author: Hony Sher
avatar: 'https://github.com/sheriby/cdn/blob/master/img/custom/head.jpg?raw=true'
authorLink: sheriby.github.io
authorAbout: 一枚小菜鸡
authorDesc: 一枚小菜鸡
categories: 技术
comments: true
tags:
  - 悦读
  - linux
photos: 'https://cdn.jsdelivr.net/gh/sheriby/cdn@1.12/img/cover/15.jpg'
date: 2020-06-28 19:43:21
keywords: linux
description: 第三章之linux中的两种链接文件，硬链接和软链接
---

# 链接文件

在linux中有两种链接文件，一种是硬链接，一种是软链接（又被称为符号链接）。

## 硬链接

我们可以使用`ln`命令来制作硬链接。如：

```bash
touch file
ln file linkfile
```

此时`linkfile`就是`file`的硬链接。

硬链接更像是给文件起一个别名。在linux文件系统中，每一个文件都被单独分配了一个`inode`，文件系统通过这个`inode`来寻找文件。我们可以通过`ls -i`命令来查看文件的的`inode`。

```bash
sher: ls -li
total 8
656782 -rw-r--r-- 2 sher sher 30 Jun 28 18:42 file
656782 -rw-r--r-- 2 sher sher 30 Jun 28 18:42 linkfile
```

两个文件的`icode`是相同的，大小等各种信息都是相同的。当我们修改`file`文件中的内容的时候，`linkfile`中的内容也会被修改。

但是自始至终都只有一份文件存在，`linkfile`只是`file`的别名，就像是`c++`中的引用，`c`语言中的指针。（`linux`内核是使用c语言写的，所以肯定是指针来实现的，不过c++的引用也只是对指针的一个封装罢了）

硬链接通过`inode`进行链接。

## 软链接

我们可以使用`ln -s`命令来制作软链接，其中的参数`s表示的是soft或者说是symbolic`，软链接就像是我们在`windows`中经常使用的快捷方式。在`windows`中可能会遇到这种情况，该快捷方式不可用，这就是原来的文件不在原有的位置导致的。

我们使用`ln -s file linkfile2`创建`file`的一个软链接。

```bash
sher: ls -li                                                   
total 8
656782 -rw-r--r-- 2 sher sher 30 Jun 28 18:42 file
656782 -rw-r--r-- 2 sher sher 30 Jun 28 18:42 linkfile
657082 lrwxrwxrwx 1 sher sher  4 Jun 28 18:49 linkfile2 -> file
```

我们发现软链接的`inode`改变了，大小也改变了，后面还跟上了一个箭头，表示这个软链接指向本文件夹中的`file`文件。

软链接通过`position`进行链接。

## 两种链接的简单比较

### 使用mv指令

`mv`指令是不会改变一个文件的`inode`的，只会改变文件所在的位置。

当我们执行`mv file file2`指令之后。

```bash
sher: ls -li
total 8
656782 -rw-r--r-- 2 sher sher 30 Jun 28 18:42 file2
656782 -rw-r--r-- 2 sher sher 30 Jun 28 18:42 linkfile
657082 lrwxrwxrwx 1 sher sher  4 Jun 28 18:49 linkfile2 -> file
```

上面的文本框没有颜色，在`linkfile2 -> file`是有红色的背景的，此时这个软链接不可用，因为本文件夹内并没有一个文件叫做`file`。但是硬链接是通过`inode`来链接的，`mv`指令并没有改变`inode`，因而使用硬链接还可以访问链接之前的文件。

也就是说，硬链接通过`inode`链接，其实际上没有大小。

软链接通过`position`链接，其有大小，存放的就是链接的位置。

### 使用rm指令

这里说的rm就是`/usr/bin/rm`这个文件，我呢实际上为了防止误删除将我们`rm`指令改成了`mv`指令。我在我的`.zshrc`中添加了如下的代码。

```bash
alias rm = trash

trash() { # 文件放入回收站
	mv $# ~/.trash
}

ctrash() { # 清空回收站
	echo -n "clear sure? [y/n] "
	read confirm
	[ $confirm = 'y' ] || [ $confirm = 'Y' ] && rm -rf ~/.trash/* && echo "clear done!"
}
```

当然上面并不是这里要说的重点。

上面说硬链接和指针相似，但是和指针还是有一点区别的，`linux`系统中会对`inode`进行计数，当我们删除一个文件时候，`inode`数量为1的时候才会真正删除这个文件。也就是说当我们执行`rm file`的时候，不会真正删除这个文件，此时`linkfile`还是可以继续使用，也就说使用硬链接之后，我们是分不清哪一个文件是链接文件的。

同理，删除了`file`之后，软链接还是无法访问的。

### 源文件不存在时

硬链接需要`inode`的存在，源文件不存在也就没有链接的目标。所以此时**无法创建硬链接**。但是软链接只需要位置，即使那个位置没有文件，我们还是可以表示出那个位置，所以**可以创建软链接。**

### 不同的存储媒体

当我们需要制作不同的存储媒体之间的链接的时候，我们只能选择**软链接**，因为不同的媒体之间无法通过`inode`进行链接。因而软链接使用的范围更大一点，在系统中软链接随处可见。比如`linux`的根目录中就有四个目录软链接。

```bash
total 52K
lrwxrwxrwx   1 root root    7 May 20 06:42 bin -> usr/bin
drwxr-xr-x   4 root root 4.0K Jun 21 08:52 boot
drwxr-xr-x  20 root root 3.6K Jun 28 18:35 dev
drwxr-xr-x  88 root root 4.0K Jun 28 18:30 etc
drwxr-xr-x   3 root root 4.0K Jun 17 07:57 home
lrwxrwxrwx   1 root root    7 May 20 06:42 lib -> usr/lib
lrwxrwxrwx   1 root root    7 May 20 06:42 lib64 -> usr/lib
drwx------   2 root root  16K Mar  6 01:33 lost+found
drwxr-xr-x   4 root root 4.0K Mar 13 22:24 mnt
drwxr-xr-x   6 root root 4.0K May 27 11:48 opt
dr-xr-xr-x 245 root root    0 Jun 29  2020 proc
drwxr-x---  20 root root 4.0K Jun 27 21:26 root
drwxr-xr-x  21 root root  560 Jun 28 18:30 run
lrwxrwxrwx   1 root root    7 May 20 06:42 sbin -> usr/bin
drwxr-xr-x   4 root root 4.0K Mar  6 01:54 srv
dr-xr-xr-x  13 root root    0 Jun 29  2020 sys
drwxrwxrwt  25 root root  680 Jun 28 19:54 tmp
drwxr-xr-x   9 root root 4.0K Jun 27 21:28 usr
drwxr-xr-x  12 root root 4.0K Jun 28 18:14 var
```

## 软链接的复制

### 保留软链接

硬链接的复制自然不必多说很容易理解，但是软链接就不是那么简单的了。

现在有文件夹`dict`，里面内容如下：

```bash
total 0
-rw-r--r-- 1 sher sher 0 Jun 28 20:08 a.txt
lrwxrwxrwx 1 sher sher 5 Jun 28 20:08 b.txt -> a.txt
```

通过`cp`指令可以复制这个文件夹。

```bash
cp -r dict linkdict
```

此时查看`linkdict`文件夹中的内容。

```bash
total 0
-rw-r--r-- 1 sher sher 0 Jun 28 20:10 a.txt
lrwxrwxrwx 1 sher sher 5 Jun 28 20:10 b.txt -> a.txt
```

我们将软链接成功的复制了过来。

上面我们是通过`cp`指令复制整个文件夹，但是如果我们单纯的复制文件的时候。

```bash
mkdir linkdict
cp dict/* linkdict
```

此时`linkdict`文件夹中内容如下，

```bash
total 0
-rw-r--r-- 1 sher sher 0 Jun 28 20:15 a.txt
-rw-r--r-- 1 sher sher 0 Jun 28 20:15 b.txt
```

软链接丢失了，这很不妙。

单独复制软链接文件时会导致软链接消失，需要保持软链接的话需要使用`cp -d`命令。

### 替换软链接

当然还有一种情况，有时候我们复制的时候不想要保留软链接，想要使用源文件替换掉软链接文件。比如说，我们要像云端上面文件夹`a`，但是`a`中有文件软链接到我们本地的`b`文件夹，此时当然不能上传软链接。

此时我们可以使用`cp -L`命令。如：

```bash
cp -Lr dict linkdict
```

此时`linkdict`文件夹中内容如下：

```bash
total 0
-rw-r--r-- 1 sher sher 0 Jun 28 20:25 a.txt
-rw-r--r-- 1 sher sher 0 Jun 28 20:25 b.txt
```

此时b虽然并不是链接空了的文件，其和原来的a文件是一样的。

无论如何，我们是绝对不可以直接复制一个软链接文件的。

## 我使用的软链接

我电脑里面也使用了不少软链接，最是我印象深刻的是`vim`。

一开始我使用的是`vim`，也配置了自己的`.vimrc`，但是后来我开始转用`neovim`。`neovim`的配置文件是可以直接使用`vim`的配置文件的，所以当时我使用了。

```bash
ln -s .vimrc .config/nvim/init.vim
```

做一个软链接。

但是后来我发现`neovim`非常好用便卸载了`vim`，但是很多时候我们都使用`vim`来打开文件而不是`nvim`。此时使用

```bash
ln -s /usr/bin/nvim /usr/bin/vim
```

此时我们使用`vim`也可以打开`nvim`了。

（当我没有卸载`vim`的时候，使用的是`alias vim = nvim`）



