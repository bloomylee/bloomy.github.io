---
layout: post
title: " 戒懒系列（四）  createBean做了什么"
subtitle: ""
author: "bloomy"
header-img: "img/post-bg-css.jpg"
header-img-credit: "@WebdesignerDepot"
header-img-credit-href: "medium.com/@WebdesignerDepot/poll-should-css-become-more-like-a-programming-language-c74eb26a4270"
date:       2018-12-31
header-mask: 0.4
tags:

  - spring
  - 框架
---

>  此处分析居于spring 5.0.7源代码。

上一篇提到的 ConfigurationClassPostProcessor类的执行流程是在org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors函数中被触发的，invokeBeanFactoryPostProcessors触发了很多 *PostProcessors结尾类的执行流程。大家可以自己跟踪一下其它实现了BeanFactoryPostProcessor接口的类的执行过程。调用函数org.springframework.context.support.PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors。org.springframework.context.support.AbstractApplicationContext#invokeBeanFactoryPostProcessors函数完成初始化部分保存在beanDefinitionMap里面的类并且调用实现了BeanFactoryPostProcessor接口的类实例。ConfigurationClassPostProcessor完成添加beans到beanDefinitionMap。

org.springframework.context.support.AbstractApplicationContext#registerBeanPostProcessors函数完成初始化部分保存在beanDefinitionMap里面的类并且调用实现了BeanPostProcessor接口的类实例。

> beanDefinitionMap里面的bean实例化和依赖注入

org.springframework.beans.factory.support.AbstractBeanFactory#getBean这个函数大家不陌生吧。就是这个函数间接的完成了类的实例化和依赖注入。getBean调用doGetBean。

~~~java、
AbstractBeanFactory.java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) {
			// Eagerly check singleton cache for manually registered singletons.
			// 第一次调用
			Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
			
			//第二次调用
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
		}
		
		// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
					return createBean(beanName, 
					}
				
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
}

~~~

doGetBean两次调用了getSingleton函数，如果第一次是调用获取已经初始化和完成依赖注入的实体类了。已删除了一些无关代码。

第二次调用createBean才是初始化实例和依赖注入的过程。

~~~java
// AbstractBeanFactory.java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
		RootBeanDefinition mbdToUse = mbd;

		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}

	Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			return beanInstance;
}
~~~

首先看resolveBeanClass函数，如果一个bean是通过@Bean注解方式创建的那么这个函数就会返回null。

resolveBeforeInstantiation函数在实例化目标类对象之前，可以设置目标类的代理类，如果设置的目标类的代理类，那么后续就不实例化目标类了，只会实例化代理类。

~~~java
// AbstractBeanFactory.java
protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
~~~

applyBeanPostProcessorsBeforeInstantiation调用InstantiationAwareBeanPostProcessor接口实现类的postProcessBeforeInstantiation方法。

applyBeanPostProcessorsAfterInitialization调用BeanPostProcessor接口实现类的postProcessAfterInitialization方法。完成实例化类操作之前的一些操作，后续就要开始进行目标类的实例化和依赖注入了。

~~~java
// AbstractBeanFactory.java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}


		// Initialize the bean instance.
		Object exposedObject = bean;
		populateBean(beanName, mbd, instanceWrapper);
		exposedObject = initializeBean(beanName, exposedObject, mbd);
	

		return exposedObject;
	}
~~~

createBeanInstance函数创建类实例，首先经过一序列的过程，依赖注入目标类的构造函数。关注org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#autowireConstructor函数的实现。

如果目标类是@Bean注解方式实例化的那么调用org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#instantiateUsingFactoryMethod进行对象的实例化。

如果没有构造函数需要即无参的构造函数，直接org.springframework.beans.factory.support.AbstractAutowireCapableBeanFactory#instantiateBean进行实例化。

applyMergedBeanDefinitionPostProcessors函数调用了实现MergedBeanDefinitionPostProcessor接口的实现类postProcessMergedBeanDefinition方法。相当于一个拦截器，扩展beanDefinition。

populateBean函数调用实现InstantiationAwareBeanPostProcessor接口的实现类postProcessAfterInstantiation函数进行目标类中需要被依赖注入的成员属性。比如org.springframework.context.annotation.CommonAnnotationBeanPostProcessor、org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor。关键函数resolveDependency。

~~~java
//DefaultListableBeanFactory.java
public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

		descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
		if (Optional.class == descriptor.getDependencyType()) {
			return createOptionalDependency(descriptor, requestingBeanName);
		}
		else if (ObjectFactory.class == descriptor.getDependencyType() ||
				ObjectProvider.class == descriptor.getDependencyType()) {
			return new DependencyObjectProvider(descriptor, requestingBeanName);
		}
		else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
			return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
		}
		else {
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
					descriptor, requestingBeanName);
			if (result == null) {
				result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
	}
~~~

前面两个判断都会解析嵌套参数类型的依赖注入，比如如下两种方式的依赖注入。

```java
public void setConfigurers(List<WebMvcConfigurer> configurers) {
   if (!CollectionUtils.isEmpty(configurers)) {
      this.configurers.addWebMvcConfigurers(configurers);
   }
}

public EnableWebMvcConfiguration(
				ObjectProvider<WebMvcProperties> mvcPropertiesProvider,
				ObjectProvider<WebMvcRegistrations> mvcRegistrationsProvider,
				ListableBeanFactory beanFactory) {
			this.mvcProperties = mvcPropertiesProvider.getIfAvailable();
			this.mvcRegistrations = mvcRegistrationsProvider.getIfUnique();
			this.beanFactory = beanFactory;
		}
```

经过一系列的流程，完成了目标对象实例化、依赖注入这些过程。

org.springframework.context.support.AbstractApplicationContext#finishBeanFactoryInitialization函数里面进行剩余的bean对象的初始化和依赖注入详见

org.springframework.beans.factory.support.DefaultListableBeanFactory#preInstantiateSingletons函数。

至此spring framework的初始化过程就大部分就结束了。还有很多细节的东西没有列举出来。大家还是要多调试，多总结。org.springframework.beans.factory.support.DefaultListableBeanFactory#beanDefinitionMap和org.springframework.beans.factory.support.AbstractBeanFactory#mergedBeanDefinitions区别是什么？

org.springframework.beans.factory.support.DefaultListableBeanFactory#beanDefinitionNames的作用是什么？

通过@Import和@ImportBeanDefinitionRegistrar等注解怎么去扩展spring？

BeanFactoryPostProcessor、BeanPostProcessor、InstantiationAwareBeanPostProcessor、DestructionAwareBeanPostProcessor、SmartInstantiationAwareBeanPostProcessor这些接口扩展怎么被加入到beanDefinnitionMap里面的？





