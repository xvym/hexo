---
title: springboot启动流程源码解析
date: 2021-02-10 11:46:16
tags:
---
好好啃一下这块硬骨头
<!--more-->
## 前言
虽然目前springboot目前已经迭代待2.5.x版本了，但鉴于springboot2.4.x版本后新增了springcloud的相关内容且相较于以前的版本代码风格有了比较大的变动，本文使用的springboot版本为2.3.9、JDK8版本。springboot的奥秘集中在三点：SpringApplication对象的创建，静态方法SpringApplication.run的执行以及一些关键注解的作用，所以本文会围绕这三点来进行源码分析。

## 一.构造SpringApplication对象
main方法调用SpringApplication.run静态方法，并传入main方法所在的启动引导类SpringbootLearningApplicationApplication.class
![图1-启动入口](https://xvym.gitee.io/static/springboot/图1-启动入口.png)
新建SpringApplication对象，并将引导类传入其构造方法
![图2-新建SpringApplication对象](https://xvym.gitee.io/static/springboot/图2-新建SpringApplication对象.png)
然后进入第一个关键步骤，**SpringApplication对象的创建**。我们先来看下SpringApplication的构造方法。
![图3-springApplication构造方法](https://xvym.gitee.io/static/springboot/图3-springApplication构造方法.png)
构造方法有两个参数，resourceLoader和primarySources，resourceLoader一般会置空，然后会在后续步骤中加载DefaultResourceLoader。primarySources是引导类，用于作为整个项目资源路径的引导标识，目前暂时还没见过有传多个引导类的情况。然后开始进行spring应用的资源初始化。
1. 首先是应用类别判断。WebApplicationType.deduceFromClasspath()方法通过ClassUtils.isPresent方法判断当前ClassLoader中是否存在相应的类来决定应该初始化哪种类型的应用，如果我们引入了springboot-starter-web模块，则web应用都类型是SERVLET，本文也是基于这个类型来进行分析。
2. 其次是各种资源的加载。在这里我们要重点关注**getSpringFactoriesInstances**方法，后面的ApplicationContextInitializer和ApplicationListener都是通过该方法进行加载的。initializers和listeners都是在此处进行赋值，便于后续spring容器刷新时对相应的对象做处理（回调）。getSpringFactoriesInstances方法会调起SpringFactoriesLoader.loadFactoryNames()方法，最终调用loadSpringFactories()方法，这个方法是Spring提供的[SPI机制](https://www.zhihu.com/search?type=content&q=SPI%20Java)的实现。SpringFactoriesLoader类会在第一次调用loadSpringFactories方法时扫描当前ClassLoader所处理的路径下所有META-INF/spring.factories文件的配置信息，来找到需要加载到Spring容器中的类，并缓存起来。这里的ClassLoader是ApplicationClassLoader，会加载用户类路径上的类库，具体可以参考这篇文章：《[JVM类加载机制]()》。
![图4-loadSpringFactories](https://xvym.gitee.io/static/springboot/图4-loadSpringFactories.png)
简单来说，就是通过查找用户路径类库中的META-INF/spring.factories文件，来找到需要加载到Spring容器中的类，如果我们需要在这些扩展点进行扩展，只需要将自定义的类按照Spring的SPI规范进行编写，便可以将类在启动阶段载入到SpringApplication对象中。springboot在启动流程的扩展机制，基本上都是使用这种方式进行的。
![图5-initializers、listeners](https://xvym.gitee.io/static/springboot/图5-initializers、listeners.png)
这里我们要搞清楚一个概念，和我们平时理解的工厂模式不同，虽然这个方法名叫做getSpringFactoriesInstances，但最终拿到的东西并不是一系列工厂实例集合，不是很清楚为什么要这样子命名。
3. 最后的deduceMainApplicationClass()是用来记录引导类的信息的，暂时没有什么很大作用。

至此，一个SpringApplication对象就构建完成了。

## 二、SpringApplication.run
SpringApplication对象构建完后，接下来就该让它跑起来了。在run方法中，有一些步骤我们没有必要去了解，这里我们只去关心有意义的步骤（比如设置显示器配置这种步骤我们直接略过就好）。

首先是进行监听器的初始化。这里的监听器和上面提到的监听器不同，实现的是SpringApplicationRunListener接口。我们可以深入以下两行代码看下其中的逻辑。
```Java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting(bootstrapContext, this.mainApplicationClass);
```
getRunListeners方法还是Spring提供的一个SPI扩展点，在底层还是去读取META-INF/spring.factories来读取SpringApplicationRunListener，与其他的扩展机制并无二致，只不过在外部进行了一层封装，将所有的SpringApplicationRunListener封装为一个SpringApplicationRunListeners对象。
这里一般也不会有太多扩展，可以看到这里只加载了一个EventPublishRunListener，用于进行Spring应用启动的时间发布。
![图6-EventPublishingRunListener-1](https://xvym.gitee.io/static/springboot/图6-EventPublishingRunListener-1.png)
然后是下面这个starting方法。这里必须得吐槽一下，在2.4.x版本，springboot有很多代码在2.4.0版本后这个方法完全用lamda表达式重写了一遍，不得不说可读性是真的一般。这些改动主要是为了增加SpringCloud的相关支持，和一般的springboot应用启动流程起始并不是很相关。最开始我看的是2.4.2版本的代码，属实难顶，这也是为什么在前言中建议只想学习springboot启动流程的话可以去看2.3.x的代码。
|2.3.x|2.4.x|
|  ----  |  ----  |
|![图7-2.3.x](https://xvym.gitee.io/static/springboot/图7-2.3.x.bmp)|![图8-2.4.x](https://xvym.gitee.io/static/springboot/图8-2.4.x.bmp)|

starting方法很简单，就是将所有listener遍历并进行事件发布。由于一般SpringApplicationRunListener的实现类只有EventPublishingRunListener，所以我们可以进一步查看这个类的代码。
![图9-EventPublishingRunListener-2](https://xvym.gitee.io/static/springboot/EventPublishingRunListener.bmp)
EventPublishingRunListener会构造一个SimpleApplicationEventMulticaster事件广播器，并将SpringApplication对象在构造方法中加载的Listener缓存在广播器中，并在starting方法会广播一个ApplicationStartingEvent，剩下的就是将事件广播给在onApplication方法中监听了这个事件的监听器了。详细步骤可以去阅读SimpleApplicationEventMulticaster的代码。所以如果我们要对spring启动监听器进行扩展，只需要按上面提到的SpringSPI方式将监听器注册到SpringApplication对象中（注意，不是容器哦，Spring容器到目前为止还没有进行创建），就可以对这些扩展点时间进行监听并执行相应的操作了。

接下来是进行配置环境的加载，就是根据一定的优先级去进行各个配置源的信息读取。各个步骤底层会涉及非常多的读取解析逻辑，但是这些步骤Springboot除了扩展点外和Spring并没有太大区别，我们就不深入到底层代码中去探索了，可以简略的看下关键步骤做的事情。
Spring应用的环境配置由ConfigurableEnvironment对象负责处理，配置的步骤在SpringApplication#prepareEnvironment方法中
``` java
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
		ApplicationArguments applicationArguments) {
    // Create and configure the environment
    // 创建ConfigurableEnvironment实例
    ConfigurableEnvironment environment = getOrCreateEnvironment();
    // 将启动参数绑定到ConfigurableEnvironment中
    configureEnvironment(environment, applicationArguments.getSourceArgs());
    ConfigurationPropertySources.attach(environment);
    // springboot通过广播ApplicationEnvironmentPreparedEvent事件设置了一个扩展点
    // 如果我们需要在环境准备阶段进行扩展，比如以微服务方式通过配置中心读取配置等，可以在这一步添加监听器来进行扩展
    listeners.environmentPrepared(bootstrapContext, environment);
    bindToSpringApplication(environment);
    // 自定义的EnvironmentConverter配置和解析，比较少用
    if (!this.isCustomEnvironment) {
        environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());
    }
    ConfigurationPropertySources.attach(environment);
    return environment;
}
```

下一步比较关键的步骤就是重量级的容器创建了，这里值得我们再细分几步来分析。Spring容器的相关知识可以参考我的另一篇文章《[SpringIoc]()》
### 2.1 **createApplicationContext**
![图10-创建容器](https://xvym.gitee.io/static/springboot/图10-创建容器.png)
首先，会通过反射的方式根据应用类型来创建容器类型实例，这里返回得容器类型是AnnotationConfigServletWebServerApplicationContext，然后会强制转换为ConfigurableApplicationContext
![图11-反射创建容器实例](https://xvym.gitee.io/static/springboot/图11-反射创建容器实例.png)
下一步加载exceptionReporters是通过SPI扩展的方式加载SpringBootExceptionReporter，这里主要是用于对springboot应用启动时发生的错误进行处理的，我们一般也不会去做太多的扩展，exceptionReporters也不会进入到容器中，在此我们不做深入分析
``` java
exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
```
### 2.2 **prepareContext** 
紧接着是容器准备阶段。这个阶段主要是进行一些容器的预配置工作。
``` java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
        // 设置容器的环境对象
		context.setEnvironment(environment);
        // 
		postProcessApplicationContext(context);
		applyInitializers(context);
		listeners.contextPrepared(context);
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		if (beanFactory instanceof DefaultListableBeanFactory) {
			((DefaultListableBeanFactory) beanFactory)
					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		if (this.lazyInitialization) {
			context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());
		}
		// Load the sources
		Set<Object> sources = getAllSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[0]));
		listeners.contextLoaded(context);
	}
```
