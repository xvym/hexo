---
title: 理解springioc（三）：理解高级容器的装配
date: 2021-01-30 16:01:59
tags:
- Java
- spring
categories: article
---
### 前言
写完理解springioc（一）（二），笔者通过对接口的分析，对spring的体系里各个角色的分工有了更深的理解。而从本文开始，就该真正探究spring的装配原理了。到这里，我们就不再对每一个ApplicationContext的实现类逐一进行分析了，直接将目光转到功能比较齐全的AnnotationConfigApplicationContext。
<!--more-->
### AnnotationConfigApplicationContext
AnnotationConfig代表这是一个支持JavaConfig注解式配置的容器，这个功能是通过**AnnotationConfigRegistry**接口来定义的；ServletWebServer代表这是一个**ServletWebServerApplicationContext**的子类，本质上是通过继承**GenericApplicationContext -> AbstractAplicationContext**来实现高级容器的基本功能。
首先，我们准备一个POJP类和一个配置类
```java
----------------------------------------
// POJO类
public class MyAnnotationConfigBean {

    private final Logger logger = LoggerFactory.getLogger(MyBean.class);

    @Value("${mybean.annotation.message}")
    private String message;

    public void test() {
        logger.info(message);
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }
}

----------------------------------------
// 配置类，通过@Configuration标识，PropertySourcesPlaceholderConfigurer用于读取.properties文件
@Configuration
public class MyAnnotationConfiguration {

    @Bean
    public MyAnnotationConfigBean myAnnotationConfigBean() {
        return new MyAnnotationConfigBean();
    }

    @Bean
    public PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer() {
        PropertySourcesPlaceholderConfigurer propertySourcesPlaceholderConfigurer = new PropertySourcesPlaceholderConfigurer();
        Resource resource = new ClassPathResource("application.properties");
        propertySourcesPlaceholderConfigurer.setLocation(resource);
        return propertySourcesPlaceholderConfigurer;
    }
}
```
然后是启动类
```java
public class AnnotationConfigApplicationContextTest {
    public static void main(String[] args) {
        AnnotationConfigApplicationContext ctx = new AnnotationConfigApplicationContext(MyAnnotationConfiguration.class);
        MyAnnotationConfigBean myAnnotationConfigBean = (MyAnnotationConfigBean) ctx.getBean("myAnnotationConfigBean");
        System.out.println(myAnnotationConfigBean.getMessage());
    }
}
```
我们来看下关键的步骤。  
**1. AnnotationConfigApplicationContext的构造方法**
```java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) {
    this();
    register(componentClasses);
    refresh();
}
```
- 1.1 this()，通过无参构造器新建BeanDefinitionReader和BeanDefinitionScanner，分别会负责解析和搜集配置
    ```java
    public AnnotationConfigApplicationContext() {
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }
    ```
- 1.2 register(componentClasses)，调用AnnotationConfigRegistry的register方法，对配置类进行解析
    ```java
    @Override
    public void register(Class<?>... componentClasses) {
        Assert.notEmpty(componentClasses, "At least one component class must be specified");
        this.reader.register(componentClasses);
    }
    ```
    底层会调用**AnnotatedBeanDefinitionReader**来进行BeanDefinition的解析，将配置类的BeanDefinition注册到AnnotationConfigApplicationContext的BeanFactory中。注意，在此处只是进行了配置类的BeanDefinition的解析，在配置类中定义的bean实际上是还没有解析的。我们可以看下此时BeanFactory中的beanDefinitionMap，可以看到，目前由用户定义的bean中，只完成了对MyAnnotationConfiguration的BeanDefinition解析。
    ![图1-registry阶段加载的BeanDefinition](https://xvym.gitee.io/static/理解springioc/三/图1-registry阶段加载的BeanDefinition.png)
- **1.3 refresh()**
    refresh方法在AbstractApplicationContext方法中实现，此处使用了模板方法模式，通过抽象父类定义容器刷新中的各个步骤，并给与子类重写覆盖的权利。此前提到过，refresh方法执行完毕后，要么所有的bean都完成了初始化，要么一个都没有初始化。在进入refresh方法前，除了图1中已经注册完毕的BeanDefinition，容器中目前是没有任何bean的信息的。接下来，我们直接进入spring容器核心中的核心，refresh方法吧。
    <details>
    <summary>refresh方法</summary>
    
    ```java
    @Override
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
    </details>
