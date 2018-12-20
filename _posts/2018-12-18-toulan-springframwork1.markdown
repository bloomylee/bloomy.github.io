---
layout: post
title: " 戒懒系列（一）  SpringFrameWork中的ApplicationContext"
subtitle: ""
author: "bloomy"
header-img: "img/post-bg-css.jpg"
header-img-credit: "@WebdesignerDepot"
header-img-credit-href: "medium.com/@WebdesignerDepot/poll-should-css-become-more-like-a-programming-language-c74eb26a4270"
date:       2018-12-18
header-mask: 0.3
tags:
  - spring
  - 框架
---


spring boot、spring cloud的出现，减少了很多项目的部署的复杂度、更加方便的集成和使用第三方功能模块。那些繁琐的配置大部分被spring 框架帮忙做了。相关介绍详见spring官方网站。

> spring boot官方文档有这样一句描述：

![img](/img/spring/sb.png)

---

> spring cloud官方文档是这样描述的

![img](/img/spring/sc.png)

---

spring cloud依赖spring boot，spring boot依赖spring framework，看来想要弄清楚他们的原理，首先得研究spring framework的东西了。透过官方文档，看到了一个[ApplicationContext]()的关键词。[以下截图都是依赖spring 5.0.9版本,spring5.0以下版本的没有*Reactive*相关的实现类。]()



打开IDE，搜索一下[ApplicationContext]()相关的定义，找到了一个和它相关的类信息如下图所示:

![img](/img/spring/aa.png)

---

[AbstractApplicationContext]()是个抽象类，有很多子类继承了它。如下图所示：

![img](/img/spring/jc.png)

还有一些子类图片中没有显示完全，排除Abstract开头的类，还是有很多子类，怎么找到对应的实现类呢？看一下类的名字，Annotation*开头的类是不是和注解有关的呢？

如果项目不是spring boot方式启动的，相关初始化堆栈如下图所示：

![img](/img/spring/odebug.png)

spring boot方式启动，初始化堆栈如下图所示：

![img](/img/spring/ndebug.png)

集成了spring cloud的spring boot启动，堆栈如下图所示：

![img](/img/spring/ndebug1.png)

从图片中refresh函数里面每个被调用的函数都值得详细的分析，除此之外堆栈信息里面的函数也是值得分析的。spring boot/cloud易学难懂。不把spring framework弄懂后面寸步难行呀。到这里才是进入spring生态的第一步，任重道远。

后续文章会继续分析spring framework 中 refresh函数里调用的函数。sf分析完后会继续分析spring boot核心逻辑。 

~~~java
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
                
~~~

上面每一个函数都能引出很多关键性的定义。时间，能力有限，如有不正确的地方请谅解。