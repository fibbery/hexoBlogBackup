---
title: jvm字节码入门
date: 2019-09-19 10:35:54
categories: 学习笔记
tags:
    - java
    - jvm
---

## 前言
java为什么能”一次编译，多次运行“，因为java在任何环境下，通过编译都能生成一种固定格式的字节码（.class）文件。之所以被称之为字节码，是因为字节码文件由十六进制值组成，而JVM以两个十六进制值为一组，即以字节为单位进行读取。
<!-- more -->
以下是java文件从编译到运行的具体流程：
 ![jvm编译运行图](/assets/blogImg/bytecode_running.jpg)
所以我们从字节码文件说起，一次去探究代码如何run起来的

## 字节码文件认识
### 文件初览
.java文件编译后生产.class文件。新建一个源码文件如下：
![HelloJava源代码](/assets/blogImg/HelloJava.jpg)
使用xxd查看生成的16进制文件Hello.class，如下
![HelloJavaByteCode](/assets/blogImg/HelloJavaByteCode.jpg)
文件结构可以通过[JVM虚拟机规范](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html)得到，大致如下：
![ClassFile Structure](/assets/blogImg/ClassFileStructure.jpg)
> 1. 第一列表示字段的类型，第二列是该字段的名称；
> 2. U2、U4分别代表占用2字节和4个字节
> 3. _info结尾的代表class文件中的表，表也是由若干个无符号数和若干个表构成，用来表示复杂一些的class内容；
> 4. 整个class文件就是一个大的表；


1. MAGIC (魔法数) 0xCAFEBABE 
2. 版本号 0000 0034   （十进制52）
3. 0022  常量池 （十进制34，排除掉下标0的，只有33个）
...



### 字节码查看
#### 自带工具类
上述方式对我们看字节码具体内容并不是十分友好，所以自带的工具类javap可以让我们十分便利的通过字节码文件了解到类的实现细节。javap使用help如下：
![javap help](/assets/blogImg/javapusage.jpg)
我们通过```javap -v Hello.class```可以得到如下结果：
```
Classfile /Users/fibbery/Documents/workspace/demo/src/main/java/com/fibbery/demo/jvm/Hello.class
  Last modified 2019-9-19; size 518 bytes
  MD5 checksum d28a8ac83ec1ee573714d295476cf07b
  Compiled from "Hello.java"
public class Hello
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #6.#20         // java/lang/Object."<init>":()V
   #2 = Fieldref           #21.#22        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #23            // Hello World
   #4 = Methodref          #24.#25        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #26            // Hello
   #6 = Class              #27            // java/lang/Object
   #7 = Utf8               <init>
   #8 = Utf8               ()V
   #9 = Utf8               Code
  #10 = Utf8               LineNumberTable
  #11 = Utf8               LocalVariableTable
  #12 = Utf8               this
  #13 = Utf8               LHello;
  #14 = Utf8               main
  #15 = Utf8               ([Ljava/lang/String;)V
  #16 = Utf8               args
  #17 = Utf8               [Ljava/lang/String;
  #18 = Utf8               SourceFile
  #19 = Utf8               Hello.java
  #20 = NameAndType        #7:#8          // "<init>":()V
  #21 = Class              #28            // java/lang/System
  #22 = NameAndType        #29:#30        // out:Ljava/io/PrintStream;
  #23 = Utf8               Hello World
  #24 = Class              #31            // java/io/PrintStream
  #25 = NameAndType        #32:#33        // println:(Ljava/lang/String;)V
  #26 = Utf8               Hello
  #27 = Utf8               java/lang/Object
  #28 = Utf8               java/lang/System
  #29 = Utf8               out
  #30 = Utf8               Ljava/io/PrintStream;
  #31 = Utf8               java/io/PrintStream
  #32 = Utf8               println
  #33 = Utf8               (Ljava/lang/String;)V
{
  public Hello();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 1: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   LHello;

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Hello World
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 3: 0
        line 4: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  args   [Ljava/lang/String;
}
SourceFile: "Hello.java"

```

#### 其他工具类
但是每次都通过javap看字节码有需要重复很多无用的工作，一个Idea插件：[jclasslib](https://plugins.jetbrains.com/plugin/9248-jclasslib-bytecode-viewer)。代码编译后在菜单栏"View"中选择"Show Bytecode With jclasslib"，可以很直观地看到当前字节码的一些信息。
![jclasslib查看字节码](/assets/blogImg/jclasslib.jpg)

### 常见的指令
从上述字节码可以看到很多的指令使用，我们从[The Java Virtual Machine Instruction Set](https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-6.html#jvms-6.5.getstatic)是可以知道指令具体的动作是什么，接下来挑选几个作为实例讲解

#### 方法调用
方法调用指令分为5个：
1. invokestatic：用于调用静态方法
2. invokespecial：用于调用私有实例方法、构造器，以及使用 super 关键字调用父类的实例方法或构造器，和所实现接口的默认方法
3. invokevirtual：用于调用非私有实例方法（即public protected pacakge)
4. invokeinterface：用于调用接口方法
5. invokedynamic：用于调用动态方法


## 字节码运行原理

### 虚拟机结构
jvm虚拟机结构如下：
![jvm内存结构](/assets/blogImg/JvmMemoryStructure.jpg)
其中的Java虚拟机栈(Java Virtual Machine Stacks)也是线程私有的,它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型:每个方法在执行的同时都会创建一个栈帧(Stack Frame)用于存储局部变量表、操作数栈、动态链接、方法出口等信息。每一个方法从调用直至执行完成的过程,就对应着一个栈帧在虚拟机栈中入栈到出栈的过程。

### 栈帧
栈帧（Stack Frame）是用于支持虚拟机进行方法调用和方法执行的数据结构 栈帧随着方法调用而创建，随着方法结束而销毁，栈帧的存储空间分配在 Java 虚拟机栈中，每个栈帧拥有自己的**局部变量表(Local Variables)**、**操作数栈(Operate Stack)** 以及 **指向运行时常量池的引用**，如下图所示：
![StackFrame](/assets/blogImg/StackFrame.jpg)
* 局部变量表：每个栈帧内部都包含一组称为局部变量表（Local Variables）的变量列表，局部变量表的大小在编译期间就已经确定。Java 虚拟机使用局部变量表来完成方法调用时的参数传递，当一个方法被调用时，它的参数会被传递到从 0 开始的连续局部变量列表位置上。当一个实例方法（非静态方法）被调用时，第 0 个局部变量是调用这个实例方法的对象的引用（也就是我们所说的 this ）
* 操作数栈：每个栈帧内部都包含了一个称为操作数栈的后进先出（LIFO）栈，栈的大小同样也是在编译期间确定。Java 虚拟机提供的一些字节码指令用来从局部变量表或者对象实例的字段中复制常量或者变量到操作数栈，也有一些指令用于从操作数栈取走数据、操作数据和把操作结果重新入栈。在方法调用时，操作数栈也用来准备调用方法的参数和接收方法返回的结果。
整个方法的执行实际上是操作数栈和局部变量表之间不断load和store的过程


## 字节码工具
### ASM
对于需要手动操纵字节码的需求，可以使用ASM，它可以直接生产 .class字节码文件。以下是asm实现的架构：
![asm](/assets/blogImg/asm.jpg)

使用工具[ASM ByteCode Outline](https://plugins.jetbrains.com/plugin/5918-asm-bytecode-outline)操作更简单


### Javassist
ASM是在指令层次上操作字节码的，阅读上文后，我们的直观感受是在指令层次上操作字节码的框架实现起来比较晦涩。故除此之外，我们再简单介绍另外一类框架：强调源代码层次操作字节码的框架Javassist。
利用Javassist实现字节码增强时，可以无须关注字节码刻板的结构，其优点就在于编程简单。直接使用java编码的形式，而不需要了解虚拟机指令，就能动态改变类的结构或者动态生成类

## Java Instrumentation 包
字节码修改工具如果只能在运行前修改，那么应用的范围其实不是很广。由于对字节码需求修改的需求巨大，jdk1.5引入了java.lang.instrument包，可以通过addTransformer增加一个ClassFileTransformer，通过ClassFileTransformer完成类的转换。  
JDK1.5支持静态Instrumentation，具体实现是在jvm启动时候添加一个代理（javaagent)，这个代理是一个jar包，这个jar包MANIFEST.MF里指定了代理类，该代理类包含一个premain的方法，jvm在运行时先运行这个jar的premain方法，然后在执行具体的main方法，这种方式可以在class被加载前进行修改，无需对应用进行修改，就可以对类进行增强。大多数apm产品都是利用这种方式实现
JDK1.6就支持更强大的动态Instrumentation，在JVM 启动后通过 Attach API 远程加载代理包到应用上，这种称为agentmain。


