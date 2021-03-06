# 精尽 Dubbo 源码分析 —— 动态代理（二）之 JDK



# 1. 概述

本文接 [《精尽 Dubbo 源码分析 —— 动态代理（一）之 Javassist》](http://svip.iocoder.cn/Dubbo/proxy-javassist/?self) 一文，分享使用 **JDK** 生成动态代理的代码实现。

如果 JDK Proxy 不熟悉的胖友，可以看下 [《 Java JDK 动态代理（AOP）使用及实现原理分析》](http://blog.csdn.net/jiankunking/article/details/52143504#) 学习下。🙂 学无止境呀。

另外，如果使用 JDK 生成代理，配置方式如下：

```
// 服务引用
<dubbo:reference proxy="jdk" />

// 服务暴露
<dubbo:service proxy="jdk" />
```

# 2. JdkProxyFactory

`com.alibaba.dubbo.rpc.proxy.jdk.JdkProxyFactory` ，实现 AbstractProxyInvoker 抽象类，代码如下：

```
 1: public class JdkProxyFactory extends AbstractProxyFactory {
 2: 
 3:     @SuppressWarnings("unchecked")
 4:     public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
 5:         return (T) Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(), interfaces, new InvokerInvocationHandler(invoker));
 6:     }
 7: 
 8:     public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
 9:         return new AbstractProxyInvoker<T>(proxy, type, url) {
10:             @Override
11:             protected Object doInvoke(T proxy, String methodName,
12:                                       Class<?>[] parameterTypes,
13:                                       Object[] arguments) throws Throwable {
14:                 // 获得方法
15:                 Method method = proxy.getClass().getMethod(methodName, parameterTypes);
16:                 // 调用方法
17:                 return method.invoke(proxy, arguments);
18:             }
19:         };
20:     }
21: 
22: }
```

- ```
  #getProxy(invoker, interfaces)
  ```

   

  方法

  - 第 5 行：创建 InvokerInvocationHandler 对象，传入 `invoker` 对象。
  - 第 5 行：调用 `java.lang.reflect.Proxy#getProxy(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)` 方法，创建 Proxy 对象。
  - 🙂 相比 Javassist 精简很多，期待 JDK Proxy 的不断性能优化。

- ```
  #getInvoker(proxy, type, url)
  ```

   

  方法

  - 第 9 至 19 行：创建 AbstractProxyInvoker 对象，实现

     

    ```
    #doInvoker(...)
    ```

     

    方法。

    - 第 15 行：调用 `Class#getMethod(String name, Class<?>... parameterTypes)` 方法，反射获得方法。
    - 第 17 行：调用 `Method#invoke(proxy, arguments)` 方法，执行方法。
    - 推荐阅读：[《Java反射原理简析》](http://www.fanyilun.me/2015/10/29/Java%E5%8F%8D%E5%B0%84%E5%8E%9F%E7%90%86/)

# 666. 彩蛋

![知识星球](http://www.iocoder.cn/images/Architecture/2017_12_29/01.png)

推荐对动态代理的性能感兴趣的胖友，可阅读 [《动态代理方案性能对比》](http://javatar.iteye.com/blog/814426) 。