# JVM的整体结构

都是针对Hotspot虚拟机













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