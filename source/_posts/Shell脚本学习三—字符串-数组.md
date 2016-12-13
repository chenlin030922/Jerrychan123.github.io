---
title: 'Shell脚本学习三—字符串,数组'
date: 2016-12-13 14:59:13
tags: shell
categories: Shell学习
---
<h3>拼接字符串</h3>

```shell
your_name="aaa"
greeting="hello, "$your_name" !"
greeting_1="hello, ${your_name} !"
echo $greeting $greeting_1
```
<!--more-->

<h3>获取字符串长度</h3>

```shell
string="abcd"
echo ${#string} #输出 4
```

<h3>提取子字符串</h3>

```shell
string="alibaba is a great company"
echo ${string:1:4} #输出liba
```

<h3>查找子字符串</h3>

```shell
string="alibaba is a great company"
echo `expr index "$string" is`
```

------

<h2>数组</h2>

```shell
#数组定义
array_name=(value0 value1 value2 value3)
array_name[0]=value0
array_name[1]=value1
array_name[2]=value2

#读取数组：格式 ${array_name[index]}
valuen=${array_name[2]}

#使用@ 或 * 可以获取数组中的所有元素，例如：
${array_name[*]}
${array_name[@]}

##获取数组的长度
# 取得数组元素的个数
length=${#array_name[@]}
# 或者
length=${#array_name[*]}
# 取得数组单个元素的长度
lengthn=${#array_name[n]}

```


