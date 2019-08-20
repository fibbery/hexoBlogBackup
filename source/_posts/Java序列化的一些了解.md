---
title: Java序列化的一些了解
date: 2018-03-06 23:53:54
categories: 学习笔记
tags:
    - java
    - serializable
---

## 引言
通常来说序列化和反序列化通常是一起的。我们常见的就是你只要将类实现Serializable接口，就能将该类的对象序列化二进制文件，同时能通过反序列化将二进制文件还原成对象。
<!-- more -->

## 具体实现
我们通过Demo来了解Java序列化和反序列化的具体情况，代码如下：
```JAVA
public class Test implements Serializable{

    private static final long serialVersionUID = -7303637756547202487L;

    public static int staticVar = 5;

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        //进行序列化到二进制文件中
        File file = new File("out");
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(file));
        out.writeObject(new Test());
        out.close();
        //从二进制文件中反序列化成对象
        ObjectInputStream ins = new ObjectInputStream(new FileInputStream(file));
        Test o = (Test) ins.readObject();
        ins.close();
        System.out.println(o.staticVar);
    }
}
```
针对以上Demo可以知道Java的序列化和反序列化其实就是使用ObjectInputStream和ObjectInputStream对实现了Serializable类的读写。深入代码可以发现，在序列化过程中，虚拟机会试图调用对象类里的 writeObject 和 readObject 方法，进行用户自定义的序列化和反序列化，如果没有这样的方法，则默认调用是 ObjectOutputStream 的 defaultWriteObject 方法以及 ObjectInputStream 的 defaultReadObject 方法。

## 序列化的特点
1. 如果某个类可以被序列化，其子类也可以被序列化。但是如果子类实现序列化，父类没有，那么子类反序列化会将父类那部分属性丢失。
2. 声明为static和transient类型的成员数据不能被序列化，因为static代表类的状态，而transient代表对象的临时数据。
3. Java 序列化机制为了节省磁盘空间，具有特定的存储规则，当写入文件的为同一对象时，并不会再将对象的内容进行存储，而只是再次存储一份引用。

根据第三点使用Demo详细说明下：
```JAVA
public class Reference implements Serializable {

    private static final long serialVersionUID = -8386243968193725155L;

    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public static void main(String[] args) throws IOException, ClassNotFoundException {
        Reference reference = new Reference();
        reference.setName("java");

        File file = new File("reference");
        ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream(file));
        os.writeObject(reference);
        //reference.setName("python")   ---- 展现特性的情况
        os.writeObject(reference); //再次写入

        ObjectInputStream is = new ObjectInputStream(new FileInputStream(file));
        Reference reference1 = (Reference) is.readObject();
        Reference reference2 = (Reference) is.readObject();
        System.out.println(reference1 == reference2);
    }
}

```
输出结果是true，同时比较写入一个对象的文件大小和写入两次相同对象文件的大小，发现没有呈2倍关系，证明其实第二次写入的只是一个引用而已。同时我们释放Demo的特性情况注释，发现结果还是True，说明在写入第一个对象之后，虽然改变了对象的成员变量，但是成员地址没有发生改变，虚拟机在进行第二次写入的时候，发现了一个相同的对象(地址相同)已经写入文件，因此只保存了一个引用，所以读取时，返回的仍然是第一次的对象。

> 使用一个文件多次writeObject的时候需要注意这个问题

## 一些序列化的特性以及对应场景
### 序列化ID的作用
#### 由来
Java的序列化机制是通过在运行时判断类的serialVersionUID来验证版本一致性的。在进行反序列化时，JVM会把传来的字节流中的serialVersionUID与本地相应实体（类）的serialVersionUID进行比较，如果相同就认为是一致的，可以进行反序列化，否则就会出现序列化版本不一致的异常。当实现java.io.Serializable接口的实体（类）没有显式地定义一个名为serialVersionUID，类型为long的变量时，Java序列化机制会根据编译的class自动生成一个serialVersionUID作序列化版本比较用，这种情况下，只有同一次编译生成的class才会生成相同的serialVersionUID。因此为了实现序列化接口的实体能够兼容先前版本，最好显式地定义一个名为serialVersionUID类型为long的变量，这样就不会存在版本不一致的问题。
序列化 ID 在 Eclipse 下提供了两种生成策略，一个是固定的 1L，一个是随机生成一个不重复的 long 类型数据（实际上是使用 JDK 工具生成），在这里有一个建议，如果没有特殊需求，就是用默认的 1L 就可以，这样可以确保代码一致时反序列化成功。那么随机生成的序列化 ID 有什么作用呢，有些时候，通过改变序列化 ID 可以用来限制某些用户的使用。

#### 应对场景
Java中RMI(Remoting Method Inovke)的实现


### 父类序列化和Transient关键字
#### 由来
> 参见[序列化特点](#序列化的特点)

#### 应对场景
平常我们都是使用transient关键字使字段不被序列化，但是如果有过多相同的类有同样的字段不需要序列化，我们可以将这些成员变量抽出来做成虚基类，根据父类对象序列化的规则，只要父类不实现Serializable接口，这写成员变量都不会被序列化。

### 敏感字段加密
#### 由来
> 参见[具体实现](#具体实现)

#### 应对场景
一些敏感信息不便于传输，可以对敏感字段进行加密，只有正确的客户端和服务端能加密和解密，避免传输过程中泄漏。因此可以在对象里自定义writeObject 和 readObject 方法，这样就可以完全控制对象序列化的过程，从而可以在序列化的过程中对某些数据进行加解密操作。
Demo代码如下：
```java
public class Account implements Serializable {

    private static final long serialVersionUID = -5790119844861945929L;

    private String password = "pass";

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    private void writeObject(ObjectOutputStream os) {
        try {
            ObjectOutputStream.PutField field = os.putFields();
            System.out.println(" 原始密码: " + password);
            String encryptedPass = encrypt(password);//伪码模拟加密
            field.put("password", encryptedPass); 
            os.writeFields();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    private void readObject(ObjectInputStream is) {
        try {
            ObjectInputStream.GetField field = is.readFields();
            Object encryptPass = field.get("password", "");
            System.out.println("获取加密的字符串：" + encryptPass);
            String decryptPass = decryptPass(encryptPass); //伪码模拟解密
            password = decryptPass; 
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }


    public static void main(String[] args) throws IOException, ClassNotFoundException {
        ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream(new File("password")));
        os.writeObject(new Account());
        os.close();

        ObjectInputStream is = new ObjectInputStream(new FileInputStream(new File("password")));
        Account o = (Account) is.readObject();
        System.out.println("解密后的密码是：" + o.password);
    }

}
```


## Reference

1.[Java中的序列化Serialable高级详解](http://blog.csdn.net/jiangwei0910410003/article/details/18989711)
2.[Java序列化的作用](http://www.oschina.net/question/4873_23270)

