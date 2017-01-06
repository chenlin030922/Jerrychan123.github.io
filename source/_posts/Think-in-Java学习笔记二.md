---
title: Think-in-Java学习笔记二:操作符&初始化与清理
date: 2016-12-31 20:43:26
tags: notes
categories: Think in Java
---

```java
a=4;
```

对于基本数据类型的赋值很简单，**基本类型存储了实际的数值，而并非指向一个对象的引用**，所以赋值的时候，是直接讲一个地方的内容复制到了另一个地方。而对于对象而言，将一个对象赋值给另外一个对象，**实际是将*“引用”*从一个人地方复制到另外一个地方**，也就是说若c=d，那么c和d都会指向原本d指向的那个对象，改变c或者d的值都会对对方产生影响。

对于前缀运算( -- a)和后缀运算(a--)而言前缀运算先生成值再执行运算，而后缀运算则是先执行运算再生成值，如：

```java
a=1;
2+(--a);//a先会减一再加二
(2+a++);//a先加二在加一
```

<!--more-->

==与equals()

==符号比较的是对象之间的引用，如果想要去比较对象的实际内容是否相同，我们需要重写equals()方法，如：

```java
Integer a=new Integer(47);
Integer b=new Integer(47);
a==b; //返回false因为比较的是引用
a.equals(b); //返回true，比较的是内容
```

如果我们创建了自己的类，没有重写equals()方法，那么对象会默认去比较引用，因为**equals()方法默认行为是比较引用**。

-----

<h3>多态中的缺陷：域和静态方法</h3>

在多态机制当中，只有方法的调用是可以多态的，如果直接访问某个域，那么这个访问在编译时期进行解析从而确定，多态则是通过后期绑定即运行时绑定方式来确定的：

```java
public class Test {
    public int a=0;
    public int getA(){
        return a;
    }
}
public class SubTest extends Test {
    public int a=1;
    public int getA(){
        return a;
    }
}
public class Main {
    public static void main(String[] args) {
        Test test=new SubTest();
        System.out.println(test.a+"-----"+test.getA());
    }
}
输出结果：
  0-----1

```

从上面结果我们结果我们就可以知道由于调用test.a是在域中的，因而在编译时期就确定了结果即调用Test中的a的值0，而方法getA()是通过后期绑定的，因此获取到的结果是1。

复杂对象调用构造器遵从的顺序：

1. 调用基类的构造器，这个步骤会不断地递归下去，首先是构造这种层结构的根，然后是下一层导出类直到最底层的到导出类
2. 按照声明顺序调用成员的初始化方法
3. 调用导出类的构造器主体

```java

public class Main {
    public static void main(String[] args) {
        Main main=new Main();
        Sandwich sandwich=main.new Sandwich();
    }
    public class Meal{
        public Meal() {
            // TODO Auto-generated constructor stub
            System.out.println("Meal()");
        }
    }
    public class Cheese{
        public Cheese() {
            // TODO Auto-generated constructor stub
            System.out.println("Cheese()");
        }
    }
    public class Lunch extends Meal{
        public Lunch() {
            System.out.println("Lunch()");
        }
    }
    public class PortableLunch extends Lunch{
        public PortableLunch() {
            System.out.println("PortableLunch()");
        }
    }
    public class Sandwich extends PortableLunch{
        private Cheese cheese=new Cheese();
        public Sandwich() {
            System.out.println("Sandwich()");
        }
        
    }
    
}
result:

Meal()
Lunch()
PortableLunch()
Cheese()
Sandwich()

```







