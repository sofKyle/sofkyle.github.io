---
layout: post
title: Dubbo Consumer Invocation Chain
tags: [Dubbo, ]

---

```
ReferenceAnnotationBeanPostProcessor#invoke
    |
    |
    V
InvokerInvocationHandler#invoke
    |
    |----> RpcInvocation
    V
MockClusterInvoker#invoke
    |
    | no mock
    V
AbstractClusterInvoker#invoke
    |
    |----> list invokers ----> AbstractClusterInvoker#initLoadBalance ----> ExtensionLoader#getExtensionLoader ----> ExtensionLoader#createExtension
    |
    |----> RpcUtils#attachInvocationIdIfAsync
    V
FailoverClusterInvoker#doInvoke
    |
    |----> retry loop
    V
InvokerWrapper#invoke
    |
    |
    V
ListenerInvokerWrapper#invoke
    |
    |
    V
ProtocolFilterWrapper.CallbackRegistrationInvoker#invoke
    |
    |
    V
ProtocolFilterWrapper#invoke
    |
    |
    V
ConsumerContextFilter#invoke
    |
    |----> setLocalAddress ----> setRemoteAddress ----> RpcContext#removeServerContext
    V
AsyncToSyncInvoker#invoke
    |
    |
    V
AbstractInvoker#invoke
    |
    |
    V
DubboInvoker#doInvoke
    |
    |----> new AsyncRpcResult(inv) ----> ExchangeClient#request ----> ReferenceCountExchangeClient#request ----> HeaderExchangeClient#request ----> HeaderExchangeChannel#request ----> AbstractPeer#send ----> AbstractClient#send ----> AbstractClient#getChannel
    V
NettyChannel#send: 通过Netty传送RpcInvocation对象
```