---
layout: '[layout]'
title: Spring Ioc的启动过程
date: 2019-11-25 17:11:27
tags:
    - spring
categories:
    - spring 源码
---
## Spring Ioc的基本概念
Ioc(inversion of Control), 被通常翻译为控制反转，或者是DI(dependency Injuction)依赖注入。

如果我们需要某个对象的功能，不需要人为手动的去创建对象的实例；只需要向框架说明我需要这个对象，框架就会自动的提供这个对象的实例。也就是原来我们需要什么东西需要自己去拿，但是现在框架就会为我们送过来。

依赖注入的实现主要有三种方式：
- 构造方法注入：被注入对象可以通过在其构造方法中声名依赖对象的参数列表，让外部对象（通常是IoC容器）知道它需要哪些依赖对象。IoC Service Provider会检查被注入对象的构造方法，取得它所需要的依赖对象列表，进而为其注入相应的对象。
- setter 方法注入：当前对象只要为其所对应的属性添加setter方法，就可以通过setter方法将相应的依赖对象设置到被注入对象中。
- 接口注入

## Spring Ioc初始化的过程
spring Ioc的最基本启动过程
```java 
public static void main(String[] arg){
    ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationfile.xml");
} 
可以认为是在classPath中找到xml文件，根据xml文件配置applicationContext
```

## Ioc源码分析
```java
public void refresh() throws BeansException, IllegalStateException {
    // 对这个实例进行加锁处理，
    //避免在applicationContext还未创建完成，
    //就出现另一个线程对这个实例的创建或者销毁操作。
        synchronized(this.startupShutdownMonitor) {
            //记下启动时间，并且将状态改为正在运行。
            this.prepareRefresh();
            // 根据配置信息，解析成一个个的bean定义，并注册到beanFactory中。
            // 创建一个beanName -> beanDefinition的map，只是建立映射，并未初始化。
            ConfigurableListableBeanFactory beanFactory = this.obtainFreshBeanFactory();
            // bean的加载器，并加载几个特殊的bean
            this.prepareBeanFactory(beanFactory);

            try {
                // 如果bean继承了BeanFactoryPostProcess这个接口
                // 在初始化的时候，spring会调用postProcessBeanFactory方法。
                this.postProcessBeanFactory(beanFactory);

                this.invokeBeanFactoryPostProcessors(beanFactory);
                this.registerBeanPostProcessors(beanFactory);
                this.initMessageSource();
                this.initApplicationEventMulticaster();
                this.onRefresh();
                this.registerListeners();
                // 初始化所有的singleton bean
                this.finishBeanFactoryInitialization(beanFactory);
                // 广播bean的初始化完成。
                this.finishRefresh();
            } catch (BeansException var9) {
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Exception encountered during context initialization - cancelling refresh attempt: " + var9);
                }

                this.destroyBeans();
                this.cancelRefresh(var9);
                throw var9;
            } finally {
                this.resetCommonCaches();
            }

        }
    }
```

到这里可以明白一个事情，虽然Application继承自beanFactory，但是不应该将ApplicationContext看作beanFactory的实现类；而是应当看作是内部持有一个beanFactory的实例，所有的BeanFacotory操作是委托给context去做的。

