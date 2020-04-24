# JVM的整体结构

都是针对Hotspot虚拟机

![架构.png](https://i.loli.net/2020/03/16/HSkVZNCvAMr8O5Q.png)

其中方法区和堆是所有线程共享的，而其他几个部分是线程独占的。



# JVM的设计架构模型

Java的编译器的输入指令流是基于**栈**的指令架构，还有一种架构是基于**寄存器的**指令架构

| 基于栈的指令架构                            | 基于寄存器的指令架构                          |
| ------------------------------------------- | --------------------------------------------- |
| 设计和实现更简单                            |                                               |
| 不受硬件的约束，跨平台性更好                | 完全依赖硬件的，可移植性较差                  |
| 指令集使用0地址指令                         | 指令集使用1地址指令、2地址指令、3地址指令为主 |
| 由于是0地址指令所以更依赖栈，编译器容易实现 | 花费更少的指令完成一项操作                    |

## 代码测试

```java
public class JvmStruct {
    public static void main(String[] args) {
        int i=0;
        i=1+2;
    }
}
```

然后在控制台进入对应的.class文件夹中，反编译.class文件，指令是：

```
javap -v jvmstruct.class
```

然后查看结果：

```console
public class com.kyriexu.JvmStruct
  minor version: 0
  major version: 55
  flags: (0x0021) ACC_PUBLIC, ACC_SUPER
  this_class: #2                          // com/kyriexu/JvmStruct
  super_class: #3                         // java/lang/Object
  interfaces: 0, fields: 0, methods: 2, attributes: 1
Constant pool:
   #1 = Methodref          #3.#19         // java/lang/Object."<init>":()V
   #2 = Class              #20            // com/kyriexu/JvmStruct
   #3 = Class              #21            // java/lang/Object
   #4 = Utf8               <init>
   #5 = Utf8               ()V
   #6 = Utf8               Code
   #7 = Utf8               LineNumberTable
   #8 = Utf8               LocalVariableTable
   #9 = Utf8               this
  #10 = Utf8               Lcom/kyriexu/JvmStruct;
  #11 = Utf8               main
  #12 = Utf8               ([Ljava/lang/String;)V
  #13 = Utf8               args
  #14 = Utf8               [Ljava/lang/String;
  #15 = Utf8               i
  #16 = Utf8               I
  #17 = Utf8               SourceFile
  #18 = Utf8               JvmStruct.java
  #19 = NameAndType        #4:#5          // "<init>":()V
  #20 = Utf8               com/kyriexu/JvmStruct
  #21 = Utf8               java/lang/Object
{
  public com.kyriexu.JvmStruct();
    descriptor: ()V
    flags: (0x0001) ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 7: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Lcom/kyriexu/JvmStruct;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: (0x0009) ACC_PUBLIC, ACC_STATIC
    Code:
      stack=1, locals=2, args_size=1
         0: iconst_0
         1: istore_1
         // 在编译的时候已经识别出了i的值是3了
         2: iconst_3
         3: istore_1
         4: return
      LineNumberTable:
        line 9: 0
        line 10: 2
        line 11: 4
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  args   [Ljava/lang/String;
            2       3     1     i   I
}
SourceFile: "JvmStruct.java"
```

由此可以看见指令集比较多，但是基于寄存器的指令集就很少，没有这么多的指令集。

## 小总结

由于java跨平台的设计，所以决定java的指令是根据栈来设计的。然后特点有：指令多、指令集小，执行的性能比基于寄存器实现的差。

# JVM的生命周期

## 虚拟机的启动

java虚拟机启动是通过引导类加载器创建一个初始类来完成的，这个类是由虚拟机的具体实现决定的。

## 虚拟机的执行

Java虚拟机运行的时候就只有一个目的——执行Java程序，程序开始的时候他就开始执行，程序结束的时候它就结束，所以执行一个java程序就是执行一个虚拟机进程

# 类加载器

![classloader.png](https://i.loli.net/2020/03/16/LSiyGOxT3q8vUJH.png)

类加载器有下面的几个：

1. 虚拟机自带的加载器
2. 启动类（根）(bootstrap)加载器 BootstrapClassLoader
3. 扩展类加载器 ExtClassLoader 
4. 系统加载器加载器 AppClassLoader

通过一个小例子来看看几个加载器：

```java
public class Demo01 {
    public static void main(String[] args) {
        Demo01 demo01 = new Demo01();
        ClassLoader dClassLoader = demo01.getClass().getClassLoader();
        System.out.println(dClassLoader);
        System.out.println(dClassLoader.getParent());
        System.out.println(dClassLoader.getParent().getParent());
    }
}
```

![classloader_code.png](https://i.loli.net/2020/03/16/EDNAc7yMajoJW4f.png)

这个`AppClassLoader`就是应用程序加载器，`PlatformClassLoader`就是在jdk1.8中的`ExtClassLoader`，为扩展类加载器。

## 类加载器子系统

![class_loader_subsystem.png](https://i.loli.net/2020/03/17/rBvFEta6JcgiK5T.png)

类加载器呢在加载的时候又分为了几个阶段：

1. 加载
2. 链接
3. 初始化

首先看一下加载

### 加载

加载的步骤有下：

1. 通过类的全限定名加载到内存去，转化为字节流
2. 将字节流的静态存储结构转换成方法区的运行时的数据结构
3. 将运行时数据结构转换成class对象，作为方法区的

![classloader_subsystem_load.png](https://i.loli.net/2020/03/17/wCfnPbHTQs5BoxK.png)

我理解的图大概就是下面这个过程：

![classloader_sub_load.png](https://i.loli.net/2020/03/17/bJAY4sZ5f8GoFd6.png)

### 链接

![classloader_sub_link.png](https://i.loli.net/2020/03/17/Nm4KkLFSBOU5Wux.png)

#### 验证

首先先说一下验证吧，通过一个工具叫`Binary Viewer`加载class文件，可以看到，所有的class文件都是一个 `cafe baby`的前缀

![cafe_baby.png](https://i.loli.net/2020/03/17/I9XPl6swvnmBiuK.png)

这也是jvm对文件校验的一种保护机制

#### 准备

就是在加载的时候就给变量分配内存，设置初始值，int、long、float初始默认值为0，对象呢就是null。

下面的第一段程序是`.java`文件，而第二段程序是通过`ByteCode Viwer`反编译得到的一个代码段

```java
public class SubLink {
    public static int a = 1;

    public final static int c = 10;

    static {
        a = 3;
        b = 2;
    }

    public static int b = 1;

    public static void main(String[] args) {
        System.out.println(SubLink.a); // a=3
        System.out.println(SubLink.b); // b=1
    }
}
```



```java
public class com/kyriexu/classloadersubsystem/SubLink {
     <ClassVersion=55>
     <SourceFile=SubLink.java>

     public static int a;
     public static final int c = 10 (java.lang.Integer);
     public static int b;

     public SubLink() { // <init> //()V
         <localVar:index=0 , name=this , desc=Lcom/kyriexu/classloadersubsystem/SubLink;, sig=null, start=L1, end=L2>

         L1 {
             aload0 // reference to self
             invokespecial java/lang/Object.<init>()V
             return
         }
         L2 {
         }
     }

     public static main(java.lang.String[] arg0) { //([Ljava/lang/String;)V
         <localVar:index=0 , name=args , desc=[Ljava/lang/String;, sig=null, start=L1, end=L2>

         L1 {
             getstatic java/lang/System.out:java.io.PrintStream
             getstatic com/kyriexu/classloadersubsystem/SubLink.a:int
             invokevirtual java/io/PrintStream.println(I)V
         }
         L3 {
             getstatic java/lang/System.out:java.io.PrintStream
             getstatic com/kyriexu/classloadersubsystem/SubLink.b:int
             invokevirtual java/io/PrintStream.println(I)V
         }
         L4 {
             return
         }
         L2 {
         }
     }

     static  { // <clinit> //()V
         L1 {
             iconst_1
             putstatic com/kyriexu/classloadersubsystem/SubLink.a:int
         }
         L2 {
             iconst_3
             putstatic com/kyriexu/classloadersubsystem/SubLink.a:int
         }
         L3 {
             iconst_2
             putstatic com/kyriexu/classloadersubsystem/SubLink.b:int
         }
         L4 {
             iconst_1
             putstatic com/kyriexu/classloadersubsystem/SubLink.b:int
             return
         }
     }
}
```

从上面我们可以看到，在准备阶段程序将非final修饰的初始值给设置成了0，但是如果使用final修饰的呢 ，就在编译的时候就已经指定了初始值。

![class_file_prepare.png](https://i.loli.net/2020/03/17/HgtdPJQVhjFXwA4.png)



### 初始化

![classloader_sub_init_process.png](https://i.loli.net/2020/03/17/c1wsNehlZg4LDpi.png)

按照上图的意思的话，每次初始化就会有一个\<client\>的构造方法

![classloader_sub_init.png](https://i.loli.net/2020/03/17/PRnBaiq35lt9zu2.png)

![classloader_sub_init_construct.png](https://i.loli.net/2020/03/17/HZu7m1aLgGyCl4S.png)

并且上面的代码的输出结果是`a=3``b=1`的原因在上面这张图说明了，初始化顺序和语句的顺序有关

# 双亲委派机制

如果现在自定一个`package`名称是`java.lang`，并且在这个包下定义一个类`String`，那么这个`类`和java官方的`String`类的名称相同，所在的包也相同，如果在这个类中定义一个`main`方法，运行的话就会报错

![java_lang_string_error.png](https://i.loli.net/2020/03/16/GJZ4X9vcuKbCeVd.png)

这个就是双亲委派机制在起作用：

**双亲委派机制**就是当某个类加载器需要加载某个`.class`文件时，它首先把这个任务委托给他的**上级类加载器**，递归这个操作，如果上级的类加载器没有加载，自己才会去加载这个类。

![](https://upload-images.jianshu.io/upload_images/7634245-7b7882e1f4ea5d7d.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

而上面的那个例子就是最后执行了`bootstrapclassloader`，过程有下面几步：

1. 类加载器收到类加载的请求
2. 将请求委托给父类加载器完成，一直向上委托，直到启动类加载器(bootstrapclassloader)
3. 启动类加载器检查是否能够加载当前这个类，能加载就结束，使用当前的加载器，否则抛出异常，使用子类的加载器
4. 重复步骤3

比如自定义一个`Car`类，就会首先询问`BootStrapClassLoader`是否能找到这个`Car.class`，找不到，就再看`PlatFormClassLoader`，也没有就用`ApplicationClassLoader`

# 栈、堆、方法区

![内存功能.png](https://i.loli.net/2020/03/16/iMUeTK1ovSJblCm.png)

**8大基本数据类型**：byte、int、float、double、boolean、char、long、short都是存储在**栈**中的，还有**本地方法栈**以及**对象的引用**，真正的**对象实例**是存储在**堆**中的。

## 方法区、永久代、元空间

方法区和堆一样，都是**各个线程共享**的，它用于储存虚拟机加载的：类信息+普通常量+静态常量+编译器编译后的代码等待。方法区是堆的一个逻辑结构，只不过不对方法区进行垃圾回收。

方法区是一个规范，而永久代是这种规范的实现，是`HotSpot`对方法区的实现

## 静态常量池和运行时常量池

**静态常量池**，即*.class文件中的常量池，class文件中的常量池不仅仅包含字符串(数字)字面量，还包含类、方法的信息，占用class文件绝大部分空间。这种常量池主要用于存放两大类常量：**字面量**和**符号引用量**，**字面量**相当于Java语言层面常量的概念，如**文本字符串**，声明为final的常量值等，**符号引用**则属于编译原理方面的概念，包括了如下三种类型的常量：

+ 类和接口的全限定名
+ 字段名称和描述符
+ 方法名称和描述符

**运行时常量池**，则是jvm虚拟机在完成类装载操作后，将class文件中的常量池载入到内存中，并保存在方法区中，我们常说的常量池，就是指方法区中的运行时常量池。

在JDK1.6及之前，运行时常量池是方法区的一个部分，同时方法区里面**存储**了**类的元数据信息**(**字段和方法数据，以及方法和构造函数的代码**)、**静态变量**、**即时编译器编译后的代码**（比如spring 使用IOC或者AOP创建bean时，或者使用cglib，反射的形式动态生成class信息等）等。

### 常量池的存放位置

在**JDK1.7**及以前，HotSpot虚拟机将**java类信息、常量池、静态变量、即时编译器编译后的代码等数据**，存储在**Perm Space**（永久代）里（对于其他虚拟机如BEA JRockit、IBM J9等是不存在永久带概念的），类的元数据和静态变量在类加载的时候被分配到永久代里，当常量池回收或者类被卸载的时候，垃圾收集器会回收这一部分内存。

而在JDK1.7的最大的改动就是把常量池放在了堆中。

JDK1.8时，HotSpot虚拟机对JVM模型进行了改造，将**类元数据**放到了**本地内存**中，将**常量池**和**静态变量**放到了Java**堆**里

在JDK1.7及以后，JVM已经将运行时常量池从方法区中移了出来，在JVM堆开辟了一块区域存放常量池。

所以在**JDK1.7及以后**，运行时常量池都放在了**堆**中。

# 堆

一个jvm只能有一个堆，大小可以调节，所以线程是共享堆内存的

在类加载的时候，会把什么放进堆中呢？

+ 实例
+ 常量
+ 变量
+ 方法

堆分为三个部分：

+ 新生区
+ 养老区
+ 永久代

![heap.png](https://i.loli.net/2020/03/16/UxQal3go7ETOVX6.png)

对于新生代来说分为三个区的原因是为了方便采用复制-清除策略而采用的策略。

复制策略就是将原来存在的内存分为两个相等的区，使用一块进行新生代的内存分配，当要GC时，则将存活的对象复制进入另一块空闲的内存，然后将使用的内存进行清除，从而又有一个空闲区和一个使用区，并且不会有碎片问题。就是在GC的时候，Eden还活着的内存则放入Survivor区。

实际上并不需要两个1：1的分区比例，因为一般存活的对象很少，所以JVM聪明的讲新生代占据的总内存分为**Eden：Survivor from：Survivor to = 8:1:1**三部分，其中**Eden**就用来**分配新的对象内存**，Survivor **from**则**用于GC时的复制**。

那为什么需要两个Survivor区呢，因为复制后Survivor from区虽然现在很整齐，没有碎片，当下一次进行回收时，Eden区和Survivor from区里都存在需要回收的对象，则Survivor from区也会出现碎片。

所有的新生代首先会在Eden区进行内存分配，**当Eden区满**时会进行一次**Minor GC**操作，将Eden区进行回收，此时判断存活的对象会被复制进入Survivor from区（年龄加1）

对于**大对象和长期存活的对象直接进入老年代**

对于**大对象**来说，**为了保证Eden区具有充足的空间可用的一种策略**，采用**-XX:PretenureSizeThreshold**参数可以设置多大的对象可以直接进入老年代内存区域。

对于**长期存活对象**来说，**为了保证Eden区到Survivor区不会频繁的进行复制一直存活的对象且对Survivor区也能保证不会具有太多的一直占据的内存**，采用**-XX:MaxTenuringThreshold=数字** 参数可以设置对象在经过多少次GC后会被放入老年代（年龄达到设置值，默认为15）

在**发生MinorGC之前**，JVM会判断之前每次晋升到老年代的平均大小是否大于老年代剩余空间的大小，若大于则进行full GC。

总结一下上面的：

当进行minor gc的时候，eden区还活着的变量会进入survior区，当处于survior区域久了就会进入老年区，在进入老年区的时候，就会判定老年区是否满了，如果满了就进行一次full gc，然后再将变量放入老年区。

**永久区**是存放JDK自身携带的class对象，interface 元数据，存储的是Java的一些运行环境或者类信息，这个区域**不存在GC**，**关闭jvm**就会**自动**释放内存

