---
layout: post
title: " 戒懒系列（三）  ConfigurationClassPostProcessor的使命"
subtitle: ""
author: "bloomy"
header-img: "img/post-bg-css.jpg"
header-img-credit: "@WebdesignerDepot"
header-img-credit-href: "medium.com/@WebdesignerDepot/poll-should-css-become-more-like-a-programming-language-c74eb26a4270"
date:       2018-12-20
header-mask: 0.4
tags:

  - spring
  - 框架
---

>  此处分析居于spring 3.2.18源代码。spring boot分析居于spring 5.x版本。

上一篇说的流程，多调试几遍，开始parse.parse的分析。

~~~java
//ConfigurationClassParser.java
public void parse(String className, String beanName) throws IOException {
		MetadataReader reader = this.metadataReaderFactory.getMetadataReader(className);
		processConfigurationClass(new ConfigurationClass(reader, beanName));
	}

protected void processConfigurationClass(ConfigurationClass configClass) throws IOException {
    ......
    ......
    do {
			metadata = doProcessConfigurationClass(configClass, metadata);
		}
		while (metadata != null);
}
~~~

直接看doProcessConfigurationClass的流程，看到了@PropertySource注解，要不要做点什么呢？

~~~java
//ConfigurationClassParser.java
protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) {
    // Process any @PropertySource annotations
		AnnotationAttributes propertySource = MetadataUtils.attributesFor(metadata,
				org.springframework.context.annotation.PropertySource.class);
		if (propertySource != null) {
			processPropertySource(propertySource);
		}
    .......
    .......
    .......
}
~~~

继续跟进到processPropertySource函数里面。还记得第一篇文章说的<a href= "/2018/12/19/toulan-springframwork2/" target="_blank">Enviroment 戳我</a>。

this.environment.resolveRequiredPlaceholders函数通过参数key值，在StandardEnviroment##propertySources##propertySourcesList数组里面查找对应的value值。自定义的配置信息已经被解析后存储到StandardEnviroment#propertySources对象里了。记住this.propertySources这个变量，最终会被添加到StandardEnviroment##propertySources##propertySourcesList里面。以后所有解析配置的操作都通过environment.resolveRequiredPlaceholders处理。

~~~java
//ConfigurationClassParser.java
private void processPropertySource(AnnotationAttributes propertySource) throws IOException {
		String name = propertySource.getString("name");
		String[] locations = propertySource.getStringArray("value");
		int locationCount = locations.length;
		if (locationCount == 0) {
			throw new IllegalArgumentException("At least one @PropertySource(value) location is required");
		}
		for (int i = 0; i < locationCount; i++) {
			locations[i] = this.environment.resolveRequiredPlaceholders(locations[i]);
		}
		ClassLoader classLoader = this.resourceLoader.getClassLoader();
		if (!StringUtils.hasText(name)) {
			for (String location : locations) {
				this.propertySources.push(new ResourcePropertySource(location, classLoader));
			}
		}
		else {
			if (locationCount == 1) {
				this.propertySources.push(new ResourcePropertySource(name, locations[0], classLoader));
			}
			else {
				CompositePropertySource ps = new CompositePropertySource(name);
				for (int i = locations.length - 1; i >= 0; i--) {
					ps.addPropertySource(new ResourcePropertySource(locations[i], classLoader));
				}
				this.propertySources.push(ps);
			}
		}
	}
~~~

doProcessConfigurationClass函数做完@PropertySource注解的处理，第10行代码继续处理@ComponentScan注解的BeanDefinition对象。

~~~java
//ConfigurationClassParser.java

this.componentScanParser = new ComponentScanAnnotationParser(
				resourceLoader, environment, componentScanBeanNameGenerator, registry);

////////////////////////////////////////////

protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) {
    .....
    .....
    // Process any @ComponentScan annotations
		AnnotationAttributes componentScan = MetadataUtils.attributesFor(metadata, ComponentScan.class);
		if (componentScan != null) {
			// The config class is annotated with @ComponentScan -> perform the scan immediately
			Set<BeanDefinitionHolder> scannedBeanDefinitions =
					this.componentScanParser.parse(componentScan, metadata.getClassName());

			// Check the set of scanned definitions for any further config classes and parse recursively if necessary
			for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
				if (ConfigurationClassUtils.checkConfigurationClassCandidate(holder.getBeanDefinition(), this.metadataReaderFactory)) {
					this.parse(holder.getBeanDefinition().getBeanClassName(), holder.getBeanName());
				}
			}
		}
    ......
    ......
}
~~~

doProcessConfigurationClass函数执行完@ComponentScan注解解析后，继续处理@Import注解。

~~~java
//ConfigurationClassParser.java
protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) {
    .....
    .....
// Process any @Import annotations
		Set<Object> imports = new LinkedHashSet<Object>();
		Set<String> visited = new LinkedHashSet<String>();
		collectImports(metadata, imports, visited);
		if (!imports.isEmpty()) {
			processImport(configClass, metadata, imports, true);
            
    .....
    .....
}
~~~

collectImports函数递归的解析@Import注解，包括注解上的注解都会被解析。说起来有点拗口了。说白了就是参数中传递一个metadata，会把这个metadata上的注解和它注解上面的注解都解析。这里用深度优先算法，并且是递归方式。能换成广度优先算法 + 循环的方式么？spring 很多地方都使用了递归的方式。

~~~java
//ConfigurationClassParser.java

private void collectImports(AnnotationMetadata metadata, Set<Object> imports, Set<String> visited) {
    String className = metadata.getClassName();
    
    StandardAnnotationMetadata stdMetadata = (StandardAnnotationMetadata) metadata;
                    for (Annotation ann : stdMetadata.getIntrospectedClass().getAnnotations()) {
                        if (!ann.annotationType().getName().startsWith("java") && !(ann instanceof Import)) {
                            collectImports(new StandardAnnotationMetadata(ann.annotationType()), imports, visited);
                        }
                    }
                    Map<String, Object> attributes = stdMetadata.getAnnotationAttributes(Import.class.getName(), false);
                    if (attributes != null) {
                        Class<?>[] value = (Class<?>[]) attributes.get("value");
                        if (!ObjectUtils.isEmpty(value)) {
                            for (Class<?> importedClass : value) {
                                // Catch duplicate from ASM-based parsing...
                                imports.remove(importedClass.getName());
                                imports.add(importedClass);
                            }
                        }
                    }
}
~~~

继续看processImport函数流程，分为三步，第一步把collectImports函数解析出来所有@Import注解Class<?>[] value()的信息。递归解析classesToImport参数中每个包含@ImportSelector，@ImportBeanDefinitionRegistrar的注解。并且还会递归解析@ImportSelector注解引入的类，从processConfigurationClass函数开始。保证被新引入的类的注解都能被注册到beanDefinitionMap里面。如果要自己扩展spring，@Import注解是少不了的。

~~~java
//ConfigurationClassParser.java
private void processImport(ConfigurationClass configClass, AnnotationMetadata metadata,
			Collection<?> classesToImport, boolean checkForCircularImports) {
    for (Object candidate : classesToImport) {
                        Object candidateToCheck = (candidate instanceof Class ? (Class) candidate :
                                this.metadataReaderFactory.getMetadataReader((String) candidate));
                        if (checkAssignability(ImportSelector.class, candidateToCheck)) {
                            // Candidate class is an ImportSelector -> delegate to it to determine imports
                            Class<?> candidateClass = (candidate instanceof Class ? (Class) candidate :
                                    this.resourceLoader.getClassLoader().loadClass((String) candidate));
                            ImportSelector selector = BeanUtils.instantiateClass(candidateClass, ImportSelector.class);
                            processImport(configClass, metadata, Arrays.asList(selector.selectImports(metadata)), false);
                        }
                        else if (checkAssignability(ImportBeanDefinitionRegistrar.class, candidateToCheck)) {
                            // Candidate class is an ImportBeanDefinitionRegistrar ->
                            // delegate to it to register additional bean definitions
                            Class<?> candidateClass = (candidate instanceof Class ? (Class) candidate :
                                    this.resourceLoader.getClassLoader().loadClass((String) candidate));
                            ImportBeanDefinitionRegistrar registrar =
                                    BeanUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class);
                            invokeAwareMethods(registrar);
                            registrar.registerBeanDefinitions(metadata, this.registry);
                        }
                        else {
                            // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                            // process it as a @Configuration class
                            this.importStack.registerImport(metadata,
                                    (candidate instanceof Class ? ((Class) candidate).getName() : (String) candidate));
                            processConfigurationClass(candidateToCheck instanceof Class ?
                                    new ConfigurationClass((Class) candidateToCheck, true) :
                                    new ConfigurationClass((MetadataReader) candidateToCheck, true));
                        }
                    }
}
~~~

doProcessConfigurationClass流程，继续处理@ImportResource、@Bean注解的解析。

~~~java
//ConfigurationClassParser.java
protected AnnotationMetadata doProcessConfigurationClass(ConfigurationClass configClass, AnnotationMetadata metadata) {
  	.................................
    .................................
    // Process any @ImportResource annotations
            if (metadata.isAnnotated(ImportResource.class.getName())) {
                AnnotationAttributes importResource = MetadataUtils.attributesFor(metadata, ImportResource.class);
                String[] resources = importResource.getStringArray("value");
                Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
                for (String resource : resources) {
                    String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
                    configClass.addImportedResource(resolvedResource, readerClass);
                }
            }
   .................................
   .................................
       
       // Process individual @Bean methods
Set<MethodMetadata> beanMethods = metadata.getAnnotatedMethods(Bean.class.getName());
		for (MethodMetadata methodMetadata : beanMethods) {
			configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
		}
}
~~~

结束ConfigurationClassPostProcessor类的分析。大家抽空调试一遍源代码，理清这个类的作用。把这个类处理@Configuration、@Import等注解和@ImportBeanDefinitionRegistrar, ImportSelector定义的接口的流程弄明白了，对spring扩展就基本入门。还有很多流程发现用文字实在是不好描述，语言功底还是薄弱。一定要调试代码。看spring 源码，深度优先、递归算法要有所了解。带着问题调试关键的代码。比如文章里一直提到DefaultListableBeanFactory#beanDefinitionMap变量，是干什么用的。BeanFactoryPostProcessor、BeanPostProcessor、InstantiationAwareBeanPostProcessor、DestructionAwareBeanPostProcessor、SmartInstantiationAwareBeanPostProcessor等接口定义的方法是在什么地方被调用？

