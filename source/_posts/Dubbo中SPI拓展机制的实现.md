---
title: Dubbo中SPI拓展机制的实现
date: 2018-01-15 23:09:56
categories: java
tags:
	- dubbo
	- spi

---

初步说明下java原生的spi拓展加载机制：
> 扫描META-INF/services下面的所有文件，文件都是使用接口或者抽象类名字作为文件名，具体实现类作为文件内容构成，在使用的时候只能通过遍历来实现查找和实例化，有可能会一次性把所有的实现都实例化，造成不必要的浪费。

<!-- more -->

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

### 拓展点自适应
很多时候拓展点是有多个实现的，我们需要给定一个参数来让使用者决定使用哪个拓展点，在dubbo中我们是使用com.alibaba.dubbo.common.URL来传递这样一个信息。因此，ExtensionLoader会有一个唯一的AdptiveClass(适配类)，然后通过URL这样一个变量容器来决定使用哪个具体的拓展点实现。当然，如果一个拓展点只有一个实现类，是不需要AdptiveClass的。获取适配类的方法为getAdaptiveExtension流程如下：
![Dubbo的ExtensionLoader获取适配类流程](/assets/blogImg/Dubbo的ExtensionLoader获取适配类流程.png)
所以，要让拓展点支持自适应，需要满足以下条件：
1. 拓展点实现类有@Adaptive注解
2. 拓展点接口方法有@Adaptive注解并且对应接口方法参数是com.alibaba.dubbo.common.URL或者包含com.alibaba.dubbo.common.URL属性  
使用适配类依赖注入的demo如下：
```java
@SPI("impl1")
public interface SimpleExt {
    // @Adaptive example, do not specify a explicit key.
    @Adaptive
    String echo(URL url, String s);

    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);

    // no @Adaptive
    String bang(URL url, int i);
}

public class SimpleExtImpl1 implements SimpleExt {
    public String echo(URL url, String s) {
        return "Ext1Impl1-echo";
    }

    public String yell(URL url, String s) {
        return "Ext1Impl1-yell";
    }

    public String bang(URL url, int i) {
        return "bang1";
    }
}

public class SimpleExtImpl2 implements SimpleExt {
    public String echo(URL url, String s) {
        return "Ext1Impl2-echo";
    }

    public String yell(URL url, String s) {
        return "Ext1Impl2-yell";
    }

    public String bang(URL url, int i) {
        return "bang2";
    }

}

//测试demo如下
   @Test
    public void test_getAdaptiveExtension_defaultAdaptiveKey() throws Exception {
        {
            SimpleExt ext = ExtensionLoader.getExtensionLoader(SimpleExt.class).getAdaptiveExtension();

            Map<String, String> map = new HashMap<String, String>();
            URL url = new URL("p1", "1.2.3.4", 1010, "path1", map);

            String echo = ext.echo(url, "haha");
            assertEquals("Ext1Impl1-echo", echo);
        }

        {
            SimpleExt ext = ExtensionLoader.getExtensionLoader(SimpleExt.class).getAdaptiveExtension();

            Map<String, String> map = new HashMap<String, String>();
            map.put("simple.ext", "impl2");
            URL url = new URL("p1", "1.2.3.4", 1010, "path1", map);

            String echo = ext.echo(url, "haha");
            assertEquals("Ext1Impl2-echo", echo);
        }
    }
```
关于拓展点再com.alibaba.dubbo.common.URL中的参数名字确定是由本身类名决定的，带入如下：
```java
/** ExtensionLoader.createAdaptiveExtensionClassCode()**/
 if (value.length == 0) {
                    char[] charArray = type.getSimpleName().toCharArray();
                    StringBuilder sb = new StringBuilder(128);
                    for (int i = 0; i < charArray.length; i++) {
                        if (Character.isUpperCase(charArray[i])) {
                            if (i != 0) {
                                sb.append(".");
                            }
                            sb.append(Character.toLowerCase(charArray[i]));
                        } else {
                            sb.append(charArray[i]);
                        }
                    }
                    value = new String[]{sb.toString()};
                }
```
例如SimpleExt接口，适配类会根据com.alibaba.dubbo.common.URL中参数simple.ext的值来获取对应拓展点，同时意味着注入的拓展点可以根据URL参数不同而变化。

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
上述代码可以知道，往拓展点注入其他的拓展点需要满足以下几点：
1. 该拓展点有set其他拓展点的方法
2. 如果是spi注入方式的拓展，那么注入的拓展点类必须满足拓展点自适应的要求，那么可以保证注入的拓展点其实是此类拓展点的Adaptive实例。(这一点主要跟注入的拓展点的拓展方式有关，目前已知的就SpiExtensionFactory和SpringExtensionFactory，意味着除了能注入spi拓展还能注入spring容器中的对象)  

对于第二点，我们可以深挖代码来解答，代码顺序如下：
```java
/** ExtensionLoader.injectExtension**/
Object object = objectFactory.getExtension(pt,property);
		||
/** AdaptiveExtensionFactory.getExtension **/
T extension = factory.getExtension(type,name)
		||
SpringExtensionFactory.getExtension()/SpiExtensionFactory
```

### 拓展点自动激活

这个特性是为了简化配置产生的，因为少数情况下需要获取到一些集合实现类实现链式操作，例如Filter。此时使用@Activate注解就能直接从cachedActivates获取对应class(cachedActivates在ExtensionLoader初始化的时候已经加载好，用拓展点实现的name作为key，用@Activate注解的内容作为value存储)。具体代码demo可以参考dubbo的ProtocolFilterWrapper.buildInvokerChain。Activate主要配合*ExtensionLoader.getActivateExtension(URL url, String key, String group)*使用，其中各个字段的释义分别如下

* group : 当组名适配的时候激活当前的拓展类
* value： 如果当前组名适配并且URL中有当前参数才能激活此拓展类
* order：绝对顺序 
* before：相对顺序(在某个拓展之前)
* after：相对顺序(在某个拓展之后)