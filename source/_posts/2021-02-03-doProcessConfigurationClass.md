---
title: 2021-02-03 doProcessConfigurationClass
date: 2021-03-12 23:02:04
tags:
---
## 前言
本来是想通过一篇系列文章来有条理地进行spring的相关总结的，但是无奈核心流程实在是太长。。。有必要先将其中一部分拿出来单独记录一下
<!--more-->
## 提前说一下，由@Import引起的循环引入是无法解决的，只能抛出异常
## ConfigurationClassParser#doProcessConfigurationClass
在该方法中，我们将会看到对很多熟悉的注解进行解析的原理。
```java
@Nullable
protected final SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
        throws IOException {

    // 先判断配置类是否标注@Component（@Configuration派生自@Componment）
    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // Recursively process any member (nested) classes first
        // 通过递归解析配置类中是否还存在内部类标注有@Configuration（或@Component的派生），且判断是否存在由@Import引起的的循环引入，内部通过一个自定义栈ImportStack来进行配置类处理，外部先入栈后处理，内部后入栈先处理，然后递归调用ConfigurationClassParser#processConfigurationClass方法，最终将所有类内部嵌套的信息存储到configurationClasses中，但这些内部@Component此时还不会将BeanDefinition注册到容器中
        processMemberClasses(configClass, sourceClass, filter);
    }

    // Process any @PropertySource annotations
    // 处理配置类上标注的@PropertySource注解，最终将解析出来的配置项存储到environment.propertySources中
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), PropertySources.class,
            org.springframework.context.annotation.PropertySource.class)) {
        if (this.environment instanceof ConfigurableEnvironment) {
            processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Process any @ComponentScan annotations
    // 处理配置类上标注的@ComponentScan注解，首先获取@ComponentScan上的元信息
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScans.class, ComponentScan.class);
    if (!componentScans.isEmpty() &&
            !this.conditionEvaluator.shouldSkip(sourceClass.getMetadata(), ConfigurationPhase.REGISTER_BEAN)) {
        // 遍历处理所有的@ComponentScan
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            // 从@ComponentScan定义的包路径下搜索所有的BeanDefinition并封装为BeanDefinitionHolder方便操作，在ComponentScanAnnotationParser#parse()方法中，若未指定@ComponentScan的路径，就会使用主配置类的所在包作为路径，通过ClassPathBeanDefinitionScanner进行BeanDefinition的收集和注册。
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                // 如果在BeanDefinition中再次发现了@Component注解，则递归进行doProcessConfigurationClass。在递归结束后，所有通过@Component进行标识的类（不包括内部类）的BeanDefinition就注册完毕了。
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    // 处理@Import注解，里面同样会判断是否由@Import引起的循环引入（底层有个判断是否成环的逻辑，有兴趣可以阅读一下），并提前报错；否则嵌套@Import的配置类同样会通过一个自定义栈ImportStack的操作来进行处理，先入栈后处理，内部后入栈先处理，递归调用ConfigurationClassParser#processConfigurationClass方法，最终将所有@Import引入的类信息存储到configurationClasses中。同样，如果@Import引入的是一个内部类，此时也还不会将其BeanDefinition注册到容器中。
    processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

    // Process any @ImportResource annotations
    // 接着是处理ImportResource注解引入的xml等其他配置文件。
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            // 此处看起来是对配置文件的占位符进行处理，但是实际上我并没有找到处理的地方。。。
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            // 添加到已ImporteredResource中，看名字应该是避免重新加载
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java") &&
                !this.knownSuperclasses.containsKey(superclass)) {
            this.knownSuperclasses.put(superclass, configClass);
            // Superclass found, return its annotation metadata and recurse
            return sourceClass.getSuperClass();
        }
    }

    // No superclass -> processing is complete
    return null;
}
```