layout: '[post]'
title: 深入了解ClassLoader
tags: Android插件
categories: Android插件学习
date: 2017-01-06 17:57:43
---


由于公司的项目是基于桌面进行开发的，其本身依附于桌面app，所以插件这一块就是我们的重中之重了，由于水平有限，所以只是在开发上层进行操作，并不涉及到插件的相关技术，最近项目不紧，所以为了提升自己的技术水平，索性就研究了一下公司的插件框架。在插件框架中，发现涉及到的东西还是很多的，包括如何使用插件中的ConText而不是宿主的Context，如何使用插件里的Res，如何构建Activity等等，所以需要一步一步来学习，这里就先对ClassLoader进行深入学习，为插件做个知识准备。

---

<h3>ClassLoader类基本介绍</h3>

`java.lang.ClassLoader`类的基本职责就是根据一个指定的类的名称，找到或者生成其对应的字节代码，然后从这些字节代码中定义出一个 Java 类，即 `java.lang.Class`类的一个实例。除此之外，`ClassLoader`还负责加载 Java 应用所需的资源，如图像文件和配置文件等，无论是 JVM 还是 Dalvik 都是通过 ClassLoader 去加载所需要的类。
<!--more-->

在ClassLoader类中提供了几个重要的方法：



| 方法                                       | 说明                                       |
| ---------------------------------------- | ---------------------------------------- |
| getParent()                              | 返回该类加载器的父类加载器。                           |
| loadClass(String name)                   | 加载名称为 `name`的类，返回的结果是 `java.lang.Class`类的实例。 |
| findClass(String name)                   | 查找名称为 `name`的类，返回的结果是 `java.lang.Class`类的实例。 |
| findLoadedClass(String name)             | 查找名称为 `name`的已经被加载过的类，返回的结果是 `java.lang.Class`类的实例。 |
| defineClass(String name, byte[] b, int off, int len) | 把字节数组 `b`中的内容转换成 Java 类，返回的结果是 `java.lang.Class`类的实例。这个方法被声明为 `final`的。 |
| resolveClass(Class c)                    | 链接指定的 Java 类。                            |

表示类名称的 `name`参数的值是类的二进制名称。需要注意的是内部类的表示，如 `com.example.Sample$1`和 `com.example.Sample$Inner`等表示方式。这些方法会在下面介绍类加载器的工作机制时，做进一步的说明。下面介绍类加载器的树状组织结构。

-----

<h3>类加载器的结构</h3>

在Java中的类加载器主要有两种类型的，一种是系统提供的，另外一种则是由Java应用开发人员编写的，系统主要提供的类加载器有三个：

- 引导类加载器（bootstrap class loader）：它用来加载 Java 的核心库，是用原生代码来实现的，并不继承自 `java.lang.ClassLoader`。
- 扩展类加载器（extensions class loader）：它用来加载 Java 的扩展库。Java 虚拟机的实现会提供一个扩展库目录。该类加载器在此目录里面查找并加载 Java 类。
- 系统类加载器（system class loader）：它根据 Java 应用的类路径（CLASSPATH）来加载 Java 类。**一般来说，Java 应用的类都是由它来完成加载的。可以通过 `ClassLoader.getSystemClassLoader()`来获取它**。

`bootstrap class loader`是所有的类加载器的父类，然后是`extensions class loader`，然后是`system class loader`,然后才是开发人员自主编写的ClassLoader，关系如下：

{% asset_img classloader.png classloader结构图 %}

每一个类都有自己的ClassLoader，我们可以通过如下代码来查看所有的类加载器的引用：

```java
public static void main(String[] args) { 
        ClassLoader loader = Test.class.getClassLoader(); 
        while (loader != null) { 
            System.out.println(loader.toString()); 
            loader = loader.getParent(); 
        } 
    } 
结果：
sun.misc.Launcher$AppClassLoader@4554617c
sun.misc.Launcher$ExtClassLoader@677327b6
```

在结果中可以看到两个Loader的存在，AppClassLoader即`system class loader`，如果不信，可以通过 `ClassLoader.getSystemClassLoader()`来进行测试获取是否为系统类加载器，如果`loader.getParent()`等于Null的话，其实就是`bootstrap class loader`，因为他没有parent。

----

<h3>类加载器的实现模式</h3>

ClassLoader实现的模式是双亲委托模式**，即类加载器在尝试自己去查找某个类的字节代码并定义它时，会先代理给其父类加载器，由父类加载器先去尝试加载这个类，依次类推。**

**注意：**首先需要说明一下 Java 虚拟机是如何判定两个 Java 类是相同的。Java 虚拟机不仅要看类的全名是否相同，还要看加载此类的类加载器是否一样。只有两者都相同的情况，才认为两个类是相同的。即认为是：ClassLoader id + PackageName + ClassName。如果出现相同两个类不能够强制转换或者判断相等，如：

```java
java.lang.ClassCastException: android.support.v4.view.ViewPager can not be cast to android.support.v4.view.ViewPager
```

那么需要去注意是否加载的ClassLoader是否相同。

源码:

```java
  protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
            // 确定是否已经加载过Class
            Class c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {
                      	//委托给parent进行加载
                        c = parent.loadClass(name, false);
                    } else {
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                    // ClassNotFoundException thrown if class not found
                    // from the non-null parent class loader
                }

                if (c == null) {
                    // If still not found, then invoke findClass in order
                    // to find the class.
                    long t1 = System.nanoTime();
                  	//自己加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                }
            }
            return c;
    }
```



那么ClassLoader之所以使用双亲委托模式目的则是：

双亲委托模式模式是为了保证 Java 核心库的类型安全。所有 Java 应用都至少需要引用 `java.lang.Object`类，也就是说在运行的时候，`java.lang.Object`这个类需要被加载到 Java 虚拟机中。**如果这个加载过程由 Java 应用自己的类加载器来完成的话，很可能就存在多个版本的 `java.lang.Object`类，而且这些类之间是不兼容的。**通过代理模式，对于 Java 核心库的类的加载工作由引导类加载器来统一完成，保证了 Java 应用所使用的都是同一个版本的 Java 核心库的类，是互相兼容的。

不同的类加载器为相同名称的类创建了额外的名称空间。**相同名称的类可以并存在 Java 虚拟机中，只需要用不同的类加载器来加载它们即可。**不同类加载器加载的类之间是不兼容的，这就相当于在 Java 虚拟机内部创建了一个个相互隔离的 Java 类空间。

-----

<h3>加载类的过程</h3>

类加载器会首先代理给其它类加载器来尝试加载某个类。**这就意味着真正完成类的加载工作的类加载器和启动这个加载过程的类加载器，有可能不是同一个。真正完成类的加载工作是通过调用 `defineClass`来实现的；而启动类的加载过程是通过调用 `loadClass`来实现的。**前者称为一个类的定义加载器（defining loader），后者称为初始加载器（initiating loader）。在 Java 虚拟机判断两个类是否相同的时候，使用的是类的定义加载器。也就是说，哪个类加载器启动类的加载过程并不重要，重要的是最终定义这个类的加载器。两种类加载器的关联之处在于：一个类的定义加载器是它引用的其它类的初始加载器。如类 `com.example.Outer`引用了类 `com.example.Inner`，则由类 `com.example.Outer`的定义加载器负责启动类 `com.example.Inner`的加载过程。

方法 `loadClass()`抛出的是 `java.lang.ClassNotFoundException`异常；方法 `defineClass()`抛出的是 `java.lang.NoClassDefFoundError`异常。

类加载器在成功加载某个类之后，会把得到的 `java.lang.Class`类的实例缓存起来。下次再请求加载该类的时候，类加载器会直接使用缓存的类的实例，而不会尝试再次加载。也就是说，对于一个类加载器实例来说，相同全名的类只加载一次，即 `loadClass`方法不会被重复调用。

----

<h3>Class.forName</h3>

`Class.forName`是一个静态方法，同样可以用来加载类。该方法有两种形式：`Class.forName(String name, boolean initialize, ClassLoader loader)`和 `Class.forName(String className)`。第一种形式的参数 `name`表示的是类的全名；`initialize`表示是否初始化类；`loader`表示加载时使用的类加载器。第二种形式则相当于设置了参数 `initialize`的值为 `true`，`loader`的值为当前类的类加载器。

----

<h3>Android中的ClassLoader：DexClassLoader</h3>

Android 也有自己的 ClassLoader，分为 dalvik.system.DexClassLoader 和 dalvik.system.PathClassLoader，区别在于 PathClassLoader 不能直接从 zip 包中得到 dex，因此只支持直接操作 dex 文件或者已经安装过的 apk（因为安装过的 apk 在 cache 中存在缓存的 dex 文件）。而 DexClassLoader 可以加载外部的 apk、jar 或 dex文件，并且会在指定的 outpath 路径存放其 dex 文件。  所以插件化的重点则在于dalvik.system.DexClassLoader的使用实现。

这里就介绍一下DexClassLoader，源码如下：

```java
/**
 * A class loader that loads classes from {@code .jar} and {@code .apk} files
 * containing a {@code classes.dex} entry. This can be used to execute code not
 * installed as part of an application.
 *
 * <p>This class loader requires an application-private, writable directory to
 * cache optimized classes. Use {@code Context.getDir(String, int)} to create
 * such a directory: <pre>   {@code
 *   File dexOutputDir = context.getDir("dex", 0);
 * }</pre>
 *
 * <p><strong>Do not cache optimized classes on external storage.</strong>
 * External storage does not provide access controls necessary to protect your
 * application from code injection attacks.
 */
public class DexClassLoader extends BaseDexClassLoader {
    /**
     * Creates a {@code DexClassLoader} that finds interpreted and native
     * code.  Interpreted classes are found in a set of DEX files contained
     * in Jar or APK files.
     *
     * <p>The path lists are separated using the character specified by the
     * {@code path.separator} system property, which defaults to {@code :}.
     *
     * @param dexPath the list of jar/apk files containing classes and
     *     resources, delimited by {@code File.pathSeparator}, which
     *     defaults to {@code ":"} on Android
     * @param optimizedDirectory directory where optimized dex files
     *     should be written; must not be {@code null}
     * @param libraryPath the list of directories containing native
     *     libraries, delimited by {@code File.pathSeparator}; may be
     *     {@code null}
     * @param parent the parent class loader
     */
    public DexClassLoader(String dexPath, String optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(dexPath, new File(optimizedDirectory), libraryPath, parent);
    }
```

方法解释中说了三点：

1. 这个类加载器加载的文件是.jar或者.apk文件，并且这个.jar或.apk中是包含classes.dex这个入口文件的，
   主要是用来执行那些没有被安装的一些可执行文件的
2. 这个类加载器需要一个属于应用的私有的，可执行写操作的目录作为它自己的缓存优化目录，**其实这个目录也就作为下面这个构造函数的第二个参数，至于怎么实现，注释中也已经给出了答案**
3. 不要把上面第二点中提到的这个缓存目录设为外部存储，因为外部存储容易收到代码注入的攻击

参数说明：

- dexPath：需要装载的APK或者Jar文件的路径。包含多个路径用`File.pathSeparator间隔开`,在Android上默认是 ":"
- optimizedDirectory：优化后的dex文件存放目录，不能为null
- libraryPath：目标类中使用的C/C++库的列表,每个目录用`File.pathSeparator间隔开`; 可以为 `null`
- parent：该类装载器的父装载器，一般用当前执行类的装载器

在DexClassLoader的初始化方法中完全只调用了父类BaseDexClassLoader的构造函数：

```java
public BaseDexClassLoader(String dexPath, File optimizedDirectory,
            String libraryPath, ClassLoader parent) {
        super(parent);
        this.pathList = new DexPathList(this, dexPath, libraryPath, optimizedDirectory);
    }
```

在BaseDexClassLoader中只是初始化了一个DexPathList，这里面存放了所有jar,dex,native lib的路径。当我们调用DexClassLoader的loadClass方法，最终会调用到BaseDexClassLoader的findClass()方法：

```java
 public Class findClass(String name, List<Throwable> suppressed) {
        for (Element element : dexElements) {
            DexFile dex = element.dexFile;
			//找到dex中包含的对应的className的class，并且返回
            if (dex != null) {
                Class clazz = dex.loadClassBinaryName(name, definingContext, suppressed);
                if (clazz != null) {
                    return clazz;
                }
            }
        }
        if (dexElementsSuppressedExceptions != null) {
            suppressed.addAll(Arrays.asList(dexElementsSuppressedExceptions));
        }
        return null;
    }
```

-----

总结：

通过DexClassLoader我们可以不安装apk而直接加载apk内的class文件，这也就为我们插件的实施提供了一个可行性的方案，我们可以去动态的加载类已达到我们修改apk的目的。









-----

参考资料：

- http://www.trinea.cn/android/android-plugin/
- http://www.iteye.com/topic/83978
- http://www.ibm.com/developerworks/cn/java/j-lo-classloader/
- http://bbs.pediy.com/showthread.php?t=199230