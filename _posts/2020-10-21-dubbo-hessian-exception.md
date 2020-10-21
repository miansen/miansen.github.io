---
layout: post
title: dubbo hessian 反序列化报错
date: 2020-10-21
categories: 杂七杂八
tags: dubbo hessian
author: 龙德
---

* content
{:toc}

今天工作中遇到一个问题，dubbo 服务A调用服务B，服务A报空指针异常，反序列化失败。异常信息如下：

```java
Failed to invoke the method getChainById in the service cn.tanzhou.chain.config.stub.service.ChainAPI. Tried 1 times of the providers [10.0.65.146:20882] (1/2) from the registry 192.168.1.20:2181 on the consumer 192.168.78.1 using the dubbo version 2.6.2. Last error is: Failed to invoke remote method: getChainById, provider: dubbo://10.0.65.146:20882/cn.tanzhou.chain.config.stub.service.ChainAPI?anyhost=true&application=chain-scheduling-dubbo&check=false&default.check=false&default.retries=0&default.service.filter=providerTraceIdFilter&default.timeout=3000&dubbo=2.6.2&generic=false&interface=cn.tanzhou.chain.config.stub.service.ChainAPI&methods=getChainById,page&organization=tz&owner=chenyx&pid=3064&register.ip=192.168.78.1&remote.timestamp=1602745219717&revision=1.0.0&side=consumer&timestamp=1602747456735&version=1.0.0, cause: com.alibaba.com.caucho.hessian.io.HessianFieldException: cn.tanzhou.chain.config.stub.domain.response.rule.AbstractEvent.childrenNodesOfTrue: 'cn.tanzhou.chain.config.stub.domain.response.rule.WaitEvent' could not be instantiated
com.alibaba.com.caucho.hessian.io.HessianFieldException: cn.tanzhou.chain.config.stub.domain.response.rule.AbstractEvent.childrenNodesOfTrue: 'cn.tanzhou.chain.config.stub.domain.response.rule.WaitEvent' could not be instantiated
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.logDeserializeError(JavaDeserializer.java:167)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectListFieldDeserializer.deserialize(JavaDeserializer.java:531)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:276)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:203)
    at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:526)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2810)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2750)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2279)
    at com.alibaba.com.caucho.hessian.io.CollectionDeserializer.readLengthList(CollectionDeserializer.java:122)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2251)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectListFieldDeserializer.deserialize(JavaDeserializer.java:525)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:276)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:203)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2808)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2146)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2075)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2119)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2075)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectFieldDeserializer.deserialize(JavaDeserializer.java:408)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:276)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:203)
    at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:526)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2810)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2750)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2279)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2724)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2279)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2081)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2075)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectFieldDeserializer.deserialize(JavaDeserializer.java:408)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:276)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:203)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2808)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2146)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2075)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2119)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2075)
    at com.alibaba.dubbo.common.serialize.hessian2.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:91)
    at com.alibaba.dubbo.common.serialize.hessian2.Hessian2ObjectInput.readObject(Hessian2ObjectInput.java:96)
    at com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:83)
    at com.alibaba.dubbo.rpc.protocol.dubbo.DecodeableRpcResult.decode(DecodeableRpcResult.java:113)
    at com.alibaba.dubbo.rpc.protocol.dubbo.DubboCodec.decodeBody(DubboCodec.java:89)
    at com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:124)
    at com.alibaba.dubbo.remoting.exchange.codec.ExchangeCodec.decode(ExchangeCodec.java:84)
    at com.alibaba.dubbo.rpc.protocol.dubbo.DubboCountCodec.decode(DubboCountCodec.java:46)
    at com.alibaba.dubbo.remoting.transport.netty.NettyCodecAdapter$InternalDecoder.messageReceived(NettyCodecAdapter.java:133)
    at org.jboss.netty.channel.SimpleChannelUpstreamHandler.handleUpstream(SimpleChannelUpstreamHandler.java:80)
    at org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:564)
    at org.jboss.netty.channel.DefaultChannelPipeline.sendUpstream(DefaultChannelPipeline.java:559)
    at org.jboss.netty.channel.Channels.fireMessageReceived(Channels.java:274)
    at org.jboss.netty.channel.Channels.fireMessageReceived(Channels.java:261)
    at org.jboss.netty.channel.socket.nio.NioWorker.read(NioWorker.java:349)
    at org.jboss.netty.channel.socket.nio.NioWorker.processSelectedKeys(NioWorker.java:280)
    at org.jboss.netty.channel.socket.nio.NioWorker.run(NioWorker.java:200)
    at org.jboss.netty.util.ThreadRenamingRunnable.run(ThreadRenamingRunnable.java:108)
    at org.jboss.netty.util.internal.DeadLockProofWorker$1.run(DeadLockProofWorker.java:44)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)
    at java.lang.Thread.run(Thread.java:748)
Caused by: com.alibaba.com.caucho.hessian.io.HessianProtocolException: 'cn.tanzhou.chain.config.stub.domain.response.rule.WaitEvent' could not be instantiated
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:316)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.readObject(JavaDeserializer.java:201)
    at com.alibaba.com.caucho.hessian.io.SerializerFactory.readObject(SerializerFactory.java:526)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObjectInstance(Hessian2Input.java:2810)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2750)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2279)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2724)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2279)
    at com.alibaba.com.caucho.hessian.io.CollectionDeserializer.readLengthList(CollectionDeserializer.java:122)
    at com.alibaba.com.caucho.hessian.io.Hessian2Input.readObject(Hessian2Input.java:2251)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer$ObjectListFieldDeserializer.deserialize(JavaDeserializer.java:525)
    ... 57 more
Caused by: java.lang.reflect.InvocationTargetException
    at sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
    at sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
    at sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    at java.lang.reflect.Constructor.newInstance(Constructor.java:423)
    at com.alibaba.com.caucho.hessian.io.JavaDeserializer.instantiate(JavaDeserializer.java:312)
    ... 67 more
Caused by: java.lang.NullPointerException
    at cn.tanzhou.chain.config.stub.domain.response.rule.WaitEvent.init(WaitEvent.java:63)
    at cn.tanzhou.chain.config.stub.domain.response.rule.WaitEvent.<init>(WaitEvent.java:59)
    ... 72 more
```

可以看到最根本的异常是：java.lang.NullPointerException。抛出此异常的方法是 init() 方法，构造函数调用了 init() 方法。所以我们先去看一下构造函数。

构造函数如下：

```java
public WaitEvent(Long nodeId, Long parentNodeId, Long chainId, Long eventId, String eventName, String eventCode,
                     Map<String, Object> waitEventProperties) {
        super(nodeId, parentNodeId, chainId, NodeEnum.NODE_2002, eventId, eventName, eventCode);
        init(waitEventProperties);
    }

private void init(Map<String, Object> waitEventProperties) {
    Object waitSecond = waitEventProperties.get("waitSecond");
    Object afterDays = waitEventProperties.get("afterDays");
    Object atTimeOfSecond = waitEventProperties.get("atTimeOfSecond");
    Object waitType = waitEventProperties.get("waitType");
    Object selectType = waitEventProperties.get("selectType");
    Object courseId = waitEventProperties.get("courseId");
    Object courseTimeOfSecond = waitEventProperties.get("courseTimeOfSecond");
    Object courseTimeType = waitEventProperties.get("courseTimeType");
    ......
```

猜想一下应该是序列化的时候调用了这个构造方法，参数传的都是 null，所以就报了空指针异常。

我 debug 跟了一下，发现参数果然都是 null。

![image](https://miansen.wang/assets/dubbo-hessian-exception-1.png)

![image](https://miansen.wang/assets/dubbo-hessian-exception-2.png)

在 JavaDeserializer 类的构造方法有这么一段：

```java
public JavaDeserializer(Class cl) {
        _type = cl;
        _fieldMap = getFieldMap(cl);

        _readResolve = getReadResolve(cl);

        if (_readResolve != null) {
            _readResolve.setAccessible(true);
        }

        Constructor[] constructors = cl.getDeclaredConstructors();
        long bestCost = Long.MAX_VALUE;

        for (int i = 0; i < constructors.length; i++) {
            Class[] param = constructors[i].getParameterTypes();
            long cost = 0;

            for (int j = 0; j < param.length; j++) {
                cost = 4 * cost;

                if (Object.class.equals(param[j]))
                    cost += 1;
                else if (String.class.equals(param[j]))
                    cost += 2;
                else if (int.class.equals(param[j]))
                    cost += 3;
                else if (long.class.equals(param[j]))
                    cost += 4;
                else if (param[j].isPrimitive())
                    cost += 5;
                else
                    cost += 6;
            }

            if (cost < 0 || cost > (1 << 48))
                cost = 1 << 48;

            cost += (long) param.length << 48;

            if (cost < bestCost) {
                _constructor = constructors[i];
                bestCost = cost;
            }
        }

        if (_constructor != null) {
            _constructor.setAccessible(true);
            Class[] params = _constructor.getParameterTypes();
            _constructorArgs = new Object[params.length];
            for (int i = 0; i < params.length; i++) {
                _constructorArgs[i] = getParamArg(params[i]);
            }
        }
    }
```

上面这段代码，是 hessian 在反序列化的时候，用于在被反序列化的类里面找一个“得分最低”的构造函数，反序列化时会加以调用。构造函数的“得分”规则大致是：参数越少得分越低；参数个数相同时，参数类型越接近 JDK 内置类得分越低。而我们的调用返回的类只有一个构造函数，当然只有这个构造函数会被选中 但是，hessian 反序列化调用被选中的构造函数时，是这样来创造该构造函数需要的参数的：

```java
protected static Object getParamArg(Class cl) {
    if (!cl.isPrimitive())
        return null;
    else if (boolean.class.equals(cl))
        return Boolean.FALSE;
    else if (byte.class.equals(cl))
        return new Byte((byte) 0);
    else if (short.class.equals(cl))
        return new Short((short) 0);
    else if (char.class.equals(cl))
        return new Character((char) 0);
    else if (int.class.equals(cl))
        return Integer.valueOf(0);
    else if (long.class.equals(cl))
        return Long.valueOf(0);
    else if (float.class.equals(cl))
        return Float.valueOf(0);
    else if (double.class.equals(cl))
        return Double.valueOf(0);
    else
        throw new UnsupportedOperationException();
}
```

可以看到，如果参数不是 primitive 类型，会被 null 代替。但不巧的是我们的构造函数内部对该参数调用了一个方法，因此抛出了空指针异常。 也就是说这个空指针异常是抛在客户端反序列化的时候而不是服务端内部，因此服务端当然找不到对应的错误日志了。解决的办法也很简单：可以新增一个无参构造函数（无参构造函数肯定“得分”最低一定会被选中），或者修改代码保证任意参数为 null 的时候都不会出问题。

我最后增加了一个空参构造函数解决了这个问题。

**总结：通过 dubbo 服务传递的对象，需要保证有无参构造函数，避免出现反序列化失败的问题。**

但是还是有一点疑问，既然这样的话，为啥刚开始测试并没有出现这个问题，而是后来才突然出现，按理说这个问题一开始就应该出现才对，至今仍然困惑中。。。

参考资料：[hessian反序列化问题](https://www.dazhuanlan.com/2020/01/31/5e3436747abd4)