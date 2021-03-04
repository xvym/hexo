---
title: 理解springioc（一）：一个基本的bean容器——DefaultListableBeanFactory
date: 2021-01-28 21:58:46
tags: 
- Java
- spring
categories: article
---
### 前言
spring源码的学习始终是Java开发迈不过去的坎，springioc更是spring精华中的精华。无奈spring版本众多，资料庞杂，也缺少一本相对来说权威的源码解读书籍，多次阅读源码都是卡在半路，不得门道。遂决定从spring底层开始逐步理解springioc，记下学习所悟并梳理成文。
# BeanFactory
BeanFactory是Spring中最核心的接口。它为我们定义了从容器中获取bean的方法。如果说springioc是一间房子，那么BeanFactory就是我们进入这间房子的大门，所有Spring容器都要实现BeanFactory接口。而DefaultListableBeanFactory就是一个默认的、最常用的BeanFactory实现类，我们可以通过DefaultListableBeanFactory来一窥spring容器的奥秘。

# DefaultListableBeanFactory
DefaultListableBeanFactory位于org.springframework.beans.factory.support包下  
DefaultListableBeanFactory是一个最基本的能真正实例化bean的一个BeanFactory实现类，同时实现了BeanDefinitionRegistry和BeanFactory接口。我们可以通过了解DefaultListableBeanFactory的工作原理来理解一个bean是怎么被加载的。注意，本文首先是通过一个全局的视角来抽象地认识一个bean容器，并不会去深入研究具体的实现原理，所以下文我们都是直接从接口的定义来认识bean容器，而不会探究其实现原理。

DefaultListableBeanFactory的类关系图如下
![DefaultListableBeanFactory的类关系图](https://xvym.gitee.io/static/理解springioc/一/图1-DefaultListableBeanFactory的类关系图.png)
其中处于顶层的类关系有这么几个：
### 1. 获取bean的门面接口：实现BeanFactory接口  
BeanFacatory位于org.springframework.beans.factory包下。它是顶层的ioc容器接口，所有ioc容器的实现均需要实现该接口。它是获取bean的门面接口，容器的使用者通过该接口来获取bean或是获取bean的关键信息。
<details>
<summary>BeanFactory接口方法</summary>

```java
public interface BeanFactory {
    
    String FACTORY_BEAN_PREFIX = "&";

    Object getBean(String name) throws BeansException;

    <T> T getBean(String name, Class<T> requiredType) throws BeansException;

    Object getBean(String name, Object... args) throws BeansException;

    <T> T getBean(Class<T> requiredType) throws BeansException;

    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);

    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    boolean containsBean(String name);

    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String name, ResolvableType typeToMatch) throws NoSuchBeanDefinitionException;

    boolean isTypeMatch(String name, Class<?> typeToMatch) throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String name, boolean allowFactoryBeanInit) throws NoSuchBeanDefinitionException;

    String[] getAliases(String name);

}
```
</details>
BeanFactory的方法不难理解，定义的大多是获取bean实例或是bean的相关信息的方法。FACTORY_BEAN_PREFIX则是用来标记要获取的是FactoryBean而非FactoryBean生产的bean。由于BeanFactory的具体实现方法还涉及到Spring的其他扩展功能，在此我们先抽象的感受一下BeanFactory的方法即可。

### 2. 注册bean元信息：实现BeanDefinitionRegistry接口
BeanDefinitionRegistry位于org.springframework.beans.factory.support包下。在spring中，bean通常通过xml文件或者JavaConfig的方式进行定义，每个bean的定义信息（如bean的作用于、bean是否懒加载等）会由一个相应的BeanDefinition对象来进行存储。而BeanDefinition则会通过BeanDefinitionRegistry注册到容器中，这样容器才能根据配置信息生成一个bean。至于从配置文件转换为BeanDefiniton的工作，则是由ClassPathBeanDefinitionScanner来进行收集、BeanDefinitionReader来进行解析的。此处我们仅需要知道有这两个角色即可，不作展开。我们可以先看下BeanDefinitionRegistry定义的方法
<details>
<summary>BeanDefinitionRegistry接口方法</summary>

```java
public interface BeanDefinitionRegistry extends AliasRegistry {

    // 关键 -> 向注册表中注册一个新的BeanDefinition实例
    void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException;

    // 移除注册表中已注册的BeanDefinition实例
    void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 从注册中心取得指定的BeanDefinition实例
    BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

    // 判断BeanDefinition实例是否在注册表中（是否注册）
    boolean containsBeanDefinition(String beanName);

    // 取得注册表中所有BeanDefinition实例的beanName（标识）
    String[] getBeanDefinitionNames();

    // 返回注册表中BeanDefinition实例的数量
    int getBeanDefinitionCount();

    // beanName（标识）是否被占用
    boolean isBeanNameInUse(String beanName);
}
```
    
</details>

其中最关键的是```registerBeanDefinition(String beanName, BeanDefinition beanDefinition)```方法，容器通过这个方法将bean的元信息BeanDefinition添加到注册表中。而在DefaultListableBeanFactory中，BeanDefinition注册表是通过一个线程安全的ConcurrentHashMap来实现的。
``` java
/** Map of bean definition objects, keyed by bean name. */
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<>(256);
```
同样，BeanDefinition还有很多不同的实现类，扩展了很多注册BeanDefinition的方法，在此我们依然是先抽象地感受一下这个类的方法即可。

### 3. 单例bean的注册中心：实现SingletonBeanRegistry接口  
SingletonBeanRegistry位于org.springframework.beans.factory.config包下。它是单例Bean的注册中心。与BeanDefinitionRegistry不同，BeanDefinitionRegistry用于注册元信息（BeanDefinition），而SingletonBeanRegistry则是真正向容器中注册一个单例的bean。当我们从容器中获取单例bean时，底层就是使用SingletonBeanRegistry来获取单例bean。
<details>
<summary>SingletonBeanRegistry</summary>

```java
public interface SingletonBeanRegistry {
    // 向Bean容器中注册单例Bean
    void registerSingleton(String beanName, Object singletonObject);

    // 根据Bean的名字获取单例Bean
    @Nullable
    Object getSingleton(String beanName);

    // 根据Bean的名字判断容器中是否存在单例Bean
    boolean containsSingleton(String beanName);

    // 获取容器中所有的单例Bean的名字
    String[] getSingletonNames();

    // 获取容器中单例Bean的数量
    int getSingletonCount();

    // 返回此注册表使用的单例互斥锁
    Object getSingletonMutex();
}
```
</details>

这里有两个方法比较关键，注册单例bean的方法```void registerSingleton(String beanName, Object singletonObject)```和获取单例bean的方法```Object getSingleton(String beanName)```。Spring在启动过程中会使用三个map（亦被称之为三级缓存）来进行bean的存储。
```java
/** Cache of singleton objects: bean name to bean instance. */
// 一级缓存，以bean的名字为key，bean实例为value
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name to ObjectFactory. */
// 三级缓存，以bean的名字为key，创建bean的工厂为value
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name to bean instance. */
// 二级缓存，以bean的名字为key，早期bean实例为value（早期bean是未构建完全的bean，实际上是不可用的，只是用来解决循环依赖的问题）
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);
```
这里我们可以稍微发散一下，先来看一下SingletonBeanRegistry最常用的实现类DefaultSingletonBeanRegistry是如何实现这两个方法的。
    
<details>
<summary>3.1 registerSingleton</summary>

```java
@Override
public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
    Assert.notNull(beanName, "Bean name must not be null");
    // 利用单例注册表的方式来保证bean是单例注册的。
    Assert.notNull(singletonObject, "Singleton object must not be null");
    synchronized (this.singletonObjects) {
        Object oldObject = this.singletonObjects.get(beanName);
        if (oldObject != null) {
            throw new IllegalStateException("Could not register object [" + singletonObject +
                    "] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
        }
        addSingleton(beanName, singletonObject);
    }
}

protected void addSingleton(String beanName, Object singletonObject) {
    synchronized (this.singletonObjects) {
        // 无论二、三级缓存是否存在bean，都会将其清空，并升级到一级缓存中，同时beanName添加到已注册列表中
        this.singletonObjects.put(beanName, singletonObject);
        this.singletonFactories.remove(beanName);
        this.earlySingletonObjects.remove(beanName);
        this.registeredSingletons.add(beanName);
    }
}
```
</details>

<details>   
<summary>3.2 getSingleton</summary>

```java
@Nullable
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // Quick check for existing instance without full singleton lock
    // 尝试从一级缓存中获取单例bean
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 如果一级缓存中不存在bean，且bean的状态为创建中，则从二级缓存中获取早期的单例bean
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            // 如果二级缓存中也不存在bean，且spring允许循环依赖（allowEarlyReference，默认为true），则会开始通过单例注册表的方式来进行单例bean的创建
            synchronized (this.singletonObjects) {
                // Consistent creation of early reference within full singleton lock
                // 检查一级缓存
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    // 检查二级缓存
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // 检查三级缓存
                        ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                        // 如果三级缓存不为空，则从对应的单例bean工厂中创建早期bean实例，并将其放入二级缓存中，同时，将三级缓存中的bean工厂删除。
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```
</details>
当然，我们一般不会直接调用这两个接口来进行单例bean的注册和获取，但是在spring中单例bean的注册和获取（BeanFactory的getBean方法）最终还是会回到这两个方法上，所以我们对这两个方法的学习还是有必要的。

### 4. 其他：bean的别名管理类：继承SimpleAliasRegistry -> 实现AliasRegistry接口；支持序列化：实现Serializable序列化接口

第四4相对来说没有那么重要，此处先跳过。如果一个容器实现了BeanDefinitionRegistry、BeanFactory、SingletonBeanRegistry，那么就已经具备了一个容器的必要的功能了。我们是可以直接使用DefaultListableBeanFactory来实现一个bean的注册和获取的。

# bean容器实战
定义一个实体类
```java
public class MyBean {

    private final Logger logger = LoggerFactory.getLogger(MyBeanA.class);

    public void test() {
        logger.info("myBean is testing!");
    }
}
```

编写一个启动类
```java
@ComponentScan
public class BootStrap {
    public static void main(String[] args) {
        // 此处我们先新建一个高级容器AnnotationConfigApplicationContext，免去很多初始化过程，我们只需知道这是一个支持注解的高级容器即可
        AnnotationConfigApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(BootStrap.class);
        // 获取基本的DefaultListableBeanFactory
        DefaultListableBeanFactory beanFactory = (DefaultListableBeanFactory) applicationContext.getBeanFactory();
        // 创建一个基本的BeanDefinition——GenericBeanDefinition，并将MyBean转换为BeanDefinition
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClass(MyBean.class);
        // 在容器中注册BeanDefiniton
        beanFactory.registerBeanDefinition("myBean", beanDefinition);
        // 获取并调用bean的方法
        MyBean myBean = (MyBean) beanFactory.getBean("myBean");
        myBean.test();
    }
}
```
启动main方法，顺利地执行了myBean的test方法
```
23:17:33.469 [main] DEBUG org.springframework.context.annotation.AnnotationConfigApplicationContext - Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@48503868
23:17:33.491 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalConfigurationAnnotationProcessor'
23:17:33.565 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerProcessor'
23:17:33.568 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.event.internalEventListenerFactory'
23:17:33.570 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalAutowiredAnnotationProcessor'
23:17:33.572 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'org.springframework.context.annotation.internalCommonAnnotationProcessor'
23:17:33.582 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'bootStrap'
23:17:33.609 [main] DEBUG org.springframework.beans.factory.support.DefaultListableBeanFactory - Creating shared instance of singleton bean 'myBean'
23:17:33.610 [main] INFO cn.com.xvym.learning.springbootlearning.MyBean - myBean is testing!
```
这就是一个最基本的bean的注册和获取过程。当然，我们常用的bean注册方式肯定不会是这么繁琐的，spring通过很多另外的接口扩展功能为我们实现了更为方便的bean注册和获取方式和很多bean的增强功能，但是底层最终还是会调用到这几个基本的方法，这些我们可以在后面再持续进行探讨。

