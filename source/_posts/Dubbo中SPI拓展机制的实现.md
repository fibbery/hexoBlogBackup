---
title: Dubbo中SPI拓展机制的实现
date: 2018-01-15 23:09:56
tags:
	- dubbo
	- spi
---
初步说明下java原生的spi拓展加载机制：
> 扫描META-INF/services下面的所有文件，文件都是使用接口或者抽象类名字作为文件名，具体实现类作为文件内容构成，在使用的时候只能通过遍历来实现查找和实例化，有可能会一次性把所有的实现都实例化，造成不必要的浪费。

dubbo在原有的基础上拓展了以下几点：
1. 根据需要实例化拓展点。
2. 自动注入需要依赖的拓展点，换而言之，就是增加了拓展点IoC和AOP的支持，一个拓展点能注入其他拓展点。
  同时，dubbo读取文件方式也增加了，放置扩展点配置文件夹在原来的基础上增加了 `META-INF/dubbo/接口全限定名`以及`META-INF/dubbo/internal/接口全限定名`，内容为：`配置名=扩展实现类全限定名`，多个实现类用换行符分隔。
  dubbo拓展点都是依赖ExtensionLoader的getExtension方法加载的，其中ExtensionLoader的type表明需要加载的的类的所属类型。其中getExtension的初次获取流程如下：
  ![Dubbo-ExtensonLoader获取拓展点实现流程](/assets/blogImg/Dubbo-ExtensonLoader获取拓展点实现流程.png)

通过上图流程可以分析出拓展点的以下属性
### 拓展点自动包装
自动包装拓展的wrapper类也是拓展点的实现类，ExtensionLoader在加载拓展点的时候，会将拥有该类型的构造函数的拓展点当做wrapper类。代码如下：
```java
//代码点ExtensionLoader.loadFile()
clazz.getConstructor(type);
Set<Class<?>> wrappers = cacheWrapperClasses;
if(wrappers == null){
  cacheWrapperClasses = new ConcurrentHashSet<Class<?>>();
  wrappers = cacheWrapperClasses;
}
wrappers.add(clazz);
```
当使用ExtensionLoader返回拓展点时，返回的其实是wrapper类的实例，该实例持有实际的拓展点实现。
这种拓展类可以有很多个，可以根据实际需要新增。
所以可以把所有拓展点的公共逻辑都置于wrapper类中，类似于aop，wrapper代理了实际需要的拓展点。

### 扩展点自动注入
injectExtension代码如下：
```java
//ExtensionLoader.injectExtension
if(objectFactory != null){
  /*判断实例中是否有set方法，该set方法满足只有一个参数并且是public的*/
  for(Method method: instance.getClass().getMethods()){
    if(method.getName().startWith("set") && method.getParameterTypes().length == 1
       && Modifier.isPublic(method.getModifier())){
      Class<?> pt = method.getParameterTypes()[0]; //获取set方法的参数类型
      String property = method.getName().length() > 3 method.getName().substring(3,4).toLowerCase() 						+ method.getName().subtring(4) : ""; //获取该set方法的属性名字
      Object object = objectFactory.getExtension(pt,property); //获取对应pt类型的adaptive类来注入
      if(object != null){
        method.invoke(instance,object);
      }
    }
  }
}
```
上述代码可以知道，

### 拓展点自适应
一般来说，拓展点都有都有许多实现。


### 拓展点自动激活