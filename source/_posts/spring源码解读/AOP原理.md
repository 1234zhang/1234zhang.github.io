---
layout: '[layout]'
title: AOP原理
date: 2020-04-12 13:59:25
tags:
    - spring
categories:
    - spring 源码
---

# AOP的目的
AOP能够将那些与业务无关，却为业务逻辑共同调用的逻辑或者责任(例如：事务处理，日志管理，权限管理等等)封装起来，便于减少系统的重复代码，降低系统模块的耦合度，并且有利于未来的可拓展性和可维护性。


# AOP的使用
1. 导入aop包
    
    ```
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aspects</artifactId>
            <version>5.2.5.RELEASE</version>
        </dependency>

    ```
2. 定义一个业务逻辑实现类；在业务逻辑进行运行的时候，将日志进行打印

    ```java
    // 定义的一个业务逻辑，计算两数之和。
        package com.brandon.aop;

        public class MathMulity {
            public int add(int i, int j){
                return i + j;
            }
        }
    ```
3. 定义一个日志切面类，在运行求和运算的时候，在必要的时候进行日志的打印
4. 给切面方法加上注解(@Aspect)表示这是一个切面方法

    ```java
    package com.brandon.aop;

    import org.aspectj.lang.annotation.*;

    @Aspect
    public class LogAspects {

        @Pointcut(value = "execution(public int com.brandon.aop.MathMulity.*(..))")
        public void pointCut(){

        }
        @Before("pointCut()") // 目标方法之前运行
        public void logStart(){
            System.out.println("计算开始");
        }
        @After("pointCut()") // 目标方法之后运行
        public void end(){
            System.out.println("计算结束");
        }
        @AfterReturning("pointCut()")// 目标方法正常返回之后运行
        public void logReturning(){
            System.out.println("returning : {}");
        }

        @AfterThrowing("pointCut()")//目标方法出现异常之后运行
        public void exception(){
            System.out.println("计算异常。。。。");
        }
         /* 环绕通知方法可以包含上面四种通知方法，环绕通知的功能最全面。
         环绕通知需要携带 ProceedingJoinPoint 类型的参数，且环绕通知必须有返回值, 返回值即为目标方法的返回值。
         */
        /*@Around("pointCut()")
        public Object around(ProceedingJoinPoint pjp){
            System.out.println("around 方法运行");
            return new Object();
        }*/
    }
    ```
5. 将所有的组件注册到容器中，并且给配置类加上@EnableAspectJAutoProxy 使得aop开启

    ```java
    @EnableAspectJAutoProxy
    @Configuration
    public class MainConfigOfAOP {

        @Bean
        public MathMulity mulity(){
            return new MathMulity();
        }

        @Bean
        public LogAspects logAspects(){
            return new LogAspects();
        }
    }
    ```

# AOP的原理
## @EnableAspectJAutoProxy注解
```java
    // 注解的实现
    @Target({ElementType.TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @Documented
    @Import({AspectJAutoProxyRegistrar.class})
    public @interface EnableAspectJAutoProxy {
        boolean proxyTargetClass() default false;

        boolean exposeProxy() default false;
    }
/*
使用@Import(),给容器导入了AspectJAutoProxyRegistrar这个类。
并使用AspectJAutoProxyRegistrar自定义的向容器注册bean
*/
```
### 使用AspectJAutoProxyRegistar给容器中注册一个什么bean?
看到这里我心中有一个疑问，要给容器中注册一个什么bean呢?, 为了解答这个疑问，debug走起！
1. 打断点

    ```java
        //我把断点打到下面的位置，
         public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
             // 断点打到这里，这里是注册的地方
            AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry); 
            AnnotationAttributes enableAspectJAutoProxy = AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
            if (enableAspectJAutoProxy != null) {
                if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
                    AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                }

                if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
                    AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
                }
            }
        }
    ```
2. 运行

    接下来的步骤主要是来创建一个AnnotationAwareAspectJAutoProxyCreator(注解模式下的AspectJ自动代理创建器)

    ```java
        @Nullable
    public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
        return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, (Object)null);
    }

    @Nullable
    public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, @Nullable Object source) {
        return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
    }
    // 下面进入registerOrEscalateApcAsRequired方法，然后判断registry中是否有AnnotationAwareAspectJAutoProxyCreator，
    // 如果没有就进行创建
    if (registry.containsBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator")){...}
    else{
        RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
            beanDefinition.setSource(source);
            beanDefinition.getPropertyValues().add("order", -2147483648);
            beanDefinition.setRole(2);
            registry.registerBeanDefinition("org.springframework.aop.config.internalAutoProxyCreator", beanDefinition);
            //创建完成之后，返回这个bean定义。
            return beanDefinition;
    }
    ```
### AnnotationAwareAspectJAutoProxyCreator 有什么用?
上一步我们知道了要创建一个什么bean，那么现在问题又来了，这个bean有什么用呢。在aop中又扮演着一个什么样的角色呢。
下面是AnnotationAwareAspectJAutoProxyCreator的继承关系图
![avatar](https://brandonxcc.top/AnnotationAwareAspectJAutoProxyCreator继承关系.png)
根据继承关系图我们可以看出，AnnotationAwareAspectJAutoProxyCreator继承了一些后置处理器，以及自动装配了beanFactory。下面我们debug分析这个的功能。

1. 我们debug的断点应该打在哪里

    在继承关系中，AnnotationAwareAspectJAutoProxyCreator继承了一个AbstractAutoProxyCreator类，这个类还实现了SmartInstantiationAwareBeanPostProcessor。说明这是一个后置处理器。我们先从这个类看，将后置处理器相关的方法(postProcessAfterInitialization、postProcessBeforeInitialization)打上断点，看在初始化前后会发生什么事情。而且这个类还实现了BeanFactoryAware，说明有setBeanFactory()这个是实现BeanFactoryAware接口的。

    接着进入AbstractAdvisorAutoProxyCreator类中，发现这个类重写了setBeanFactory()方法，并且在方法内部调用了initBeanFactory()方法。所以在这个方法上打上断点

    接着再进入AnnotationAwareAspectJAutoProxyCreator类。可以看到这个类又重写了initBeanFactory()这个方法。所以我们也在这里打上断点。
2. AnnotationAwareAspectJAutoProxyCreator注册
这里本质就是bean的创建 // TODO bean创建的博客总结预计完成在4月18日
这里特殊的就是AnnotationAwareAspectJAutoProxyCreator的beanName是 org.springframework.aop.config.internalAutoProxyCreator

3. AnnotationAwareAspectJAutoProxyCreator的执行过程
容器中会被注册一个AnnotationAwareAspectJAutoProxyCreator组件，这个组件由于后置处理器不同，会在每个bean创建的时候进行下面一个后置处理器
    ```java
    // 可以把断点打在这里，可以观察到AnnotationAwareAspectJAutoProxyCreator的作用。
    // 用来创建代理和切面。
    public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
            Object cacheKey = getCacheKey(beanClass, beanName);

            if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
                if (this.advisedBeans.containsKey(cacheKey)) {
                    return null;
                }
                if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
                    this.advisedBeans.put(cacheKey, Boolean.FALSE);
                    return null;
                }
            }
    ```
    每一个bean创建之前，调用postProcessBeforeInstantiation()；
    - 关心MathCalculator和LogAspect的创建
        - 1）、判断当前bean是否在advisedBeans中（保存了所有需要增强bean）
        - 2）、判断当前bean是否是基础类型的Advice、Pointcut、Advisor、AopInfrastructureBean，或者是否是切面（@Aspect）
        - 3）、是否需要跳过
            - 1）、获取候选的增强器（切面里面的通知方法）【List<Advisor> candidateAdvisors】每一个封装的通知方法的增强器是 InstantiationModelAwarePointcutAdvisor；判断每一个增强器是否是 AspectJPointcutAdvisor 类型的；返回true
            - 2）、永远返回false

4. 创建AOP代理
    ```java
    postProcessAfterInitialization；
    return wrapIfNecessary(bean, beanName, cacheKey);//如果需要包装的情况下
    ```
    1. 获取当前bean的所有增强器（通知方法）  Object[]  specificInterceptors
        1. 找到候选的所有的增强器（找哪些通知方法是需要切入当前bean方法的）
        2. 获取到能在bean使用的增强器。
        3. 给增强器排序
    2. 保存当前bean在advisedBeans中
    3. 如果当前bean需要增强，创建当前bean的代理对象；
        1. 获取所有增强器（通知方法）
        2. 保存到proxyFactory
        3. 创建代理对象：Spring自动决定
            - JdkDynamicAopProxy(config);jdk动态代理；
            - ObjenesisCglibAopProxy(config);cglib的动态代理；
    4. 给容器中返回当前组件使用cglib增强了的代理对象
    5. 以后容器中获取到的就是这个组件的代理对象，执行目标方法的时候，代理对象就会执行通知方法的流程；

到此aop创建的流程大体上走完了，可能会比较乱，打算在有空的时候，再来好好整理一下。下一次复习的时候，会补上一个流程图。
![avatar](https://brandonxcc.top/20200412.jpg)