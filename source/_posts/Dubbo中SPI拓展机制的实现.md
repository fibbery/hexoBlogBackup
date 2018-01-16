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

## 拓展点特性
### 拓展点自适应

### 扩展点自动装配
### 拓展点拓展包装
### 拓展点自动激活