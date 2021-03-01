---
title: springboot启动流程源码解析
date: 2021-02-23 11:46:16
tags:
- Java
- spring
- springboot
categories: article
---
好好啃一下这块硬骨头
<!--more-->
## 前言
虽然目前springboot目前已经迭代到2.5.x版本了，但鉴于springboot2.4.x版本后新增了springcloud的相关内容且相较于以前的版本代码风格有了比较大的变动，本文使用的springboot版本为2.3.9、JDK8版本。springboot的奥秘集中在三点：SpringApplication对象的创建，静态方法SpringApplication.run的执行以及一些关键注解的作用，所以本文会围绕这三点来进行源码分析。

## 1、构造SpringApplication对象
main方法调用SpringApplication.run静态方法，并传入main方法所在的启动引导类SpringbootLearningApplicationApplication.class
![图1-启动入口](https://xvym.gitee.io/static/springboot/图1-启动入口.png)
新建SpringApplication对象，并将引导类传入其构造方法
![图2-新建SpringApplication对象](https://xvym.gitee.io/static/springboot/图2-新建SpringApplication对象.png)
然后进入第一个关键步骤，**SpringApplication对象的创建**。我们先来看下SpringApplication的构造方法。
![图3-springApplication构造方法](https://xvym.gitee.io/static/springboot/图3-springApplication构造方法.png)
构造方法有两个参数，resourceLoader和primarySources，resourceLoader一般会置空，然后会在后续步骤中加载DefaultResourceLoader。primarySources是引导类，用于作为整个项目资源路径的引导标识，目前暂时还没见过有传多个引导类的情况。然后开始进行spring应用的资源初始化。
-. 首先是应用类别判断。WebApplicationType.deduceFromClasspath()方法通过ClassUtils.isPresent方法判断当前ClassLoader中是否存在相应的类来决定应该初始化哪种类型的应用，如果我们引入了springboot-starter-web模块，则web应用都类型是SERVLET，本文也是基于这个类型来进行分析。
-. 其次是各种资源的加载。在这里我们要重点关注**getSpringFactoriesInstances**方法，后面的ApplicationContextInitializer和ApplicationListener都是通过该方法进行加载的。initializers和listeners都是在此处进行赋值，便于后续spring容器刷新时对相应的对象做处理（回调）。getSpringFactoriesInstances方法会调起SpringFactoriesLoader.loadFactoryNames()方法，最终调用loadSpringFactories()方法，这个方法是Spring提供的[SPI机制](https://www.zhihu.com/search?type=content&q=SPI%20Java)的实现。SpringFactoriesLoader类会在第一次调用loadSpringFactories方法时扫描当前ClassLoader所处理的路径下所有META-INF/spring.factories文件的配置信息，来找到需要加载到Spring容器中的类，并缓存起来。这里的ClassLoader是ApplicationClassLoader，会加载用户类路径上的类库，具体可以参考这篇文章：《[JVM类加载机制]()》。
![图4-loadSpringFactories](https://xvym.gitee.io/static/springboot/图4-loadSpringFactories.png)
简单来说，就是通过查找用户路径类库中的META-INF/spring.factories文件，来找到需要加载到Spring容器中的类，如果我们需要在这些扩展点进行扩展，只需要将自定义的类按照Spring的SPI规范进行编写，便可以将类在启动阶段载入到SpringApplication对象中。springboot在启动流程的扩展机制，基本上都是使用这种方式进行的。
![图5-initializers、listeners](https://xvym.gitee.io/static/springboot/图5-initializers、listeners.png)
这里我们要搞清楚一个概念，和我们平时理解的工厂模式不同，虽然这个方法名叫做getSpringFactoriesInstances，但最终拿到的东西并不是一系列工厂实例集合，不是很清楚为什么要这样子命名。
-. 最后的deduceMainApplicationClass()是用来记录引导类的信息的，暂时没有什么很大作用。

至此，一个SpringApplication对象就构建完成了。

## 2、SpringApplication.run
SpringApplication对象构建完后，接下来就该让它跑起来了。在run方法中，有一些步骤我们没有必要去了解，这里我们只去关心有意义的步骤（比如设置显示器配置这种步骤我们直接略过就好）。

首先是进行监听器的初始化。这里的监听器和上面提到的监听器不同，实现的是SpringApplicationRunListener接口。我们可以深入以下两行代码看下其中的逻辑。
```Java
SpringApplicationRunListeners listeners = getRunListeners(args);
listeners.starting(bootstrapContext, this.mainApplicationClass);
```
getRunListeners方法还是Spring提供的一个SPI扩展点，在底层还是去读取META-INF/spring.factories来读取SpringApplicationRunListener，与其他的扩展机制并无二致，只不过在外部进行了一层封装，将所有的SpringApplicationRunListener封装为一个SpringApplicationRunListeners对象。
这里一般也不会有太多扩展，可以看到这里只加载了一个EventPublishRunListener，用于进行Spring应用启动的事件发布。
![图6-EventPublishingRunListener-1](https://xvym.gitee.io/static/springboot/图6-EventPublishingRunListener-1.png)
然后是下面这个starting方法。这里必须得吐槽一下，在2.4.x版本，springboot有很多代码在2.4.0版本后这个方法完全用lamda表达式重写了一遍，不得不说可读性是真的一般。这些改动主要是为了增加SpringCloud的相关支持，和一般的springboot应用启动流程起始并不是很相关。最开始我看的是2.4.2版本的代码，属实难顶，这也是为什么在前言中建议只想学习springboot启动流程的话可以去看2.3.x的代码。
![图7-2.3.x](https://xvym.gitee.io/static/springboot/图7-2.3.x.bmp)
![图8-2.4.x](https://xvym.gitee.io/static/springboot/图8-2.4.x.bmp)

starting方法很简单，就是将所有listener遍历并进行事件发布。由于一般SpringApplicationRunListener的实现类只有EventPublishingRunListener，所以我们可以进一步查看这个类的代码。
![图9-EventPublishingRunListener-2](https://xvym.gitee.io/static/springboot/图9-EventPublishingRunListener-2.png)
EventPublishingRunListener会构造一个SimpleApplicationEventMulticaster事件广播器，并将SpringApplication对象在构造方法中加载的Listener缓存在广播器中，并在starting方法会广播一个ApplicationStartingEvent，剩下的就是将事件广播给在onApplication方法中监听了这个事件的监听器了。详细步骤可以去阅读SimpleApplicationEventMulticaster的代码。所以如果我们要对spring启动监听器进行扩展，只需要按上面提到的SpringSPI方式将监听器注册到SpringApplication对象中（注意，不是容器哦，Spring容器到目前为止还没有进行创建），就可以对这些扩展点事件进行监听并执行相应的操作了。

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

### 2.1 **createApplicationContext**
下一个关键的步骤就是容器创建了，这里值得我们再细分几步来分析。Spring容器的相关知识可以参考我的另一篇文章《[SpringIoc]()》
![图10-创建容器](https://xvym.gitee.io/static/springboot/图10-创建容器.png)
首先，会通过反射的方式根据应用类型来创建容器类型实例，这里返回的容器类型是AnnotationConfigServletWebServerApplicationContext，然后会强制转换为ConfigurableApplicationContext
![图11-反射创建容器实例](https://xvym.gitee.io/static/springboot/图11-反射创建容器实例.png)
下一步加载exceptionReporters是通过SPI扩展的方式加载SpringBootExceptionReporter，这里主要是用于对springboot应用启动时发生的错误进行处理的，我们一般也不会去做太多的扩展，exceptionReporters也不会进入到容器中，在此我们不做深入分析
``` java
exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
```
### 2.2 **prepareContext** 
在容器实例创建好后，接着进行容器装配的准备工作。我们可以逐步分析一下其中的关键步骤。
``` java
private void prepareContext(ConfigurableApplicationContext context, ConfigurableEnvironment environment,
			SpringApplicationRunListeners listeners, ApplicationArguments applicationArguments, Banner printedBanner) {
		context.setEnvironment(environment);
        // 容器初始化后置处理，这一步会通过单利模式在容器的BeanFactory中预支ApplicationConversionService，用于进行各种数据转换处理
		postProcessApplicationContext(context);
		// 应用容器初始化器，详情可看下文
		applyInitializers(context);
		// 广播容器准备事件，同样，如果监听器需要在这一步执行操作，就可以监听这一事件
		listeners.contextPrepared(context);
		// 日志打印，打印进程和应用的信息
		if (this.logStartupInfo) {
			logStartupInfo(context.getParent() == null);
			logStartupProfileInfo(context);
		}
		// Add boot specific singleton beans
		ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
		// 注册特殊bean：命令行参数对象
		beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
		// 注册特殊bean：springboot logo打印对象
		if (printedBanner != null) {
			beanFactory.registerSingleton("springBootBanner", printedBanner);
		}
		// 设置是否允许同名的bean中后发现的覆盖先发现的，通过spring.main.allow-bean-definition-overriding配置，默认为true
		if (beanFactory instanceof DefaultListableBeanFactory) {
			((DefaultListableBeanFactory) beanFactory)
					.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		// 设置bean是否全局懒加载，通过spring.main.lazy-initialization篇日志，默认为false
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
其中，```applyInitializers(context)```这一步，在前文提到的，在SpringApplication实例构造阶段，spring会通过SPI机制加载ApplicationContextInitializer容器初始化器，这些初始化器在这一个步骤就会被启用，并在容器刷新前对容器进行必要的配置。如果我们编写的组件需要在容器刷新前进行必要的配置，那我们就可以通过容器初始化器来实现这个步骤。

接着就是重中之重，容器的刷新（refreshContext）了。逐步递进，refreshContext操作最终会进入到接口ConfigurableApplicationContext的refresh方法中并且由AbstractApplicationContext实现，这个方法使用了模板方法模式，定义了操作中的各个抽象步骤并且由子类进行具体实现，我们可以看下这个方法都做了什么。
```java 
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				initMessageSource();

				// Initialize event multicaster for this context.
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				onRefresh();

				// Check for listener beans and register them.
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```
我们来逐步的分析一下refresh方法做了什么。由于本文中的应用类型是SERVLET，故AbstractApplicationContext中的抽象方法都是由AnnotationConfigServletWebServerApplicationContext来进行实现的。
1. prepareRefresh();
	prepareRefresh方法设置容器启动时间和活动标志，以及通过调用initPropertySources()方法完成所有的property资源的初始化。总结下来，这一步没有做什么很重要的操作
	```java
	protected void prepareRefresh() {
			// Switch to active.
			this.startupDate = System.currentTimeMillis();
			// 设置容器状态
			this.closed.set(false);
			this.active.set(true);

			if (logger.isDebugEnabled()) {
				if (logger.isTraceEnabled()) {
					logger.trace("Refreshing " + this);
				}
				else {
					logger.debug("Refreshing " + getDisplayName());
				}
			}

			// Initialize any placeholder property sources in the context environment.
			// 初始化property sources，但一路debug发现里面什么也没做（所有判断条件均跳过了），暂时不清楚里面做了什么
			initPropertySources();

			// Validate that all properties marked as required are resolvable:
			// see ConfigurablePropertyResolver#setRequiredProperties
			// 校验是否所有必须的配置项都已配置好了
			getEnvironment().validateRequiredProperties();

			// Store pre-refresh ApplicationListeners...
			// 将SpringApplication实例初始化过程中加载的监听器添加到容器中，并标记为earlyApplicationListeners
			if (this.earlyApplicationListeners == null) {
				this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
			}
			else {
				// Reset local application listeners to pre-refresh state.
				this.applicationListeners.clear();
				this.applicationListeners.addAll(this.earlyApplicationListeners);
			}

			// Allow for the collection of early ApplicationEvents,
			// to be published once the multicaster is available...
			// 意义不明
			this.earlyApplicationEvents = new LinkedHashSet<>();
		}
	```

2. ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
	获取beanFactory。在obtainFreshBeanFactory()中有两步，refreshBeanFactory()和getBeanFactory();
	``` java 
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
			refreshBeanFactory();
			return getBeanFactory();
	}
	```
	refreshBeanFactory()方法由GenericApplicationContext实现，在其中spring并没有做任何容器相关的操作，但是通过compareAndSet操作来保证beanFactory只会别刷新一次，可以查看这个方法的注释
	```java
	/**
	 * Do nothing: We hold a single internal BeanFactory and rely on callers
	 * to register beans through our public methods (or the BeanFactory's).
	 * @see #registerBeanDefinition
	 */
	@Override
	protected final void refreshBeanFactory() throws IllegalStateException {
		if (!this.refreshed.compareAndSet(false, true)) {
			throw new IllegalStateException(
					"GenericApplicationContext does not support multiple refresh attempts: just call 'refresh' once");
		}
		this.beanFactory.setSerializationId(getId());
	}

	```
	beanFactory早在GenericApplicationContext的构造方法就已完成实例创建。在注释中提到，bean是只会通过registerBeanDefinition方法来注册的，所以在目前这一步刷新beanFactory的操作还不会进行bean的注册。

3. prepareBeanFactory(beanFactory);
	在这一步中，会将各种bean注册所必须的资源添加到beanFactory中，使beanFactory拥有注册bean的能力。
	```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		// 设置类加载器
		beanFactory.setBeanClassLoader(getClassLoader());
		// 设置el表达式解析器
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		// 设置配置属性解析器
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		// 将ApplicationContextAwareProcessor添加到后置处理器列表
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		// 忽略下列接口的自动装配（为什么要忽略？）
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		// 指定下列接口注入时使用当前的beanFactory和容器实现类来进行注入
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
	    // 将ApplicationListenerDetector添加到后置处理器列表
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		// 如果配置了LTW（一种代码织入器）的话，就将其添加到后置处理
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		// 将环境配置注册为bean
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		// 将系统配置注册为bean
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		// 将系统环境注册为bean
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
	```

4. invokeBeanFactoryPostProcessors(beanFactory)
	这一步是向beanFactory中实例化BeanFactoryPostProcessor。BeanFactoryPostProcessor是spring为我们提供的一个扩展点，允许我们在bean初始化前获得并修改bean的元信息。
	```java
	protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
		// 获取beanFactoryPostProcessor。在这里，通过getBeanFactoryPostProcessors()方法可以拿到三个预置的beanFactoryPostProcessor，这些都是在之前通过初始化器和监听器在容器准备阶段完成初始化的。
		PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());

		// Detect a LoadTimeWeaver and prepare for weaving, if found in the meantime
		// (e.g. through an @Bean method registered by ConfigurationClassPostProcessor)
		if (beanFactory.getTempClassLoader() == null && beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}
	}
	```
	我们可以进一步进入```PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors())```方法。
	这个方法非常长，我们可以只关注关键的几步。
	
