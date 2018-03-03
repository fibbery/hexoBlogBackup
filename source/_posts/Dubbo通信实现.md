---
title: Dubbo通信实现
date: 2018-03-03 21:28:03
categories: java
tags:
	- dubbo
	- netty
	- filter
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

```
 





