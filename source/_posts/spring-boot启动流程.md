---
layout: 
title: spring boot启动流程
date: 2017-09-21 21:50:54
tags: spring boot,源码分析
---
### 引言
早在15年的时候就开始用spring boot进行开发了，然而一直就只是用用，并没有深入去了解spring boot是以什么原理怎样工作的，说来也惭愧。今天让我们从spring boot启动开始，深入了解一下spring boot的工作原理。

### 为什么用spring boot
在使用一个东西或者一个工具之前，我们总是会问自己，我为什么要用？用他能给我带来什么好处？
* 最大的好处就是spring boot遵从了java**约定大于配置**不用面对一大堆的配置文件，spring boot是根据你用的包来决定提供什么配置。
* 服务器以jar包的形式内嵌于项目中，对于微服务满天飞的情况，spring boot天生适合微服务架构，方便部署。
* 提供devtools从此改代码就需重启成为历史。

有优点就一定有缺点，缺点来源于优点优点来源于缺点（感觉在说哲学问题了哈哈哈）
* 正因为配置对开发者不透明，不看源码会不清楚spring boot如何进行诸如JDBC加载、事务管理等，出现错误也很难调错。
* 自动配置之后要自定义配置需编码javaConfig，需要了解这些配置类api。
* 版本迭代太快，新版本对老版本改动太多导致不兼容，比如1.3.5之前的springBootTest和1.4.0之后的springBootTest。

**只有合适的架构才是最好的架构**如果能接受spring boot这些缺点，spring boot确实是一个可以提高开发效率的不错的选择。

### 启动流程
扯了这么多，该上正题了，让我们来看看spring boot是怎样启动和启动做了哪些事情。

以下代码是spring boot项目标准的启动方式，使用注解@SpringBootApplication并且在main方法中调用SpringApplication的run方法，就可以完成。我们就从这个run方法开始看看spring boot的启动过程。
```
@SpringBootApplication
public class Application {


    public static void main(String[] args){
        SpringApplication.run(Application.class,args);
    }
}

```
我们进入run方法，可以看到最终是调用了 **new SpringApplication(sources).run(args);new SpringApplication(sources).run(args);** 这个方法，可以看到，springBoot的启动可以分为两个部分，第一部分：SpringApplication的实例化；第二部分：调用该实例运行run方法。我们先来看看这个SpringApplication的实例化过程。
```
private void initialize(Object[] sources) {
		if (sources != null && sources.length > 0) {
			this.sources.addAll(Arrays.asList(sources));
		}
		//判定是否为webEnvironment
		this.webEnvironment = deduceWebEnvironment();
		//实例化并加载所有可以加载的ApplicationContextInitializer
		setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class));
		//实例化并加载所有可以加载的ApplicationListener
		setListeners((Collection)
		getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
关键点在两个set方法上**setInitializers((Collection) getSpringFactoriesInstances(
				ApplicationContextInitializer.class))** 和 **setListeners((Collection)
		getSpringFactoriesInstances(ApplicationListener.class))** 这两个方法一毛一样，挑实例化ApplicationContextInitializer讲一讲。
```
private <T> Collection<? extends T> getSpringFactoriesInstances(Class<T> type,
			Class<?>[] parameterTypes, Object... args) {
			//拿到类加载器
		ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
		// Use names and ensure unique to protect against duplicates
		//使用loadFactoryNames方法载入所有的ApplicationContextInitializer的类全限定名
		Set<String> names = new LinkedHashSet<String>(
				SpringFactoriesLoader.loadFactoryNames(type, classLoader));
		//使用反射将所有的ApplicationContextInitializer实例化		
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes,
				classLoader, args, names);
		//排序		
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}
```
自动配置的关键就是这个 `getSpringFactoriesInstances`方法,确切的说是这个方法里的`loadFactoryNames`方法，浪我们看看这个`loadFactoryNames`方法干了啥，咋就能实现自动配置。
```
    public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {
        String factoryClassName = factoryClass.getName();

        try {
            Enumeration<URL> urls = classLoader != null?classLoader.getResources("META-INF/spring.factories"):ClassLoader.getSystemResources("META-INF/spring.factories");
            ArrayList result = new ArrayList();

            while(urls.hasMoreElements()) {
                URL url = (URL)urls.nextElement();
                Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
                String factoryClassNames = properties.getProperty(factoryClassName);
                result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));
            }

            return result;
        } catch (IOException var8) {
            throw new IllegalArgumentException("Unable to load [" + factoryClass.getName() + "] factories from location [" + "META-INF/spring.factories" + "]", var8);
        }
    }
```
可以看到这个方法就做了一件事，就是从`META-INF/spring.factories`这个路径取出所有"url"来，我们可以去到这个路径下看看到底是些啥？
```
# Initializers
org.springframework.context.ApplicationContextInitializer=\
org.springframework.boot.autoconfigure.logging.AutoConfigurationReportLoggingInitializer

# Application Listeners
org.springframework.context.ApplicationListener=\
org.springframework.boot.autoconfigure.BackgroundPreinitializer

```
这下大家都应该明白了，spring是通过将所有你加载的jar包中找到它需要的`ApplicationContextInitializer`来进行动态的配置的，只要你有用到特定的maven包，初始化的时候会找这个包下的`META-INF/spring.factories`的需要的类比如ApplicationContextInitializer进行实例化bean，你就可以用了，不需要任何配置。
说到这已经将所有SpringApplication实例化说完了，只是在加载完`ApplicationContextInitializer`和`ApplicationListener`这之后还有一步，就是找到启动类所在的位置并且设入属性mainApplicationClass中。

接下来让我们回到``new SpringApplication(sources).run(args)``方法来看看run方法是怎么run的。
```
public ConfigurableApplicationContext run(String... args) {
        //开启启动计时器，项目启动完会打印执行时间出来
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		FailureAnalyzers analyzers = null;
		configureHeadlessProperty();
		//获取SpringApplicationRunListener并启动监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(
					args);
			//环境变量的加载		
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			//启动后console的打印出来的一堆配置信息		
			Banner printedBanner = printBanner(environment);
			//终极大boss->ApplicationContext实例化
			context = createApplicationContext();
			analyzers = new FailureAnalyzers(context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			listeners.finished(context, null);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			return context;
		}
		catch (Throwable ex) {
			handleRunFailure(context, listeners, analyzers, ex);
			throw new IllegalStateException(ex);
		}
	}

```
从这个方法里面做的最关键的三件事情就是：

* 获取监听器并启动
* 加载环境变量，该环境变量包括system environment、classpath environment和用户自己加的application.properties
* 创建ApplicationContext

```
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
		context.getBeanFactory().registerSingleton("springApplicationArguments",
				applicationArguments);
		if (printedBanner != null) {
			context.getBeanFactory().registerSingleton("springBootBanner", printedBanner);
		}

		// Load the sources
		Set<Object> sources = getSources();
		Assert.notEmpty(sources, "Sources must not be empty");
		load(context, sources.toArray(new Object[sources.size()]));
		listeners.contextLoaded(context);
	}
```
前两点没什么好说的，重点说说第三个，创建ApplicationContext。创建applicationContext又分为几部：实例化applicationContext、prepareContext、refreshContext。实例化applicationContext会根据在之前我们说的`webEnvironment`这个属性判断是使用webContext类`AnnotationConfigEmbeddedWebApplicationContext`还是普通context类`AnnotationConfigApplicationContext`（在这里我们使用的是webContext为例）然后通过反射进行实例化。applicationContext实例化完了会进入prepareContext流程，这个prepareContext方法会加载之前准备好的environment进入context中，然后如果有`beanNameGenerator`和`resourceLoader`那么提前创建bean加载进applicationContext，但是一般这两个都是空的，所以直接进入applyInitializers方法，将之前实例化的所有initializers进行初始化，**所有的bean就是在这里进行bean的扫描和加载的**因这次讲的是启动过程，所以不再细讲。最后把创建好的applicationContext设置进入listener，prepareContext过程就结束了。最后是refreshContext，这个就和spring的bean加载过程一致了，bean的注入、beanFactory、postProcessBeanFactory等等，详情可以去看看spring bean的生命周期。

### 总结

spring boot 初始化内容还是很多的，但是总结起来就四点：
* 创建SpringApplication实例，判定环境，是web环境还是普通环境。加载所有需要用到的Initializers和Listeners，这里使用约定大于配置的理念揭开了自动配置的面纱。
* 加载环境变量，环境变量包括system environment、classpath environment、application environment（也就是我们自定义的application.properties配置文件）
* 创建SpringApplicationRunListeners 
* 创建ApplicationContext，设置装配context，在这里将所有的bean进行扫描最后在refreshContext的时候进行加载、注入。最终将装配好的context作为属性设置进SpringApplicationRunListeners，这就完成了一个spring boot项目的启动。