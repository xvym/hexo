---
title: 2021-01-30-理解springioc（二）：认识spring的高级容器
date: 2021-01-30 23:03:55
tags:
- Java
- spring
categories: article
---
### 前言
在[《理解springioc（一）：一个基本的bean容器——DefaultListableBeanFactory》](https://xvym.com.cn/2021/01/28/2021-01-28-%E7%90%86%E8%A7%A3springioc%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E4%B8%80%E4%B8%AA%E5%9F%BA%E6%9C%AC%E7%9A%84bean%E5%AE%B9%E5%99%A8%E2%80%94%E2%80%94DefaultListableBeanFactory/#more)一文中，我们初步认识了一个关键的基础容器——DefaultListableBeanFactory，而spring中的高级容器往往都会使用DefaultListableBeanFactory来作为BeanFactory。在本文中，我们就来更逐步地探讨一下spring中的高级容器。
<!--more-->
## ApplicationContext
![图1-ApplicationContext的类图](https://xvym.gitee.io/static/理解springioc/二/图1-ApplicationContext的类图.png)
ApplicationContext是BeanFactory的一个扩展，该接口除了拥有BeanFactory的所有功能外，还具有以下扩展：
1. 继承MessageSource，用于支撑国际化功能
2. 继承ApplicationEventPublisher，提供Spring的事件机制
3. 继承ResourceLoader，用于加载资源文件 
4. 继承EnvironmentCapable，用于获取Environment对象进而支持多环境配置

在ApplicationContext中新定义了以下的方法
<details>
<summary>ApplicationContext接口方法</summary>

```java
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory,
		MessageSource, ApplicationEventPublisher, ResourcePatternResolver {

    // 获取ApplicationContext的id
	@Nullable
	String getId();

    // 获取ApplicationContext的名字
	String getApplicationName();

    // 获取一个便于查看的ApplicationContext的名字
	String getDisplayName();

    // 获取ApplicationContext的启动日期
	long getStartupDate();

    // 获取ApplicationContext的父容器
	@Nullable
	ApplicationContext getParent();

    // 将AutowireCapableBeanFactory暴露给外部使用，一般比较少使用，此处先略过
	AutowireCapableBeanFactory getAutowireCapableBeanFactory() throws IllegalStateException;

}
```
</details>

可以看到，这些方法只定义了对ApplicationContext一些信息获取的方法，其功能扩展是通过集成更多的接口来达成的。

比较ApplicationContext和DefaultListableBeanFactory的类图，我们可以看到ApplicationContext并没有去实现SingletonBeanRegistry和BeanDefinitionRegistry等接口，这与我们在上一篇文章中认为的容器应该实现这些接口有出入。我们可以带着这些疑问继续往下看。
![图2-DefaultListableBeanFactory的类关系图](https://xvym.gitee.io/static/理解springioc/一/图1-DefaultListableBeanFactory的类关系图.png)

## ConfigurableApplicationContext、WebApplicationContext和ConfigurableWebApplicationContext
ApplicationContext接口有两个直接子接口，ConfigurableApplicationContext和WebApplicationContext，在此之上，还有一个继承了这两个子接口的ConfigurableWebApplicationContext。我们可以通过ConfigurableWebApplicationContext的类图来了解这些接口的联系
![图3-ConfigurableWebApplicationContext的类图](https://xvym.gitee.io/static/理解springioc/二/图3-ConfigurableWebApplicationContext的类图.png)
我们可以先看下两个子接口的代码
### 1. WebApplicationContext接口
WebApplicationContext的定义比较简单，主要是用于支持web应用
<details>
<summary>WebApplicationContext接口方法</summary>

```java
public interface WebApplicationContext extends ApplicationContext {

    // 定义了web应用的根路径，这将允许spring从web应用根路径中读取配置文件
	String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";

    // 以下是对一些web应用的概念进行名称定义
	String SCOPE_REQUEST = "request";

	String SCOPE_SESSION = "session";

	String SCOPE_APPLICATION = "application";

    // 定义特殊的bean名称
	String SERVLET_CONTEXT_BEAN_NAME = "servletContext";

	String CONTEXT_PARAMETERS_BEAN_NAME = "contextParameters";

	String CONTEXT_ATTRIBUTES_BEAN_NAME = "contextAttributes";

    // WebApplication在web应用启动时会通过ContextLoaderListener被绑定到更上层的容器ServletContext，此处就是取出上层的ServletContext，这点我们暂时先跳过。
	@Nullable
	ServletContext getServletContext();

}
```
</details>

### 2. ConfigurableApplicationContext接口  
在spring的体系中，ConfigurableApplicationContext应该是一个相当重要的接口。它为我们定义了高级容器在初始化过程中需要执行的各种操作，并定义了很多功能扩展点。在这个接口中我们将能看到很多熟悉的高级容器的特性。相较于ApplicationContext，ConfigurableApplicationContext又另外继承了两个接口：
2.2. 继承Lifecycle接口，用于管理容器的生命周期
2.3. 继承Closeable，用于关闭资源。
    
此外，该接口在ApplicationContext的基础上增加了很多配置容器的方法，并将配置容器、控制容器生命周期的方法进行了封装，避免直接调用底层的EnvironmentCapable接口和Lifecycle接口。
<details>
<summary>ConfigurableApplicationContext接口方法</summary>

```java
public interface ConfigurableApplicationContext extends ApplicationContext, Lifecycle, Closeable {

    // spring规定的用于分割配置文件路径的符号（ConfigurableApplicationContext支持读取多个配置文件）
    String CONFIG_LOCATION_DELIMITERS = ",; \t\n";

    // 下方几个定义的XX_NAME静态变量是Spring中规定的特殊bean的名字，spring只会利用这些名字去取相应的bean
    String CONVERSION_SERVICE_BEAN_NAME = "conversionService";

    String LOAD_TIME_WEAVER_BEAN_NAME = "loadTimeWeaver";

    String ENVIRONMENT_BEAN_NAME = "environment";

    String SYSTEM_PROPERTIES_BEAN_NAME = "systemProperties";

    String SYSTEM_ENVIRONMENT_BEAN_NAME = "systemEnvironment";

    String SHUTDOWN_HOOK_THREAD_NAME = "SpringContextShutdownHook";
    
    // 设置容器id
    void setId(String id);

    // 设置父容器id
    void setParent(@Nullable ApplicationContext parent);

    // 设置环境对象
    void setEnvironment(ConfigurableEnvironment environment);
    
    // 重写继承自EnvironmentCapable的getEnvironment方法，返回可配置的环境对象
    @Override
    ConfigurableEnvironment getEnvironment();

    // 向容器中添加BeanFactoryPostProcessor，会在读取容器配置的时候调用，增加的Processor会在容器refresh的时候使用
    void addBeanFactoryPostProcessor(BeanFactoryPostProcessor postProcessor);

    // 向容器增加一个ApplicationListener，如果容器还没有启动，那么在此增加的监听器将会在refresh中全部被调用，如果容器已经是active状态，则会通过multicaster中通过广播的方式进行调用
    void addApplicationListener(ApplicationListener<?> listener);

    // 设置类加载器，这个类加载器会传递到内部bean工厂
    void setClassLoader(ClassLoader classLoader);

    // 向容器中增加ProtocolResolver，用于解析协议
    void addProtocolResolver(ProtocolResolver resolver);

    // 高级容器的重中之重，加载资源配置文件，刷新容器并初始化所有的bean，在refresh方法执行完毕后，要么所有的bean都完成了初始化，要么就一个都没有完成初始化
    void refresh() throws BeansException, IllegalStateException;

    // 向JVM注册一个回调函数，用以在JVM关闭时销毁容器
    void registerShutdownHook();

    // 关闭容器，释放所有的资源和锁并销毁所有缓存的singletonBean。spring的注释中提到，不要去调用其父类容器的close方法，父类容器有其自己的生命周期。
    @Override
    void close();

    // 检测该容器是否被启动过
    boolean isActive();

    // 返回此容器的BeanFactory，注意不要通过该方法来对容器中的bean进行处理，因为单例bean在此之前已经生成
    ConfigurableListableBeanFactory getBeanFactory() throws IllegalStateException;
}
```
</details>  

<details>
<summary>WebApplicationContext接口方法</summary>

```java
public interface WebApplicationContext extends ApplicationContext {

    String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";

    String SCOPE_REQUEST = "request";

    String SCOPE_SESSION = "session";

    String SCOPE_APPLICATION = "application";

    String SERVLET_CONTEXT_BEAN_NAME = "servletContext";

    String CONTEXT_PARAMETERS_BEAN_NAME = "contextParameters";

    String CONTEXT_ATTRIBUTES_BEAN_NAME = "contextAttributes";

    @Nullable
    ServletContext getServletContext();

}
```
</details>

在ConfigurableApplicationContext中，我们终于第一次比较完整地看到了spring高级容器的全貌。
我们可以看一下ConfigurableApplicationContext的实现类代码，来真正地认识一下容器的工作原理。一般来说现在大家都比较推崇使用JavaConfig的注解方式来进行配置，支持JavaConfig的实现类实现的是更为通用的GenericApplicationContext。

在看GenericApplicationContext之前，我们不妨先来回顾一下我们之前一直带着的问题，**为什么bean在BeanFactory中是懒加载，而在ApplicationContext中是在容器初始化的时候就完成了加载？ApplicationContext为什么没有实现没有去实现BeanDefinitionRegistry和SingletonBeanRegistry接口？**


## 理解BeanFactory和ApplicationContext的语义区别
我们可以先看下BeanFactory是怎么完成bean的加载，而ApplicationContext又是怎么完成加载的。

```java
public class DefaultListableBeanFactoryTest {

    public static void main(String[] args) {
        // 使用资源加载器加载配置文件
        ClassPathResource classPathResource = new ClassPathResource("spring-bean.xml");
        // 新建工厂类
        DefaultListableBeanFactory defaultListableBeanFactory = new DefaultListableBeanFactory();
        // 通过BeanDefinition解析配置文件
        XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(defaultListableBeanFactory);
        reader.loadBeanDefinitions(classPathResource);
        // 获取bean
        MyBean myBean = (MyBean) defaultListableBeanFactory.getBean("myBean");
        System.out.println(myBean.getMessage());
    }
}
```

```java
public class ApplicationContextTest {

    public static void main(String[] args) {
        // 新建容器，ClassPathXmlApplicationContext是一个基本的ApplicationContext实现类
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext("spring-bean.xml");
        // 获取bean
        MyBean myBean = (MyBean) applicationContext.getBean("myBean");
        System.out.println(myBean.getMessage());
    }
}
```

可以看到，ApplicationContext不仅没有实现SingletonBeanRegistry和BeanDefinitionRegistry，加载容器的过程中相较起来也缺少了资源加载器、BeanDefinitionReader之类角色的显式创建。这是因为，ApplicationContext将其余的步骤都封装了起来。同时在ApplicationContext的实现类中，基本上都会包含一个DefaultListableBeanFactory，这个才是实际意义上的beanFactory，也就是存储bean的基本容器。而上面提到的所有缺失的接口实现，都会由这个DefaultListableBeanFactory来进行。所以，虽然ApplicationContext和DefaultListableBeanFactory都实现了BeanFactory，但它们所担任的角色是不一样的。在spring体系中，后缀context的语义是上下文，它不仅是一个bean容器，更包含着一个应用的所有资源和信息，这其中也包含着beanFactory；而后缀beanFactory的语义，才是真正的bean容器直接门面。这也是为什么BeanFactory加载bean是懒加载，而ApplicationContext是启动阶段就完成了所有bean的加载，因为BeanFactory不需要管理应用的所有资源，而ApplicationContext需要。