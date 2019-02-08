---
title: SpringBoot启动流程解析
categories: Spring
tags:
  - Spring
  - SpringBoot
date: 2019-02-06 20:00:00
---


SpringBoot是由Pivotal团队提供的全新框架，其设计目的是用来简化新Spring应用的初始搭建以及开发过程。

下面通过源码来了解下SpringBoot的启动流程。(展示代码基于`2.1.1.RELEASE`)

```java
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}
}
```

Spring Boot的启动入口在`SpringApplication.run()`方法。

```java
	public static ConfigurableApplicationContext run(Class<?>[] primarySources,
			String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```

首先创建一个SpringApplication，然后执行run方法进行项目初始化。

### 初始化SpringApplication

```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
		this.webApplicationType = WebApplicationType.deduceFromClasspath();		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```

首先根据项目中是否存在web容器相关的类来判断当前项目是否是一个web项目，然后初始化`ApplicationContextInitializer`和`ApplicationListener`，这两组类的初始化是通过`SpringFactoriesLoader`来进行初始化的，先看下`SpringFactoriesLoader`。

```java
public final class SpringFactoriesLoader {
	public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

	private static final Map<ClassLoader, MultiValueMap<String, String>> cache = new ConcurrentReferenceHashMap<>();

	public static List<String> loadFactoryNames(Class<?> factoryClass, @Nullable ClassLoader classLoader) {
		String factoryClassName = factoryClass.getName();
		return loadSpringFactories(classLoader).getOrDefault(factoryClassName, Collections.emptyList());
	}
	
	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}
		try {
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryClassName = ((String) entry.getKey()).trim();
					for (String factoryName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryClassName, factoryName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}
}
```

`SpringFactoriesLoader`的工作原理是在CLASSPATH下每个Jar包中的`META-INF/spring.factories`配置文件中找到对应类型的处理类。

在初始化SpringApplication的时候使用`SpringFactoriesLoader`获取所有`spring.factories`文件中的`ApplicationContextInitializer`和`ApplicationListener`的处理类。

`ApplicationContextInitializer`用于在容器`refresh`之前对容器进行扩展。

`ApplicationListener`是SpringBoot的监听器，用于监听SpringBoot应用启动过程中各个环节对应的事件(ApplicationEvent)，便于开发人员在特定启动环节进行定制化扩展。


### 执行SpringApplication的run方法

```java
	public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
		// 1
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			// 2
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			// 3
			Banner printedBanner = printBanner(environment);
			// 4
			context = createApplicationContext();
			// 5
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			// 6
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			// 7
			refreshContext(context);
			// 8
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			// 9
			listeners.started(context);
			// 10
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			// 11
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```

`run`方法是SpringBoot整个启动过程的核心流程。

1. 获取`SpringApplicationRunListener`，`SpringApplicationRunListener`的默认实现是`EventPublishingRunListener`，其作用是用来发布SpringBoot应用启动过程中各个环节对应的事件(ApplicationEvent)，由`ApplicationListener`来监听处理，`starting`方法将会发布`ApplicationStartingEvent`事件。

2. 创建Environment，初始化配置文件(profile)和属性文件(properties)，创建完成后会发布`ApplicationEnvironmentPreparedEvent`事件。

3. 输出Banner，SpringBoot启动时的面板样式。

4. 创建Spring容器ConfigurableApplicationContext，如果是web项目则是`AnnotationConfigServletWebServerApplicationContext`, 如果不是web项目则是`AnnotationConfigApplicationContext`。

5. 从`SpringFactoriesLoader`加载`SpringBootExceptionReporter`用于处理启动异常，默认为`FailureAnalyzers`。

6. 执行`prepareContext`方法，在Spring容器初始化前进行预处理。

	```java
		private void prepareContext(ConfigurableApplicationContext context,
			ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments, Banner printedBanner) {
			context.setEnvironment(environment);
			postProcessApplicationContext(context);
			applyInitializers(context);
			listeners.contextPrepared(context);
			if (this.logStartupInfo) {
				logStartupInfo(context.getParent() == null);
				logStartupProfileInfo(context);
			}
			// Add boot specific singleton beans
			ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
			beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
			if (printedBanner != null) {
				beanFactory.registerSingleton("springBootBanner", printedBanner);
			}
			if (beanFactory instanceof DefaultListableBeanFactory) {
				((DefaultListableBeanFactory) beanFactory)
						.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
			}
			// Load the sources
			Set<Object> sources = getAllSources();
			Assert.notEmpty(sources, "Sources must not be empty");
			load(context, sources.toArray(new Object[0]));
			listeners.contextLoaded(context);
		}
	```
	* `applyInitializer`方法用于获取所有的`ApplicationContextInitializer`，执行其`initialize`方法在容器启动前进行扩展。
	* `listeners.contextPrepared(context)`方法用于发布`ApplicationContextInitializedEvent`事件。
	* `load`方法用于解析要加载的bean成为BeanDefinition注册到容器中，这里只加载了项目启动类，也就是`@SpringBootApplication`注解所在的bean。
	* `listeners.contextLoaded(context)`方法用于发布`ApplicationPreparedEvent`事件。

7. 执行`refreshContext`方法，初始化Spring容器。

	```java
		private void refreshContext(ConfigurableApplicationContext context) {
			refresh(context);
			if (this.registerShutdownHook) {
				try {
					context.registerShutdownHook();
				}
				catch (AccessControlException ex) {
					// Not allowed in some environments.
				}
			}
		}
		protected void refresh(ApplicationContext applicationContext) {
			Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
			((AbstractApplicationContext) applicationContext).refresh();
		}
	```
	该方法实际就是执行Spring容器的refresh方法进行容器的初始化，并注册了钩子用于项目关闭时，发布`ContextClosedEvent`事件，然后进行容器销毁操作。
	
	下面是Spring容器的初始化流程，SpringBoot基于Spring容器进行了定制化扩展，主要通过继承`AbstractApplicationContext`，以及使用Spring的扩展点`BeanFactoryPostProcessor`来对Spring容器进行定制扩展，下面看下主要的变化点：
	
	```java
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
	* `obtainFreshBeanFactory()`方法用于获取容器的BeanFactory，常规Spring项目的容器实现自`AbstractRefreshableApplicationContext`，在这一步会解析xml文件，将待加载的bean创建成BeanDefinition注册到容器中；而在SpringBoot项目中则实现自`GenericApplicationContext`，由于SpringBoot项目没有xml文件，所以这里没有初始化BeanDefinition，SpringBoot解析加载bean成为BeanDefinition是通过`BeanFactoryPostProcessor`扩展点进行的。在SpringBoot容器创建的时候，会根据是否是web项目创建`AnnotationConfigServletWebServerApplicationContext`或`AnnotationConfigApplicationContext`，这两种容器在创建时会通过`AnnotationConfigUtils.registerAnnotationConfigProcessors`注册一个beanName为`org.springframework.context.annotation.internalConfigurationAnnotationProcessor`的`BeanFactoryPostProcessor`，具体实现类为`ConfigurationClassPostProcessor`，这个类是SpringBoot初始化bean的核心类。

	* `invokeBeanFactoryPostProcessors(beanFactory)`方法用于执行处理`BeanFactoryPostProcessor`类型扩展点，该扩展点用于在容器实例化对象之前，对注册到容器的BeanDefinition所保存的信息做一些额外的操作，比如修改bean定义的某些属性或者增加其他信息等。这里将会执行`ConfigurationClassPostProcessor`。
	
		```java
			protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {
				PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
		
				... ...
				
			}
			final class PostProcessorRegistrationDelegate {
				public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {
		
					... ...
					
					List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
		
					String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
					for (String ppName : postProcessorNames) {
						if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
							currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
							processedBeans.add(ppName);
						}
					}
					sortPostProcessors(currentRegistryProcessors, beanFactory);
					registryProcessors.addAll(currentRegistryProcessors);
					invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
					currentRegistryProcessors.clear();
		
					... ...
						
				}
			}
		```
		这里获取的`BeanDefinitionRegistryPostProcessor`实现类就是`ConfigurationClassPostProcessor`，执行其`postProcessBeanDefinitionRegistry`方法进行处理。
		
		```java
			public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
				... ...
				processConfigBeanDefinitions(registry);
			}
			public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
			
				... ...
			
				// Parse each @Configuration class
				ConfigurationClassParser parser = new ConfigurationClassParser(
						this.metadataReaderFactory, this.problemReporter, this.environment,
						this.resourceLoader, this.componentScanBeanNameGenerator, registry);
			
				parser.parse(candidates);
				parser.validate();
				
				Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
				configClasses.removeAll(alreadyParsed);
	
				// Read the model and create bean definitions based on its content
				if (this.reader == null) {
					this.reader = new ConfigurationClassBeanDefinitionReader(
							registry, this.sourceExtractor, this.resourceLoader, this.environment,
							this.importBeanNameGenerator, parser.getImportRegistry());
				}
				this.reader.loadBeanDefinitions(configClasses);
				
				... ...
					
			}
		```
		`ConfigurationClassPostProcessor`内部功能主要是生成`ConfigurationClassParser`，执行其`parse`方法，用来解析类上的各种注解，执行注解对应的处理逻辑，生成配置信息，最后通过`reader.loadBeanDefinitions(configClasses)`方法解析成BeanDefinitions，这里以SpringBoot的启动类，也就是`@SpringBootApplication`注解所在的类为入口进行解析。
		
		```java
			public void parse(Set<BeanDefinitionHolder> configCandidates) {
				for (BeanDefinitionHolder holder : configCandidates) {
					BeanDefinition bd = holder.getBeanDefinition();
					try {
						if (bd instanceof AnnotatedBeanDefinition) {
							parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
						}
						else if (bd instanceof AbstractBeanDefinition && ((AbstractBeanDefinition) bd).hasBeanClass()) {
							parse(((AbstractBeanDefinition) bd).getBeanClass(), holder.getBeanName());
						}
						else {
							parse(bd.getBeanClassName(), holder.getBeanName());
						}
					}
					catch (BeanDefinitionStoreException ex) {
						throw ex;
					}
					catch (Throwable ex) {
						throw new BeanDefinitionStoreException(
								"Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
					}
				}
		
				this.deferredImportSelectorHandler.process();
			}
		```
		`parse`方法用于解析各种注解：`@ComponentScan`、`@ComponentScans`、`@Configuration`、`@Component`、`@Bean`、`@Import`、`@ImportResource`、`@PropertySource`。
		
		`deferredImportSelectorHandler.process()`方法用来处理自动配置功能。主要通过`AutoConfigurationImportSelector`来处理，该类是通过`@SpringBootApplication`注解上的`@Import`注解导入的，`AutoConfigurationImportSelector`通过`SpringFactoriesLoader`获取所有`spring.factories`文件中的`EnableAutoConfiguration`类型自动配置类，根据类上的`@Conditional`系列注解，过滤掉不需要加载的自动配置类，将剩余需要加载的自动配置类进行初始化解析。
		
	* `onRefresh()`方法用于创建web容器，如果是web项目，则容器继承于`ServletWebServerApplicationContext`，将会在这一步创建web容器。
	
		```java
			protected void onRefresh() {
				super.onRefresh();
				try {
					createWebServer();
				}
				catch (Throwable ex) {
					throw new ApplicationContextException("Unable to start web server", ex);
				}
			}
		```

	* `finishRefresh()`方法用于启动web容器，如果是web项目，则容器继承于`ServletWebServerApplicationContext`，将会在这一步启动web容器，并发布`ServletWebServerInitializedEvent `事件。
	
		```java
			protected void finishRefresh() {
				super.finishRefresh();
				WebServer webServer = startWebServer();
				if (webServer != null) {
					publishEvent(new ServletWebServerInitializedEvent(webServer, this));
				}
			}
		```

8. 执行`afterRefresh`来在Spring容器初始化完成后进行扩展，默认没有实现。

9. 发布`ApplicationStartedEvent`事件。

10. 获取容器中的`ApplicationRunner`和`CommandLineRunner`，执行其`run`方法，用于在启动完成前进行扩展。

11. 发布`ApplicationReadyEvent`事件。


至此SpringBoot启动完成。从整个启动流程来看，SpringBoot的核心就是在Spring容器初始化的基础上，通过Spring的扩展点进行了定制化扩展，并进行了一层包装。主要通过继承`AbstractApplicationContext`，以及使用Spring的扩展点`BeanFactoryPostProcessor`进行扩展，并新增了`ApplicationContextInitializer`和`ApplicationListener`进行扩展。


## 参考资料:

[给你一份Spring Boot知识清单](https://www.jianshu.com/p/83693d3d0a65)

