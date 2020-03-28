---
layout:     post
title:      "spring源码阅读（2）IOC与Bean的实例化与初始化"
date:       2020-01-19
tags:
    - java
---

## 前言

之前写到spring中的bean主要分为两部分：

bean的解析与注册

bean的实例化和初始化

打算先写写bean的实例化和初始化，也就是springmvc也好，springboot也好，所有的bean已经转换成了BeanDefinition放入了内存中。

顺带在后面也会总结总结bean的生命周期

## 过程

上篇文章中看了下BeanFactory相关的接口，所以目前对整体的结构有了个大致的了解

BeanDefinition转化为Bean对象是从**AbstracBeanFacory#getBean()**方法开始的，很快可以找到，**doGetBean()**中有我们想要的东西，省略部分非关键代码，整理如下

``` java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
                          @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

  final String beanName = transformedBeanName(name);
  Object bean;

  // 提前获取已经暴露的对象，防止出现依赖循环，后续会再展开写
  Object sharedInstance = getSingleton(beanName);
  if (sharedInstance != null && args == null) {
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
  }
  else {
    //....
    // 校验是否为多例，且是否正在创建
    //....
    // 如果当前beanFactory无法找到该BeanDefination，则递归到父BeanFactory中查找

    try {
      //获取BeanDefinition
      final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
      //.....
      //如果当前Bean有依赖（DependsOn）,则递归创建bean

      // Create bean instance.
      if (mbd.isSingleton()) {
        sharedInstance = getSingleton(beanName, () -> {
            return createBean(beanName, mbd, args);
        });
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
      }

      else if (mbd.isPrototype()) {
        // It's a prototype -> create a new instance.
        Object prototypeInstance = null;
        prototypeInstance = createBean(beanName, mbd, args);
        bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
      }
    }
    //...
    //对指定的scope进行实例化
    //...
    catch (BeansException ex) {
    }
  }
  return (T) bean;
}

```

可以看到，其中对单例和多例采取了不同的处理方式，多例的直接通过**createBean()**方法创建了一个新的对象实例，而单例是向**getSingleton()**中传入了一个**ObjectFacory**的匿名类，**getSingleton()**方法中核心的代码其实就是调用**ObjectFacory.getObject();**（注：getSingleton()中有部分代码涉及到spring解决循环依赖的问题，之前会另外写文章分析），那其实可以直接来看下**createBean()**方法

### createBean

关键代码

```java
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
    //....省略部分不关键代码
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
      return bean;
    }
    //....省略部分不关键代码
    Object beanInstance = doCreateBean(beanName, mbdToUse, args);
}
```

createBean中有两块需要注意，一个*resolveBeforeInstantiation*，一个是*doCreateBean*

resolveBeforeInstantiation重要的原因是：这块是和之后的AOP有非常重要的联系

### resolveBeforeInstantiation

深入两层以后，发现了这段代码

``` java
for (BeanPostProcessor bp : getBeanPostProcessors()) {
  if (bp instanceof InstantiationAwareBeanPostProcessor) {
    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
    Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
    if (result != null) {
      return result;
    }
  }
}
```

在这里，引出了我们接触到的第一个BeanPostProcessor:InstantiationAwareBeanPostProcessor，它相较于BeanPostProcessor多了**postProcessBeforeInstantiation()**和**postProcessAfterInstantiation()**,**postProcessProperties()**方法，此处执行的便是**postProcessBeforeInstantiation**方法，即实例化前执行

### doCreateBean

``` java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
  throws BeanCreationException {

  //1.实例化bean
  BeanWrapper instanceWrapper = null;
  instanceWrapper = createBeanInstance(beanName, mbd, args);
  final Object bean = instanceWrapper.getWrappedInstance();

  // Allow post-processors to modify the merged bean definition
  // 2.MergedBeanDefinitionPostProcessor处理
  synchronized (mbd.postProcessingLock) {
    if (!mbd.postProcessed) {
      applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
      mbd.postProcessed = true;
    }
  }


  //3.循环依赖相关
  boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                    isSingletonCurrentlyInCreation(beanName));
  if (earlySingletonExposure) {
    addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
  }

  // 4.初始化bean
  Object exposedObject = bean;
  populateBean(beanName, mbd, instanceWrapper);
  exposedObject = initializeBean(beanName, exposedObject, mbd);

  if (earlySingletonExposure) {
    //....
    //5.循环依赖相关
  }

  return exposedObject;
}
```

该方法中有非常多重要的步骤，按顺序主要是这几个

- 将RootBeanDefinition转换为BeanWrapper，并创建新的一个实例

这一步其实内部逻辑挺复杂的，主要是通过java的反射，找到一个合适的构造器来进行实例化。就不细看了

- MergedBeanDefinitionPostProcessor处理

又碰到了一种新的BeanPostProcessor，做了postProcessMergedBeanDefinition操作。这个其实也挺重要，就比如@Autowird，@PostConstruct和@PreDestroy注解，就是在这里做了一个预处理。

- 循环依赖相关处理
- bean的初始化

两个重要的函数，**populateBean**以及**initializeBean**，前者是做了依赖属性的填充，后者是对bean作了初始化的操作

- 循环依赖相关处理

### populateBean

``` java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
  //...
	//InstantiationAwareBeanPostProcessor执行postProcessAfterInstantiation，如果返回true，则停止继续处理
  PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);
  if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME || mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
    // Add property values based on autowire by name if applicable.
    //按名称注入
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_NAME) {
      autowireByName(beanName, mbd, bw, newPvs);
    }
    // Add property values based on autowire by type if applicable.
        //按类型注入
    if (mbd.getResolvedAutowireMode() == AUTOWIRE_BY_TYPE) {
      autowireByType(beanName, mbd, bw, newPvs);
    }
    pvs = newPvs;
  }

  for (BeanPostProcessor bp : getBeanPostProcessors()) {
    if (bp instanceof InstantiationAwareBeanPostProcessor) {
      InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
      //获取依赖的属性
      PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
      if (pvsToUse == null) {
        //已经被打上@Deprecated
        filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
      }
      pvs = pvsToUse;
    }
  }
  if (pvs != null) {
    //将pvs赋值到对应的对象上
    applyPropertyValues(beanName, mbd, bw, pvs);
  }
}

```

populateBean分为以下几步：

- InstantiationAwareBeanPostProcessor执行postProcessAfterInstantiation
- 按名称注入（即getBean()）和按类型注入，按类型注入会复杂一些，按类型注入的主要的逻辑在**AutowireCapableBeanFactory#resolveDependency**中
- InstantiationAwareBeanPostProcessor执行postProcessProperties，这一步也很重要，我们常用的@Autowired注解就是在此处进行依赖查找（其实也是调用了**AutowireCapableBeanFactory#resolveDependency**）
- 最后通过**applyPropertyValues**方法，将参数赋值到bean的对象中，这块又涉及了比较BeanWrapper，此处就没细看了

### initializeBean

先看看个人简化后的代码

``` java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {

  //调用Aware相关的方法，这里涉及到spring的另外一个功能Aware，里面代码也很直观
  invokeAwareMethods(beanName, bean);
  //大名鼎鼎的BeanPostProcessor
  wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
  //InitializingBean
  boolean isInitializingBean = (wrappedBean instanceof InitializingBean);
  if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
      ((InitializingBean) bean).afterPropertiesSet();
  }
	//init-method
  if (mbd != null && bean.getClass() != NullBean.class) {
    String initMethodName = mbd.getInitMethodName();
    if (StringUtils.hasLength(initMethodName) &&
        !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
        !mbd.isExternallyManagedInitMethod(initMethodName)) {
      invokeCustomInitMethod(beanName, bean, mbd);
    }
  }
  //大名鼎鼎的BeanPostProcessor
  wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
  return wrappedBean;
}

```

如果日常我们spring用的多的话，这段就很容易看懂了，也涉及到了面试常常会问到的Bean的生命周期

至此，createBean就完成了，可以理解为dfs到头了，接着回到doGetBean中看getObjectForBeanInstance方法

### getObjectForBeanInstance

这块其实是FactoryBean的功能，里面会调用**object = factory.getObject();**方法，并对生成的对象调用BeanPostProcessors的方法

## bean的生命周期

通过以上的阅读分析，bean的生命周期就非常清晰了：

1. InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation()  例子：AOP
2. 实例化，即构造器
3. MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition(). 例子：@Autowird @PostContruct @PreDestroy的初始化解析
4. InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation()
5. InstantiationAwareBeanPostProcessor#postProcessProperties(). 比如@Autowired的属性字段
6. 相关属性以及依赖的注入
7. Aware相关方法的执行
8. BeanPostProcessor#postProcessBeforeInitialization()
9. InitializingBean#afterPropertiesSet
10. Init-method
11. BeanPostProcessor#postProcessAfterInitialization()

## 总结

至此，已基本清楚bean的生命周期，后续还得继续看看依赖循环的解决，各种BeanPostProcessor的实现

