---
layout: post
title: " 戒懒系列（二）  AbstractApplicationContext中的refresh"
subtitle: ""
author: "bloomy"
header-img: "img/post-bg-infinity.jpg"
date:       2018-12-19
header-mask: 0.4
tags:

  - spring
  - 框架
---

>  此处分析居于spring 3.2.18源代码。spring boot分析居于spring 5.x版本。

不知大家没有对照着上一篇文章描述到的非spring boot方式启动堆栈信息进行调试。从[FrameWorkServlet]()中的[initWebApplicationContext]()里的函数也很重要，涉及到了创建了[ApplicationContext]()实例、设置[Environment]()、[ConfigLocation]()等操作。

> FrameWorkServlet.java

~~~java
public abstract class FrameworkServlet extends HttpServletBean {
    
    private Class<?> contextClass = DEFAULT_CONTEXT_CLASS;
    public static final Class<?> DEFAULT_CONTEXT_CLASS = XmlWebApplicationContext.class;
    
public Class<?> getContextClass() {
		return this.contextClass;
	}
}
~~~

~~~java
protected WebApplicationContex  initWebApplicationContext() {
......
......
if (wac == null) {
			wac = findWebApplicationContext();
		}
		if (wac == null) {
			// No context instance is defined for this servlet -> create a local one
			wac = createWebApplicationContext(rootContext);
		}

		if (!this.refreshEventReceived) {
			// Either the context is not a ConfigurableApplicationContext with refresh
			// support or the context injected at construction time had already been
			// refreshed -> trigger initial onRefresh manually here.
			onRefresh(wac);
		}
~~~

~~~java
protected WebApplicationContext createWebApplicationContext(ApplicationContext parent) {
......
......
Class<?> contextClass = getContextClass();
ConfigurableWebApplicationContext wac =
				(ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);

		wac.setEnvironment(getEnvironment());
		wac.setParent(parent);
		wac.setConfigLocation(getContextConfigLocation());

		configureAndRefreshWebApplicationContext(wac);
}
~~~

执行完第六行代码后创建了XmlWebApplicationContext类的对象contextClass=XmlWebApplicationContext.class。后续继续初始化ApplicationContext的成员变量。

执行[createEnvironment]()的过程中会触发 [customizePropertySources]()的执行，完成Enviroment的初始化。

~~~java
protected ConfigurableEnvironment createEnvironment() {
		return new StandardServletEnvironment();
	}

public class StandardServletEnvironment extends StandardEnvironment implements ConfigurableWebEnvironment {
    .....
    .....
}

public class StandardEnvironment extends AbstractEnvironment {
    /** System environment property source name: {@value} */
	public static final String SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME = "systemEnvironment";

	/** JVM system properties property source name: {@value} */
	public static final String SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME = "systemProperties";
    
    @Override
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
		propertySources.addLast(new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
	}
}
~~~

> Enviroment保存了什么数据？

![img](/img/spring/2/env.png)

> MapPropertySource里面保存了jvm的一些配置信息，如图所示：

![img](/img/spring/2/jvminfo.png)

> SystemEnvironmentPropertySource里面保存了系统配置信息，如同所示：

![img](/img/spring/2/system.png)

大家想一想 ,如果配置文件里面配置了test.age=2,如下代码里面age值是怎么被设置为2的？

~~~java
@Value("${test.age}")
private int age; //age=2
~~~

赋值过程和Enviroment有关系？我们自定义的一些配置信息会存储在Enviroment里面？

>  obtainFreshBeanFactory流程

refresh过程中 obtainFreshBeanFactory经过一系列的调用，会设置FrameworkServlet类里面 contextConfigLocation 成员的值，contextConfigLocation=WEB-INF/dispatcherServlet-servlet.xml。

从web.xml中解析出servlet-name字段值（解析过程是在tomcat中完成的，调用了StandardWrapper.getServletConfig()返回），然后调用getDefaultConfigLocations()函数拼接字符串值。

~~~xml
<servlet>
	<servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
~~~

详细过程参考如下函数：

~~~java
//XmlWebApplicationContext.java
public class XmlWebApplicationContext extends AbstractRefreshableWebApplicationContext {
    
/** Default config location for the root context */
	public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";

	/** Default prefix for building a config location for a namespace */
	public static final String DEFAULT_CONFIG_LOCATION_PREFIX = "/WEB-INF/";

	/** Default suffix for building a config location for a namespace */
	public static final String DEFAULT_CONFIG_LOCATION_SUFFIX = ".xml";

protected String[] getDefaultConfigLocations() {
		if (getNamespace() != null) {
			return new String[] {DEFAULT_CONFIG_LOCATION_PREFIX + getNamespace() + DEFAULT_CONFIG_LOCATION_SUFFIX};
		}
		else {
			return new String[] {DEFAULT_CONFIG_LOCATION};
		}
}
~~~

执行到loadBeanDefinitions()函数，循环加载WEB-INF/web.xml里面配置的servlet-name上面已经分析完。

~~~java
//XmlWebApplicationContext.java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}
~~~

reader.loadBeanDefinitions(configLocation)函数会加载解析dispatcherServlet-servlet.xml文件

~~~java
//DefaultBeanDefinitionDocumentReader.java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
~~~

delegate.parseCustomElement(root) 执行到parseCustomElement()函数，ele参数就是dispatcherServlet-servlet.xml文件里面的字段值，通过ele参数获取到namespaceUri值。

~~~java
//BeanDefinitionParserDelegate.java
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
		String namespaceUri = getNamespaceURI(ele);
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
~~~

this.readerContext.getNamespaceHandlerResolver()获取到如图所示的数据：

![img](/img/spring/2/handler.png)

所有数据其实是从META-INF/spring.handlers文件中加载的。

接下来handler.parse函数调用findParserForElement函数根据参数找到对应的解析类，相关解析类都实现了 [BeanDefinitionParser]()接口，很多很多解析类。这里就不上图了。找到对应的解析类后，在继续调用具体parse方法的实现类型。

~~~java
//BeanDefinitionParser.java
public interface BeanDefinitionParser {
	BeanDefinition parse(Element element, ParserContext parserContext);

}
~~~

~~~java
//NamespaceHandlerSupport.java
public BeanDefinition parse(Element element, ParserContext parserContext) {
		return findParserForElement(element, parserContext).parse(element, parserContext);
	}
~~~

在BeanDefinitionParser接口的多个实现类中有实现registerComponents方法。这个方法是干什么？