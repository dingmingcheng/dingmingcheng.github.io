---
layout:     post
title:      "spring源码阅读（1）IOC与BeanFactory"
date:       2020-01-09
tags:
    - java
---

## 前言

(以下分析均基于spring-5.1.8)

因为spring这个框架在日常工作中的应用实在是太广泛了，便决定下一个阶段学学spring。因为spring非常庞大，一时不知道怎么下手，向hzk咨询了下，决定先看[spring的官方文档](https://docs.spring.io/spring/docs/5.2.3.BUILD-SNAPSHOT/spring-framework-reference/core.html#beans-introduction)，再结合相关书籍资料去读源码。spring虽然已经从springmvc到springboot，但核心还是spring-core,spring-context,spring-beans,spring-aop这些。

所以这一阶段的学习主要是针对ioc和aop的学习。另外希望能从源码的角度找到部分问题的答案：

1.spring bean的生命周期是怎么样的？

2.spring是如何解决循环依赖的？

3.如何对spring进行扩展

4.spring对aop的实现方式

...

## IOC

spring作为一个bean的容器，其最基本的功能就是IOC（控制反转），其中也主要包含了两部分：依赖查找和依赖注入，打个比方，一个大的项目中会有非常多的对象（单例或多例），各个对象间还会有依赖，如果没有spring给我们管理，特别是程序员水平不一的情况下，很可能会导致项目越来越复杂，维护成本越来越高。IOC正是帮我很好的解决了各个对象间依赖的问题，让大家更关注于业务本身。

而管理bean的正是我们接下来要看的BeanFactory

总的来说，sping从启动开始，bean主要有两个过程：

### bean的解析与注册

无论是早起的解析**xml**配置，还是通过**component-scan**的方式，其目的就是将类转换成**AbstractBeanDefinition**并添加到spring的中

### bean的实例化和初始化

实例化就是new一个对象，并把其所依赖的对象，属性字段也注入进去。其主要的过程就是将上一步中生成的**BeanDefinition**转换成实例对象

## BeanFactory总体架构

先上一张BeanFactory的依赖图，最上层的类便是
![](/img/in-post/spring/beanFactory.png)
DefaultListableBeanFactory：

第一眼看会觉得BeanFacory的体系非常复杂，而且部分类中的代码也是非常得长，还是得先仔细看看相关的接口

### AliasRegistry

其中就四个方法，均是别名相关

``` java
	void registerAlias(String name, String alias);
	void removeAlias(String alias);
	boolean isAlias(String name);
	String[] getAliases(String name);
```

### BeanDefinitionRegistry

``` java
void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException;
void removeBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
BeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;

boolean containsBeanDefinition(String beanName);

String[] getBeanDefinitionNames();

int getBeanDefinitionCount();

boolean isBeanNameInUse(String beanName);
```

这个接口也比较好理解，都是针对BeanDefinition的“增删查”操作

以上两个接口是bean的解析与注册相关，再来看看bean的实例化与初始化

### BeanFactory

最基本的接口，BeanFactory之于spring就如同chuck berry之于摇滚

忽略重载，其中主要是以下几个接口

``` java
//根据条件（比如名称，类名等）找到对应的bean实例
Object getBean(...);
//查找对应的ObjectProvider
ObjectProvider<T> getBeanProvider(...);
//是否存在bean
boolean containsBean(String name);
//是否为单例
boolean isSingleton(String name);
//是否为多例
boolean isPrototype(String name);
//名称和类是否匹配
boolean isTypeMatch(String name, Class<?> targetType)
//根据bean的名称获取bean的别名
String[] getAliases(String name);
//根据名称获取class类型
Class<?> getType(String name);
```

除了其中ObjectProvider是4.3才引进来的，其余的都好理解。ObjectProvider继承自ObjectFactory，我理解是给了使用者更大的自由度来决定注入的Bean，特别是在存在多个同类型的bean时。

### ListableBeanFactory

大致的接口如下：

``` java
boolean containsBeanDefinition(String beanName);
//查询beanDefinition总数
int getBeanDefinitionCount();
String[] getBeanDefinitionNames();
//根据
String[] getBeanNamesForType();
<T> Map<String, T> getBeansOfType();
String[] getBeanNamesForAnnotation();
  Map<String, Object> getBeansWithAnnotation();
<A extends Annotation> A findAnnotationOnBean()
```

正如Listable所表示，这些接口会用于列出bean以及beanDefinition

###HierarchicalBeanFactory

``` java
//拿到父工厂
BeanFactory getParentBeanFactory();
//当前工厂是否含有bean
boolean containsLocalBean(String name);
```

###AutowireCapableBeanFactory

和HierarchicalBeanFactory，ListableBeanFactory一样，该接口也只继承了BeanFactory接口。但相较于另外两个，这个会复杂很多

``` java

//-------------------------------------------------------------------------
// Typical methods for creating and populating external bean instances
//-------------------------------------------------------------------------
//根据类名创建bean
<T> T createBean(Class<T> beanClass) throws BeansException;
//重新装配一个bean对象(existingBean)
void autowireBean(Object existingBean) throws BeansException;
//会重新实例化一个beanName类型的对象（existingBean）
Object configureBean(Object existingBean, String beanName) throws BeansException;


//-------------------------------------------------------------------------
// Specialized methods for fine-grained control over the bean lifecycle
//-------------------------------------------------------------------------

//创建bean
Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;
//自动装配bean的相关参数
Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck) throws BeansException;
//自动装配bean的相关参数
void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
  throws BeansException;
//自动装配bean的相关参数
void applyBeanPropertyValues(Object existingBean, String beanName) throws BeansException;
//初始化一个bean
Object initializeBean(Object existingBean, String beanName) throws BeansException;

//初始化之前执行BeanPostProcessors
Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
  throws BeansException;

//初始化之后执行BeanPostProcessors
Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
  throws BeansException;

void destroyBean(Object existingBean);


//-------------------------------------------------------------------------
// Delegate methods for resolving injection points
//-------------------------------------------------------------------------

<T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType) throws BeansException;

Object resolveBeanByName(String name, DependencyDescriptor descriptor) throws BeansException;

@Nullable
Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName) throws BeansException;
//依赖查找
@Nullable
Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName, @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException;

```

这个接口主要包含了两部分内容：

1.创建以及组装bean实例相关的接口

2.进行依赖查找的接口，就比如项目中常常用到Autowird注解和Value注解，在自动装配bean的相关参数时就会调用AutowiredAnnotationBeanPostProcessor#postProcessPropertyValues方法进行依赖查找以及对象的注入，当然这个后续会具体去看看

### SingletonBeanRegistry

``` java
void registerSingleton(String beanName, Object singletonObject);
@Nullable
Object getSingleton(String beanName);
boolean containsSingleton(String beanName);
String[] getSingletonNames();
int getSingletonCount();
//返回所有的单例类
Object getSingletonMutex();
```

SingletonBeanRegistry也是最底层的接口，它实现的是所有单例类的添加，查找功能，比较简单明了，但很重要，主要就是在操作存放单例类的map

###ConfigurableBeanFactory

ConfigurableBeanFactory继承了HierarchicalBeanFactory和SingletonBeanRegistry，其中方法非常多，就不再列举了，涉及的点非常多，包括父工厂相关，类加载器相关，属性解析器相关，Bean后处理器相关，bean别名相关，FactoryBean相关等等。

### ConfigurableListableBeanFactory

ConfigurableListableBeanFactory继承了AutowireCapableBeanFactory和ConfigurableBeanFactory和ListableBeanFactory，新增的部分接口也不在此一一列举了

## 总结

至此，BeanFactory相关的架构，相关比较重要的接口也写完了。总的来说，接口非常多，但仔细看，还是可以发现各个接口各司其职，井井有条