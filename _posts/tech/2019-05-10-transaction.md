---
layout: post
title: 解决spring事务不生效问题
category: 技术
tags: spring
description: AOP相关的影响
---

aop概念请参考

> 【[http://www.iteye.com/topic/1122401](https://link.jianshu.com?t=http%3A%2F%2Fwww.iteye.com%2Ftopic%2F1122401)】【[http://jinnianshilongnian.iteye.com/blog/1418596](https://link.jianshu.com?t=http%3A%2F%2Fjinnianshilongnian.iteye.com%2Fblog%2F1418596)】

spring的事务管理，请参考

> 【[[http://jinnianshilongnian.iteye.com/blog/1441271\]](https://link.jianshu.com?t=http%3A%2F%2Fjinnianshilongnian.iteye.com%2Fblog%2F1441271%255D)



![img](https://upload-images.jianshu.io/upload_images/4031250-e930651ec2ed085b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/798/format/webp.png)

image.png

### 注意看数字编号

也就是说我们首先调用的是AOP代理对象而不是目标对象，首先执行事务切面，事务切面内部通过TransactionInterceptor环绕增强进行事务的增强，即进入目标方法之前开启事务，退出目标方法时提交/回滚事务。

### 简单的一个例子

```
public interface AService {  
    public void a();  
    public void b();  
}  
   
@Service()  
public class AServiceImpl1 implements AService{  
    @Transactional(propagation = Propagation.REQUIRED)  
    public void a() {  
        this.b();  
    }  
    @Transactional(propagation = Propagation.REQUIRES_NEW)  
    public void b() {  
    }  
}  
```

此处的this指向目标对象，因此调用this.b()将不会执行b事务切面，即不会执行事务增强，因此b方法的事务定义`@Transactional(propagation = Propagation.REQUIRES_NEW)`将不会实施.

### 解决方案

此处a方法中调用b方法时，只要通过AOP代理调用b方法即可走事务切面，即可以进行事务增强。

```
public void a() {  
aopProxy.b();//即调用AOP代理对象的b方法即可执行事务切面进行事务增强  
}  
```

判断一个Bean是否是AOP代理对象可以使用如下三种方法：

AopUtils.isAopProxy(bean)        ： 是否是代理对象；

AopUtils.isCglibProxy(bean)       ： 是否是CGLIB方式的代理对象；

AopUtils.isJdkDynamicProxy(bean) ： 是否是JDK动态代理方式的代理对象；

#### 方法一：通过ThreadLocal暴露Aop代理对象

1、开启暴露Aop代理到ThreadLocal支持（如下配置方式从spring3开始支持）

```
 <aop:aspectj-autoproxy expose-proxy="true"/><!—注解风格支持-->  

 <aop:config expose-proxy="true"><!—xml风格支持-->   
```

2、修改我们的业务实现类

```
this.b();
-----------修改为--------->
((AService) AopContext.currentProxy()).b();
```

原理分析



![img](https:////upload-images.jianshu.io/upload_images/4031250-c278c344f267fcba.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/886/format/webp)

image.png

> 1、在进入代理对象之后通过AopContext.serCurrentProxy(proxy)暴露当前代理对象到ThreadLocal，并保存上次ThreadLocal绑定的代理对象为oldProxy；
>  2、接下来我们可以通过 AopContext.currentProxy() 获取当前代理对象；
>  3、在退出代理对象之前要重新将ThreadLocal绑定的代理对象设置为上一次的代理对象，即AopContext.serCurrentProxy(oldProxy)。

有些人不喜欢这种方式，说通过ThreadLocal暴露有性能问题，其实这个不需要考虑，因为事务相关的（Session和Connection）内部也是通过SessionHolder和ConnectionHolder暴露到ThreadLocal实现的。

不过自我调用这种场景确实只有很少情况遇到，因此不用这种方式我们也可以通过如下方式实现。

