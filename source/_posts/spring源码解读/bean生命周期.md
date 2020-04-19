---
layout: '[layout]'
title: bean生命周期
date: 2020-04-11 14:49:06
tags:
    - spring
categories:
    - spring 源码
---
# 前言
bean是一个被实例化，组装并通过spring IOC容器管理的对象。这些bean都是由用容器提供的配置元数据构成的。

下列是每个bean定义的属性，也即是配置元信息

| 属性 | 描述 | 
| :----: |:----: |
|class | 这个属性是强制的，并且指定用来创建bean | 
| name | 这个属性指定唯一的bean标识符。在ioc容器中，可以通过该标识符找到对应的类 |
| scope | 这个属性指定由bean定义创建的对象的作用域。 |
| lazy-initialization mode | 延迟初始化，在第一次被请求时才创建对象，而不是在容器启动时就被创建|
|initialization | 在bean的所有属性都被赋值之后，ioc容器调用这个初始化方法。 |
| destroy | 在ioc被销毁时候，调用每个bean的销毁方法 |

# bean的作用域
在spring boot中可以使用@Scope注解为每一个bean自定义作用域。其中总共有六种作用域提供给使用者，来定义每个bean的作用域

| Scope | 描述 |
| :----: | :----: |
| singleton | spring ioc中只存在一个bean实例。而且是在容器创建时，就把这个bean创建在容器中。 |
| prototype | 即是多例，这个和单例正好相反，在容器创建时，并不会创建bean，而是在这个bean第一次使用的时候进行创建。而且容器也不会管理这个bean。即在容器被销毁时，也不会调用bean的destroy方法 |
| request | 容器会为每一个http请求创建一个request的bean。任何一个实例的任何状态更改对其他的实例都是不可见的。一旦请求完成，这个实例就会被销毁 |
| session | 容器会为每一个http会话创建一个新的实例，即是如果有二十个会话，就会创建二十个实例。在单个会话中，每个请求都可以访问该会话范围内相同的单个bean |
| application | 容器为每个web应用程序运行时创建一个实例，几乎是单例范围。但有两个不同的地方：1. 应用程序作用域bean是每个ServletContext的单例对象。2. 应用程序作用域bean作为ServletContext属性可见 |
| webSocket | 协议支持客户端和远程主机之间的双向通信，远程主机选择与客户端通信。WebSocket协议为两个方向的通信提供了一个单独的TCP连接。这对于具有同步编辑和多用户游戏的多用户应用程序特别有用。在这种类型的Web应用程序中，HTTP仅用于初始握手。如果服务器同意，服务器可以以HTTP状态101（交换协议）进行响应。如果握手成功，则TCP套接字保持打开状态，客户端和服务器都可以使用该套接字向彼此发送消息。|

# 生命周期
接下来就是紧张刺激的重头戏——`bean生命周期`。bean的生命周期：bean创建->初始化->销毁。

构造一个bean
- 单实例：在容器启动时，创建对象
- 多实例：每次获取时，创建一个对象

## 自定义bean的初始化和销毁方法

### 通过@Bean的方式指定初始化方法和销毁方法
在对象创建完成，并完成相关属性赋值之后，就进行初始化。容器关闭时进行销毁。
```java
    /**
    * 定义一个对象
    */
    public class Car {
    public Car(){
        System.out.println("car constructor ......");
    }
    public void init(){
        System.out.println("car init ....");
    }

    public void destroy(){
        System.out.println("car destroy.........");
    }
}

 // 创建一个bean
    @Configuration
    public class MainConfigOfLifeCycle {

        @Bean(initMethod = "init", destroyMethod = "destroy")
        public Car car(){
            return new Car();
        }
}

```

### 实现接口InitializingBean(定义初始化)和DisposableBean(定义销毁)
```java
public class Car implements InitializingBean, DisposableBean {
    public Car(){
        System.out.println("car constructor ......");
    }
    public void init(){
        System.out.println("car init ....");
    }
    

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("这个是实现InitializingBean接口中的方法。。。来进行bean的初始化。。。");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("这个是实现DisposableBean接口中的方法。。。来进行bean的销毁。。。");
    }
}
```

### BeanPostProcessor
bean的后置处理器，在bean的初始化前后完成一些处理工作。
```java
/*
 这是一个接口
*/
public interface BeanPostProcessor {
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 在初始化之前进行一些调用工作
        return bean;// 可以直接返回bean，也可以返回包装好的bean
        
    }

    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        // 在任何初始化方法调用之后，进行处理
        return bean;// 可以直接返回bean，也可以返回包装好的bean
        
    }
}
```

### spring 中BeanPostProcessor工作原理
```java
package com.brandon.bean;

import org.springframework.beans.BeansException;
import org.springframework.beans.factory.config.BeanPostProcessor;
import org.springframework.stereotype.Component;

@Component
public class MyBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        System.out.println("调用postProcessorAfterInitialization方法 ： " + beanName + " =>" + bean);
        return bean;
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 在下面一句中打上断点
        System.out.println("调用postProcessionBeforeInitialization 。。。。 " + beanName + "= > " + bean); 
        return bean;
    }
}

```

```java
// ...经过调用栈之后可以到下面一步
// 为bean中所有属性赋值，mdb就是定义的bean属性 scope等等。。
this.populateBean(beanName, mbd, instanceWrapper);
// 赋值完成之后调用初始化方法。
exposedObject = this.initializeBean(beanName, exposedObject, mbd);
```

```java
// inittializeBean方法内部，只截取了关键部分。。。
        if (mbd == null || !mbd.isSynthetic()) {
            // 在初始化方法之前调用BeanPostProcessorsBeforeInitialization，将所有的beanPostProcessorBeforeInitialization执行一遍
            wrappedBean = this.applyBeanPostProcessorsBeforeInitialization(bean, beanName);
        }

        try {
            // 调用bean的初始化方法
            this.invokeInitMethods(beanName, wrappedBean, mbd);
        } catch (Throwable var6) {
            throw new BeanCreationException(mbd != null ? mbd.getResourceDescription() : null, beanName, "Invocation of init method failed", var6);
        }

        if (mbd == null || !mbd.isSynthetic()) {
            // 在初始化完成之后，调用BeanPostProcessorsAfterInitialization， 将所有BeanPostProcessorsAfterInitialization执行一遍。
            wrappedBean = this.applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
        }
```
```java
    public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName) throws BeansException {
        Object result = existingBean;

        Object current;
        for(Iterator var4 = this.getBeanPostProcessors().iterator(); var4.hasNext(); result = current) {
            BeanPostProcessor processor = (BeanPostProcessor)var4.next();
            // 挨个执行所有的postProcessBeforeInitialization
            // 如果有null，则直接返回   
            current = processor.postProcessBeforeInitialization(result, beanName);
            if (current == null) {
                return result;
            }
        }

        return result;
    }
```

![avatar](https://brandonxcc.top/20200411.jpg)