---
layout: post
title: Spring AOP代理构造流程
tags: [Spring, ]

---

总的说来，Spring AOP代理对象的生成贯穿了如下两个阶段：一、BeanFactory的初始化；二、Bean的加载

#### 1、初始化
主要流程为：启动ApplicationContext的时候，将AnnotationAwareAspectJAutoProxyCreator注册至AbstractBeanFactory中的**beanPostProcessors**。  

> **PostProcessor**是Spring中很重要的一个扩展点概念，主要包括：BeanPostProcessor，BeanFactoryPostProcessor，分别作用于Bean的加载阶段于BeanFactory的加载阶段。Spring AOP的实质，就是扩展了BeanPostProcessor，利用JdkDynamicAopProxy或是CglibAopProxy构造代理对象，将代理方法与Advise关联起来。  
> BeanPostProcessor有两个方法：postProcessBeforeInitialization、postProcessAfterInitialization，分别代表着Bean实例化前的回调与Bean实例化后的回调。  

基本的调用流程如下：  
```java
AbstractApplicationContext#refresh
    |
    |
    V
AbstractApplicationContext#registerBeanPostProcessors
    |
    |
    V
AbstractBeanFactory#addBeanPostProcessor
```

其中，addBeanPostProcessor即为将扩展的BeanPostProcessor注册至AbstractBeanFactory中的**beanPostProcessors**队列中。  

#### 2、Bean的加载
Bean的加载，也可以称之为Bean的实例化。我们在调用ApplicationContext、BeanFactory的getBean时即为此阶段。  
无论是ApplicationContext的getBean，还是BeanFactory的getBean，其本质上都是调用AbstractBeanFactory中的getBean方法，所以Bean加载的主要逻辑在AbstractBeanFactory中。  
在*初始化*阶段，我们已经向AbstractBeanFactory中注册了各种BeanPostProcessor，如何解析各种BeanPostProcessor，并装配对应的Proxy，主要逻辑都在AbstractAutowireCapableBeanFactory中。  

```java
AbstractBeanFactory#getBean
    |
    |
    V
AbstractBeanFactory#doGetBean
    |
    |
    V
AbstractAutowireCapableBeanFactory#createBean
    |
    |
    V
AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation
    |
    |
    V
AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInstantiation
    |
    |
    V
AbstractAutoProxyCreator#postProcessBeforeInstantiation
```

在进入AbstractAutoProxyCreator#postProcessBeforeInstantiation之后，就开始了组装代理的操作。在组装前，势必要先将当前所加载的Bean关联的Advice匹配出来。AbstractAutoProxyCreator通过调用getAdvicesAndAdvisorsForBean，匹配出Advice，但具体说来，这里运用了模板模式，AbstractAutoProxyCreator本身并没有实现getAdvicesAndAdvisorsForBean，实际调用的是其子类AbstractAdvisorAutoProxyCreator中的实现。  

```java
接上一段：
AbstractAutoProxyCreator#postProcessBeforeInstantiation
    |
    |
    V
AbstractAutoProxyCreator#createProxy
    |
    |
    V
ProxyFactory#getProxy
    |
    |
    V
ProxyCreatorSupport#createAopProxy
    |
    |
    V
DefaultAopProxyFactory#createAopProxy
```

到了DefaultAopProxyFactory#createAopProxy这，会做一个抉择，决定具体使用JdkDynamicAopProxy构造代理还是使用Cglib2AopProxy构造代理。  
抉择的规则如下：  
> 1、在没有设置optimize = true，proxyTargetClass = true，且没有提供SpringProxy之外的其他代理时，会直接使用JdkDynamicAopProxy；  
> 2、当前代理的目标为接口时，使用JdkDynamicAopProxy；  
> 3、当前代理的目标非接口时，使用Cglib2AopProxy。  

至此，决定好代理方式之后，就会调用具体代理类的getProxy方法，来为Bean生成代理。这样，当调用目标对象时，就会通过层层构造的代理对象，逐个调用通知方法。