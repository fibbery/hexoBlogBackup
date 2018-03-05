---
title: Dubbo通信实现
date: 2018-03-03 21:28:03
categories: java
tags:
	- dubbo
	- netty
---

## 前提

基于dubbo默认通讯服务器Netty来解析服务端和客户端如何通讯
<!-- more -->

## 具体实现

### 暴露服务

首先我们看下协议接口类Protocol中暴露服务的方法

```java
/**
 * 暴露远程调用服务：
 * 1. 该通信协议必须在接到请求之后记录请求的源地址：
 *		RpcContext.getContext().setRemoteAddress()
 * 2. export必须是幂等的。意味着使用相同的URL不论是调用一次还是调用两次，结果都是一样的
 * 3. invoker实例是由框架实现的，跟协议没任何关系
 * @param T 服务的类型
 * @param invoker 服务实现的实例
 * @return 返回被暴露服务实例的引用，用于取消服务
 */
@Adaptive
<T> Exporter<T> export(Invoker<T> invoker) throws RpcException;
```

然后我们看分析dubbo协议的暴露服务的具体实现DubboProtocol.export的过程如下

```
export ---> openServer ---> createServer --->  Exchangers.bind(url,handler)
```

openServer根据invoker携带的URL信息获取host以及ip，如果已经存在，则不会创建服务。
然后我们继续看下*Exchanger*的实现

```java
 public static ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        if (url == null) {
            throw new IllegalArgumentException("url == null");
        }
        if (handler == null) {
            throw new IllegalArgumentException("handler == null");
        }
        url = url.addParameterIfAbsent(Constants.CODEC_KEY, "exchange");
        return getExchanger(url).bind(url, handler);
    }
```

*getExchanger(url)*典型的策略模式，得到HeaderExchanger，代码如下

```java
 public ExchangeServer bind(URL url, ExchangeHandler handler) throws RemotingException {
        return new HeaderExchangeServer(Transporters.bind(url, new DecodeHandler(new HeaderExchangeHandler(handler))));
    }
```

1. *HeaderExchangeServer*其实是真正server的包裹类，提供了server的一些操作接口已经对server的一个心跳检测功能。
2. Transporters.bind这一步就是根据url选择对应的通信框架开启服务(此例中采用NettyTransport)。
3. new DecodeHandler(new HeaderExchangeHandler(handler)))采用装饰器模式构造服务端通道的handler

> 此处不讨论DecodeHandler如何对该协议通道的信息decode以及encode的过程

### 引用服务

我们看下Protocol中引用服务的方法

```java
   /**
     * 引用远程服务
     * 1.当使用者调用invoker(通信协议使用refer方法生成)的invoke方法的时候，通信协议
     * 会相应的让服务端对应的invoker调用invoke方法。
     * 2.refer返回的invoker实例是由通信协议生成实现的。实现的过程只不过是将方法调用的过程
     * 封装成用远程请求的方式。
     * 3.当URL设置check=false的时候，客户端不应该在连接服务端失败的时候抛出异常，并且有
     * 义务尝试重连
     * @param <T> 远程服务的类型
     * @param type 远程服务类的class
     * @param url 远程服务的url
     * @return invoker 远程服务的本地代理实现
     */
@Adaptive
<T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;
```
我们还是通过DubboProtocol来看客户端如何实现远程调用，refer方法在初始的时候会初始化一个与对应远端的连接，同时返回持有这个连接的对象invoker。我们需要将其转换成对应需要使用的服务类，在这里就需要用到动态代理这样的一个方式来生成一个本地句柄，以便客户端像调用本地服务一样调用。所以，一般客户端使用如下(伪码)：
```java
ProxyFactory proxyFactory = ExtensionLoader.getExtensionLoader(ProxyFactory.class).getAdaptiveExtension();
DemoService service = proxyFactory.getProxy(refer(type,url)); //本地生成代理类
service.echo(); //调用方法
```
接下来我们具体看JavassistProxyFactory是怎么生成一个客户端需要的代理类的，代码如下：
```java
   public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```
InvokerInvocationHandler这个类是很明显的装饰器模式，让属于Object类中的方法以及toString、hashCode、equals一些通用方法只在本地调用，实际上我们还是使用的DubboInvoker的invoker方法，也就是其中具体的doInvoke方法。代码点实现也只是使用返回的invoker对象中持有的客户端连接来发送请求消息，获取返回结果。
> dubbo暴露方法和对远程方法的调用可以阅读源码DubboProtocolTest.testDemoProtocol来了解
>

### 对消息的编码和解码
上面了解到远程调用的过程实际上是本地发送请求消息，远端根据请求返回结果的一个过程，这就需要我们对传输过程中消息的编码/解码有一个清晰的认识，我们仍然用Netty作为传输层来阐明Dubbo消息的编码/解码。
1. 




