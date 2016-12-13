---
title: 'Shell脚本学习二—替换,运算符'
date: 2016-12-13 14:41:17
tags: shell

categories: Shell学习
---

<h3>替换</h3>

<h4>转义符</h4>

| 转义字符 | 含义                 |
| ---- | ------------------ |
| \\   | 反斜杠                |
| \a   | 警报，响铃              |
| \b   | 退格（删除键）            |
| \f   | 换页(FF)，将当前位置移到下页开头 |
| \n   | 换行                 |
| \r   | 回车                 |
| \t   | 水平制表符（tab键）        |
| \v   | 垂直制表符              |

<!--more-->

<h4>命令替换</h4>

命令替换是指Shell可以先执行命令，将输出结果暂时保存，在适当的地方输出。

命令替换的语法：

```shell
#注意是反引号，不是单引号，这个键位于 Esc 键下方。
`command`
```

将命令执行结果保存在变量中。

```shell
#!/bin/bash
DATE=`date`
echo "Date is $DATE"
USERS=`who | wc -l`
echo "Logged in user are $USERS"
UP=`date ; uptime`
echo "Uptime is $UP"

Date is Thu Jul  2 03:59:57 MST 2009
Logged in user are 1
Uptime is Thu Jul  2 03:59:57 MST 2009
03:59:57 up 20 days, 14:03,  1 user,  load avg: 0.13, 0.07, 0.15
```

<h4>变量替换</h4>

变量替换可以根据变量的状态（是否为空、是否定义等）来改变它的值。

可以使用的变量替换形式：

| 形式              | 说明                                       |
| --------------- | ---------------------------------------- |
| ${var}          | 变量本来的值                                   |
| ${var:-word}    | 如果变量 var 为空或已被删除(unset)，那么返回 word，但不改变 var 的值。 |
| ${var:=word}    | 如果变量 var 为空或已被删除(unset)，那么返回 word，并将 var 的值设置为 word。 |
| ${var:?message} | 如果变量 var 为空或已被删除(unset)，那么将消息 message 送到标准错误输出，可以用来检测变量 var 是否可以被正常赋值。若此替换出现在Shell脚本中，那么脚本将停止运行。 |
| ${var:+word}    | 如果变量 var 被定义，那么返回 word，但不改变 var 的值。      |

```shell
#!/bin/bash

echo ${var:-"Variable is not set"}
echo "1 - Value of var is ${var}"

echo ${var:="Variable is not set"}
echo "2 - Value of var is ${var}"

unset var
echo ${var:+"This is default value"}
echo "3 - Value of var is $var"

var="Prefix"
echo ${var:+"This is default value"}
echo "4 - Value of var is $var"

echo ${var:?"Print this message"}
echo "5 - Value of var is ${var}"


Variable is not set
1 - Value of var is
Variable is not set
2 - Value of var is Variable is not set
3 - Value of var is
This is default value
4 - Value of var is Prefix
Prefix
5 - Value of var is Prefix
```

-----

<h3>Shell运算符</h3>

Bash 支持很多运算符，包括算数运算符、关系运算符、布尔运算符、字符串运算符和文件测试运算符。

原生bash不支持简单的数学运算，但是可以通过其他命令来实现，例如 awk 和 expr，expr 最常用。

expr 是一款表达式计算工具，使用它能完成表达式的求值操作。

```shell
#!/bin/bash
val=`expr 2 + 2`
echo "Total value : $val"
```



两点注意：

- 表达式和运算符之间要有空格，例如 2+2 是不对的，必须写成 2 + 2，这与我们熟悉的大多数编程语言不一样。
- 完整的表达式要被  包含，注意这个字符不是常用的单引号，在 Esc 键下边。

```shell
#!/bin/sh
a=10
b=20
val=`expr $a + $b`
echo "a + b : $val"
val=`expr $a - $b`
echo "a - b : $val"
val=`expr $a \* $b`
echo "a * b : $val"
val=`expr $b / $a`
echo "b / a : $val"
val=`expr $b % $a`
echo "b % a : $val"
if [ $a == $b ]
then
echo "a is equal to b"
fi
if [ $a != $b ]
then
echo "a is not equal to b"
fi
```

注意：

- 乘号(*)前边必须加反斜杠(\)才能实现乘法运算



<h4>算术运算符列表</h4>

| 运算符  | 说明                        | 举例                        |
| ---- | ------------------------- | ------------------------- |
| +    | 加法                        | `expr $a + $b` 结果为 30。    |
| -    | 减法                        | `expr $a - $b` 结果为 10。    |
| *    | 乘法                        | `expr $a \* $b` 结果为  200。 |
| /    | 除法                        | `expr $b / $a` 结果为 2。     |
| %    | 取余                        | `expr $b % $a` 结果为 0。     |
| =    | 赋值                        | a=$b 将把变量 b 的值赋给 a。       |
| ==   | 相等。用于比较两个数字，相同则返回 true。   | [ $a == $b ] 返回 false。    |
| !=   | 不相等。用于比较两个数字，不相同则返回 true。 | [ $a != $b ] 返回 true。     |

注意：

- 条件表达式要放在方括号之间，并且要有空格，例如 [$a==$b] 是错误的，必须写成 [ $a == $b ]



<h4>关系运算符</h4>

关系运算符只支持数字，不支持字符串，除非字符串的值是数字。

关系运算符列表

| 运算符  | 说明                            | 举例                      |
| ---- | ----------------------------- | ----------------------- |
| -eq  | 检测两个数是否相等，相等返回 true。          | [ $a -eq $b ] 返回 true。  |
| -ne  | 检测两个数是否相等，不相等返回 true。         | [ $a -ne $b ] 返回 true。  |
| -gt  | 检测左边的数是否大于右边的，如果是，则返回 true。   | [ $a -gt $b ] 返回 false。 |
| -lt  | 检测左边的数是否小于右边的，如果是，则返回 true。   | [ $a -lt $b ] 返回 true。  |
| -ge  | 检测左边的数是否大等于右边的，如果是，则返回 true。  | [ $a -ge $b ] 返回 false。 |
| -le  | 检测左边的数是否小于等于右边的，如果是，则返回 true。 | [ $a -le $b ] 返回 true。  |



布尔运算符列表

| 运算符  | 说明                                 | 举例                                    |
| ---- | ---------------------------------- | ------------------------------------- |
| !    | 非运算，表达式为 true 则返回 false，否则返回 true。 | [ ! false ] 返回 true。                  |
| -o   | 或运算，有一个表达式为 true 则返回 true。         | [ $a -lt 20 -o $b -gt 100 ] 返回 true。  |
| -a   | 与运算，两个表达式都为 true 才返回 true。         | [ $a -lt 20 -a $b -gt 100 ] 返回 false。 |



字符串运算符列表

| 运算符  | 说明                      | 举例                    |
| ---- | ----------------------- | --------------------- |
| =    | 检测两个字符串是否相等，相等返回 true。  | [ $a = $b ] 返回 false。 |
| !=   | 检测两个字符串是否相等，不相等返回 true。 | [ $a != $b ] 返回 true。 |
| -z   | 检测字符串长度是否为0，为0返回 true。  | [ -z $a ] 返回 false。   |
| -n   | 检测字符串长度是否为0，不为0返回 true。 | [ -z $a ] 返回 true。    |
| str  | 检测字符串是否为空，不为空返回 true。   | [ $a ] 返回 true。       |

文件测试运算符

文件测试运算符用于检测 Unix 文件的各种属性。

例如，**变量 file 表示文件“/var/www/tutorialspoint/unix/test.sh”**，它的大小为100字节，具有 rwx 权限。下面的代码，将检测该文件的各种属性：

```shell
#!/bin/sh
file="/var/www/tutorialspoint/unix/test.sh"
if [ -r $file ]
then
echo "File has read access"
else
echo "File does not have read access"
fi
if [ -w $file ]
then
echo "File has write permission"
else
echo "File does not have write permission"
fi
if [ -x $file ]
then
echo "File has execute permission"
else
echo "File does not have execute permission"
fi
if [ -f $file ]
then
echo "File is an ordinary file"
else
echo "This is sepcial file"
fi
if [ -d $file ]
then
echo "File is a directory"
else
echo "This is not a directory"
fi
if [ -s $file ]
then
echo "File size is zero"
else
echo "File size is not zero"
fi
if [ -e $file ]
then
echo "File exists"
else
echo "File does not exist"
fi



File has read access
File has write permission
File has execute permission
File is an ordinary file
This is not a directory
File size is zero
File exists
```

文件测试运算符列表

| 操作符     | 说明                                       | 举例                     |
| ------- | ---------------------------------------- | ---------------------- |
| -b file | 检测文件是否是块设备文件，如果是，则返回 true。               | [ -b $file ] 返回 false。 |
| -c file | 检测文件是否是字符设备文件，如果是，则返回 true。              | [ -b $file ] 返回 false。 |
| -d file | 检测文件是否是目录，如果是，则返回 true。                  | [ -d $file ] 返回 false。 |
| -f file | 检测文件是否是普通文件（既不是目录，也不是设备文件），如果是，则返回 true。 | [ -f $file ] 返回 true。  |
| -g file | 检测文件是否设置了 SGID 位，如果是，则返回 true。           | [ -g $file ] 返回 false。 |
| -k file | 检测文件是否设置了粘着位(Sticky Bit)，如果是，则返回 true。   | [ -k $file ] 返回 false。 |
| -p file | 检测文件是否是具名管道，如果是，则返回 true。                | [ -p $file ] 返回 false。 |
| -u file | 检测文件是否设置了 SUID 位，如果是，则返回 true。           | [ -u $file ] 返回 false。 |
| -r file | 检测文件是否可读，如果是，则返回 true。                   | [ -r $file ] 返回 true。  |
| -w file | 检测文件是否可写，如果是，则返回 true。                   | [ -w $file ] 返回 true。  |
| -x file | 检测文件是否可执行，如果是，则返回 true。                  | [ -x $file ] 返回 true。  |
| -s file | 检测文件是否为空（文件大小是否大于0），不为空返回 true。          | [ -s $file ] 返回 true。  |
| -e file | 检测文件（包括目录）是否存在，如果是，则返回 true。             | [ -e $file ] 返回 true。  |
