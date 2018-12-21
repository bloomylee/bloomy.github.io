---
layout: post
title: " 戒懒系列（二）  AbstractApplicationContext中的refresh"
subtitle: ""
author: "bloomy"
header-img: "img/post-bg-css.jpg"
header-img-credit: "@WebdesignerDepot"
header-img-credit-href: "medium.com/@WebdesignerDepot/poll-should-css-become-more-like-a-programming-language-c74eb26a4270"
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

refresh过程中 obtainFreshBeanFactory经过一系列的调用，会设置FrameworkServlet类里面 contextConfigLocation 成员的值。从web.xml中解析出servlet-name字段值（解析过程是在tomcat中完成的，调用了StandardWrapper.getServletConfig())，然后调用getDefaultConfigLocations()函数拼接字符串值。servlet-name定义如下。

~~~xml
<servlet>
	<servlet-name>dispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <load-on-startup>1</load-on-startup>
</servlet>
~~~

getDefaultConfigLocations拼接路径后contextConfigLocation=WEB-INF/dispatcherServlet-servlet.xml。

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

XmlWebApplicationContext里的loadBeanDefinitions函数进过一些列执行最终会执行到XmlBeanDefinitionReader里的loadBeanDefinitions()函数，循环加载WEB-INF/web.xml里面配置的servlet-name上面已有分析。

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

调用链如图所示：

![img](/img/spring/2/use.png)

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

this.readerContext.getNamespaceHandlerResolver()获取到DefaultNamespaceHandlerResolver类的对象，然后调用resolve()。

~~~java
//DefaultNamespaceHandlerResolver.java
public NamespaceHandler resolve(String namespaceUri) {
		Map<String, Object> handlerMappings = getHandlerMappings();
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			String className = (String) handlerOrClassName;
			Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
				namespaceHandler.init();
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}		
	}
~~~

resolve()调用getHandlerMappings()，ClassUtils.forName动态加载getHandlerMappings()函数返回的数据并且实例化对象，namespaceHandler.init()初始化或者自定义对注解的解析类。参见NamespaceHandlerSupport的子类、NamespaceHandler接口的实现。

~~~java
private Map<String, Object> getHandlerMappings() {
......
......
Properties mappings =				PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
~~~

NacosNamespaceHandler就是自定义的NamespaceHandler实现类，扩展自定义解析类。

~~~java
public class NacosNamespaceHandler extends NamespaceHandlerSupport {

    @Override
    public void init() {
        registerBeanDefinitionParser("annotation-driven", new NacosAnnotationDrivenBeanDefinitionParser());
        registerBeanDefinitionParser("global-properties", new GlobalNacosPropertiesBeanDefinitionParser());
        registerBeanDefinitionParser("property-source", new NacosPropertySourceBeanDefinitionParser());
    }
}
~~~

PropertiesLoaderUtils.loadAllProperties函数加载解析加载、解析META-INFO/spring.handlers文件。getHandlerMappings()函数返回如图所示的数据：

![img](/img/spring/2/handler.png)

接下来parseCustomElement函数中handler.parse函数调用findParserForElement函数根据参数找到对应的解析类，findParserForElement函数中 this.parsers 保存的就是NamespaceHandler实现类中init()函数调用registerBeanDefinitionParser注册的解析器。 相关解析类都实现了 [BeanDefinitionParser]()接口，很多很多解析类。这里就不上图了。找到对应的解析类后，在继续调用具体parse方法的实现类型。

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

private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		String localName = parserContext.getDelegate().getLocalName(element);
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
}
~~~

### AnnotationConfigBeanDefinitionParser类中parse做了些什么？

~~~java
//AnnotationConfigBeanDefinitionParser.java

public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);

		// Obtain bean definitions for all relevant BeanPostProcessors.
		Set<BeanDefinitionHolder> processorDefinitions =
				AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);

		// Register component for the surrounding <context:annotation-config> element.
		CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
		parserContext.pushContainingComponent(compDefinition);

		// Nest the concrete beans in the surrounding component.
		for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
			parserContext.registerComponent(new BeanComponentDefinition(processorDefinition));
		}

		// Finally register the composite component.
		parserContext.popAndRegisterContainingComponent();

		return null;
	}
~~~

AnnotationConfigUtils.registerAnnotationConfigProcessors----->AnnotationConfigUtils.registerPostProcessor--->BeanDefinitionRegistry.registerBeanDefinition，创建BeanDefinition结构和注册到DefaultListableBeanFactory类的成员变量beanDefinitionMap中，后续使用。

### invokeBeanFactoryPostProcessors过程

~~~java
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
    ........
    ........
        
    Map<String, BeanDefinitionRegistryPostProcessor> beanMap =
					beanFactory.getBeansOfType(BeanDefinitionRegistryPostProcessor.class, true, false);
			List<BeanDefinitionRegistryPostProcessor> registryPostProcessorBeans =
					new ArrayList<BeanDefinitionRegistryPostProcessor>(beanMap.values());
			OrderComparator.sort(registryPostProcessorBeans);
			for (BeanDefinitionRegistryPostProcessor postProcessor : registryPostProcessorBeans) {
				postProcessor.postProcessBeanDefinitionRegistry(registry);
			}
			invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(registryPostProcessorBeans, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
			processedBeans.addAll(beanMap.keySet());
		}
		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(getBeanFactoryPostProcessors(), beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		List<String> orderedPostProcessorNames = new ArrayList<String>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<String>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		OrderComparator.sort(priorityOrderedPostProcessors);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		OrderComparator.sort(orderedPostProcessors);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<BeanFactoryPostProcessor>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);
}
~~~

大家还记得之前BeanDefinitionParser接口其中一个实现类AnnotationConfigBeanDefinitionParser的parse方法？往上翻一下，注意AnnotationConfigUtils.registerAnnotationConfigProcessors函数。里面多次调用了registerPostProcessor上面也说了，封装BeanDefinition结构，保存到beanMap里面。后续排序后，按照顺序@PriorityOrdered-->@Ordered--->没有这两个注解的类分别处理。

~~~java
//AnnotationConfigUtils.java
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
			BeanDefinitionRegistry registry, Object source) {

		Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<BeanDefinitionHolder>(4);

		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		if (!registry.containsBeanDefinition(REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(RequiredAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, REQUIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
		if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		// Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
		if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition();
			try {
				def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
						AnnotationConfigUtils.class.getClassLoader()));
			}
			catch (ClassNotFoundException ex) {
				throw new IllegalStateException(
						"Cannot load optional framework class: " + PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME, ex);
			}
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
		}

		return beanDefs;
	}

	private static BeanDefinitionHolder registerPostProcessor(
			BeanDefinitionRegistry registry, RootBeanDefinition definition, String beanName) {

		definition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(beanName, definition);
		return new BeanDefinitionHolder(definition, beanName);
	}
~~~

beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false)函数从beanDefinitionMap里面获取到指定类型的对象。这里首先获取到ConfigurationClassPostProcessor类

~~~java
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
		Ordered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
            
   public int getOrder() {
		return Ordered.HIGHEST_PRECEDENCE;
        //-2147483648 越小排序越靠前
	}
}
~~~

~~~java
//ConfigurationClassPostProcessor.java
public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
		RootBeanDefinition iabpp = new RootBeanDefinition(ImportAwareBeanPostProcessor.class);
		iabpp.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(IMPORT_AWARE_PROCESSOR_BEAN_NAME, iabpp);

		int registryId = System.identityHashCode(registry);
		if (this.registriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanDefinitionRegistry already called for this post-processor against " + registry);
		}
		if (this.factoriesPostProcessed.contains(registryId)) {
			throw new IllegalStateException(
					"postProcessBeanFactory already called for this post-processor against " + registry);
		}
		this.registriesPostProcessed.add(registryId);

		processConfigBeanDefinitions(registry);
	}
~~~

postProcessBeanDefinitionRegistry添加ImportAwareBeanPostProcessor到beanDefinitionMap里面,然后调用 processConfigBeanDefinitions函数，判断是否beanDefinitionMap里面保存的BeanDefinition对象信息是否有@Configuration，@Component、@Bean注解，创建 ConfigurationClassParser对象对beanDefinitionMap里面的BeanDefinition信息做解析。很重要这个函数。

~~~java
//ConfigurationClassPostProcessor.java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    Set<BeanDefinitionHolder> configCandidates = new LinkedHashSet<BeanDefinitionHolder>();
            for (String beanName : registry.getBeanDefinitionNames()) {
                BeanDefinition beanDef = registry.getBeanDefinition(beanName);
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                    configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
	......
    ......
}
~~~

创建ConfigurationClassParser parse对象，configCandidates容器里面保存有@Configuration、@Component和@Bean注解的BeanDefinition对象。根据不同的类型进行解析。

~~~java
//ConfigurationClassPostProcessor.java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ConfigurationClassParser parser = new ConfigurationClassParser(
                    this.metadataReaderFactory, this.problemReporter, this.environment,
                    this.resourceLoader, this.componentScanBeanNameGenerator, registry);
            for (BeanDefinitionHolder holder : configCandidates) {
                BeanDefinition bd = holder.getBeanDefinition();
                try {
                    if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
                        parser.parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
                    }
                    else {
                        parser.parse(bd.getBeanClassName(), holder.getBeanName());
                    }
                }
                catch (IOException ex) {
                    throw new BeanDefinitionStoreException("Failed to load bean class: " + bd.getBeanClassName(), ex);
                }
            }
}
~~~

写在此片最后，parse.parse循环解析上面已保存并且符合条件的类型BeanDefinition对象。重头戏都在这个函数里面。绝对是核心函数。绝对值得投入时间弄懂流程。

spring 提供了很多工具类，不需要重复造轮子，总有你用得到的。ConfigurationClassUtils、ReflectionUtils、AnnotationConfigUtils、BeanUtils是不是看名字就能猜到怎么用了。

未完........