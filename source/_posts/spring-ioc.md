---
title: spring ioc
date: 2021-02-25 23:22:06
tags:
- Java
- spring
- springboot
categories: article
---
把以前总结的spring内容重新整理一下
<!--more-->
# 为什么需要控制反转？
这篇问题的第一个回答我觉得解释得很好 [点我跳转](https://www.zhihu.com/question/23277575/answer/169698662)

# Bean注册的基本逻辑
通过配置文件或注解标记Bean实例的生成方法，将该方法返回的实例添加到容器中进行管理

# BeanFactory
BeanFacatory位于org.springframework.beans.factory包下
BeanFactory是顶层的IOC容器接口，所有IOC容器的实现均需要实现该接口。它是获取Bean的门面接口，客户端通过该接口来获取Bean或是获取Bean的关键信息。但是BeanFactory并不会负责生成Bean的元信息（BeanDefinition），还需要其他的类来协助进行Bean的实际生成。

# BeanDefinition
BeanDefinition位于org.springframework.beans.factory.config包下
每一个受容器管理的对象，在容器中都有一个BeanDefinition的实例与之对应，负责保存对象所有的必要信息（即上述元信息），包括对象的class类型、是否为抽象类、构造方法参数及其他属性等等。BeanFactory通过这些信息为客户端返回一个完备可用的Bean对象

# BeanDefinitionReader
BeanDefinitionReader位于org.springframework.beans.factory.support包下
BeanDefinitionReader的作用是读取Spring配置文件中的内容，并转换为BeanDefinition。
BeanDefinitionReader有三种实现，XmlBeanDefinitionReader、PropertiesBeanDefinitionReader、GroovyBeanDefinitionReader，类如其名，就是从不同类型的配置文件中读取Bean的元信息。最常用的是XmlBeanDefinitionReader，负责从xml文件中读取Bean的装配信息。

# ConfigurationClassBeanDefinitionReader
ConfigurationClassBeanDefinitionReader位于org.springframework.context.annotation，负责加载通过JavaConfig方式配置的Bean。同时，在这个类中还引入BeanDefinitionReader的三种实现类，用于加载由@Import注解引入的配置文件中定义的Bean元信息。

# DefaultListableBeanFactory
DefaultListableBeanFactory位于org.springframework.beans.factory.support包下
DefaultListableBeanFactory是一个最常用的能真正实例化Bean的一个BeanFactory实现类

# ApplicationContext
Web程序常用的ApplicationContext间接继承自BeanFactory，是一个高级的IOC容器，其类图关系如下 
![ApplicationContext的继承关系](E:\学习文章\static\pic\6724b26c-e41d-4769-948f-3e2638d0b8ac.bmp)

# FactoryBean
FactoryBean位于org.springframework.beans.factory包下
有时候，为了达到解耦的目的，我们会通过工厂模式来进行Bean的管理。FactoryBean是spring定义的一种用于生产Bean的工厂接口。实现了该接口的工厂类会实现三个方法：  
![FactoryBean](D:\Code\Notes\Spring\FactoryBean定义的三个方法.PNG)  
其中，getObject方法会返回工厂生产的对象实例，Spring会将该方法生产的实例注册到容器中。如果我们根据实现了FactoryBean的工厂类的BeanId来获取实例，返回的会是getObjcet方法返回的实例。若需要获取实际的FactoryBean，则要在BeanId前加上&号来进行标识

# 容器启动及Bean装配流程分析
