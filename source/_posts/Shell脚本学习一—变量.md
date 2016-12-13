---
title: Shell脚本学习一—变量
date: 2016-12-13 14:10:01
tags: shell
categories: Shell学习
---

**注意，shell脚本学习的所有资料来源于http://c.biancheng.net/cpp/view，且仅供学习**
<h4>基本使用</h4>

```shell
chmod +x ./test.sh  #使脚本具有执行权限
./test.sh  #执行脚本
```

上述命令直接可以运行./test.sh来跑脚本，也可以sh test.sh来跑

读取：

```shell

#read命令从bash中读取输入参数传递给PERSON,如果要使用PERSON，需要在前面加一#个$表示引用
read PERSON
echo "Hello, $PERSON"
```

----
<!--more-->


<h4>变量</h4>

```shell
#定义变量
variableName="value"

#变量名外面的花括号是可选的，加不加都行，加花括号是为了帮助解释器识别变量的边界,比如echo "I am good at ${skill}Script"
${your_name}==$your_name

```

<h5>只读变量</h5>

```shell
#设置只读，使用 readonly 命令可以将变量定义为只读变量，只读变量的值不能被改变
myUrl="http://see.xidian.edu.cn/cpp/shell/"
readonly myUrl
```

<h5>删除变量<h5>

```shell
unset variable_name
```

<h5>变量类型</h5>

1. **局部变量**：局部变量在脚本或命令中定义，仅在当前shell实例中有效，其他shell启动的程序不能访问局部变量。
2. **环境变量**：所有的程序，包括shell启动的程序，都能访问环境变量，有些程序需要环境变量来保证其正常运行。必要的时候shell脚本也可以定义环境变量。
3. **shell变量**：shell变量是由shell程序设置的特殊变量。shell变量中有一部分是环境变量，有一部分是局部变量，这些变量保证了shell的正常运行

<h5>Shell特殊变量</h5>

| 变量   | 含义                                       |
| ---- | ---------------------------------------- |
| $0   | 当前脚本的文件名                                 |
| $n   | 传递给脚本或函数的参数。n 是一个数字，表示第几个参数。例如，第一个参数是$1，第二个参数是$2。 |
| $#   | 传递给脚本或函数的参数个数。                           |
| $*   | 传递给脚本或函数的所有参数。                           |
| $@   | 传递给脚本或函数的所有参数。被双引号(" ")包含时，与 $* 稍有不同，下面将会讲到。 |
| $?   | 上个命令的退出状态，或函数的返回值。                       |
| $$   | 当前Shell进程ID。对于 Shell 脚本，就是这些脚本所在的进程ID。   |

$n的意义在于调用shell时候，传递的参数：

```shell
#!/bin/bash
echo "File Name: $0"
echo "First Parameter : $1"
echo "First Parameter : $2"
echo "Quoted Values: $@"
echo "Quoted Values: $*"
echo "Total Number of Parameters : $#"

$./test.sh Zara Ali
File Name : ./test.sh
First Parameter : Zara
Second Parameter : Ali
Quoted Values: Zara Ali
Quoted Values: Zara Ali
Total Number of Parameters : 2
```

$*和￥@的区别：

$* 和 $@ 都表示传递给函数或脚本的所有参数，不被双引号(" ")包含时，都以"$1" "$2" … "$n" 的形式输出所有参数。

但是当它们被双引号(" ")包含时，"$*" 会将所有的参数作为一个整体，以"$1 $2 … $n"的形式输出所有参数；"$@" 会将各个参数分开，以"$1" "$2" … "$n" 的形式输出所有参数。

如下：

```shell
#!/bin/bash
echo "\$*=" $*
echo "\"\$*\"=" "$*"
echo "\$@=" $@
echo "\"\$@\"=" "$@"
echo "print each param from \$*"
for var in $*
do
    echo "$var"
done
echo "print each param from \$@"
for var in $@
do
    echo "$var"
done
echo "print each param from \"\$*\""
for var in "$*"
do
    echo "$var"
done
echo "print each param from \"\$@\""
for var in "$@"
do
    echo "$var"
done
执行 ./test.sh "a" "b" "c" "d"，看到下面的结果：
$*=  a b c d
"$*"= a b c d
$@=  a b c d
"$@"= a b c d
print each param from $*
a
b
c
d
print each param from $@
a
b
c
d
print each param from "$*"
a b c d
print each param from "$@"
a
b
c
d
```

退出变量：

$? 可以获取上一个命令的退出状态。所谓退出状态，就是上一个命令执行后的返回结果。

退出状态是一个数字，一般情况下，大部分命令执行成功会返回 0，失败返回 1。

不过，也有一些命令返回其他值，表示不同类型的错误。

$? 也可以表示函数的返回值

