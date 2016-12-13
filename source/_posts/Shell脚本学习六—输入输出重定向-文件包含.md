---
title: Shell脚本学习六—输入输出重定向&文件包含
date: 2016-12-13 15:59:03
tags: shell
categories: Shell学习
---

<h2>输入输出重定向</h2>

<h3>输出重定向</h3>

命令输出重定向的语法为：

```shell
command > file

#ex
who > users
```
<!--more-->
输出重定向会覆盖文件内容，如果不想覆盖，使用 >>(相当于append)

<h3>输入重定向</h3>

```shell
command < file
```

这样，本来需要从键盘获取输入的命令会转移到文件读取内容。



-----

<h3>重定向深入讲解</h3>

一般情况下，每个 Unix/Linux 命令运行时都会打开三个文件：

- 标准输入文件(stdin)：stdin的文件描述符为0，Unix程序默认从stdin读取数据。
- 标准输出文件(stdout)：stdout 的文件描述符为1，Unix程序默认向stdout输出数据。
- 标准错误文件(stderr)：stderr的文件描述符为2，Unix程序会向stderr流中写入错误信息。

默认情况下，**command > file 将 stdout 重定向到 file，command < file 将stdin 重定向到 file。**

如果希望 stderr 重定向到 file，可以这样写：

```shell
$command 2 > file
```

如果希望 stderr 追加到 file 文件末尾，可以这样写：

```shell
$command 2 >> file
```

2 表示标准错误文件(stderr)。



如果希望将 stdout 和 stderr 合并后重定向到 file，可以这样写：

```shell
$command > file 2>&1
or
$command >> file 2>&1
```

如果希望对 stdin 和 stdout 都重定向，可以这样写：

```shell
$command < file1 >file2
```

command 命令将 stdin 重定向到 file1，将 stdout 重定向到 file2。 

<h3>全部可用的重定向命令列表</h3>

| 命令              | 说明                             |
| --------------- | ------------------------------ |
| command > file  | 将输出重定向到 file。                  |
| command < file  | 将输入重定向到 file。                  |
| command >> file | 将输出以追加的方式重定向到 file。            |
| n > file        | 将文件描述符为 n 的文件重定向到 file。        |
| n >> file       | 将文件描述符为 n 的文件以追加的方式重定向到 file。  |
| n >& m          | 将输出文件 m 和 n 合并。                |
| n <& m          | 将输入文件 m 和 n 合并。                |
| << tag          | 将开始标记 tag 和结束标记 tag 之间的内容作为输入。 |



<h3>/dev/null 文件</h3>

如果希望执行某个命令，但又不希望在屏幕上显示输出结果，那么可以将输出重定向到 /dev/null：

```shell
$ command > /dev/null
```

/dev/null 是一个特殊的文件，写入到它的内容都会被丢弃；如果尝试从该文件读取内容，那么什么也读不到。但是 /dev/null 文件非常有用，将命令的输出重定向到它，会起到”禁止输出“的效果。

如果希望屏蔽 stdout 和 stderr，可以这样写：

```shell
$ command > /dev/null 2>&1
```

------

<h2>文件包含</h2>

像其他语言一样，Shell 也可以包含外部脚本，将外部脚本的内容合并到当前脚本。

Shell 中包含脚本可以使用：

```shell
. filename
or
source filename
```

两种方式的效果相同，简单起见，一般使用点号(.)，**但是注意点号(.)和文件名中间有一空格。**

例如，创建两个脚本，一个是被调用脚本 subscript.sh，内容如下：

```shell
url="http://see.xidian.edu.cn/cpp/view/2738.html"
```

一个是主文件 main.sh，内容如下：

```shell
#!/bin/bash
. ./subscript.sh
echo $url
```


