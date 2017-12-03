---
title: springMVC源码分析之HandlerMapping请求映射
date: 2016-09-16 18:52:01
tags: springMVC,HandlerMapping
---
在上一篇我们说了springMVC的初始化，作为springMVC启动的第一步完成的任务是初始化spring的IoC容器、初始化DispatchServlet的IoC容器，同时将webApplicationContext进行初始化，接下来就是strategy的初始化。这一系列初始化后接下来就是要完成根据url来映射到对应的具体方法并调用到最后返回页面。在接下的的两篇博文中我会分两篇来说说springMVC具体的映射分析。
#### Handler注册
在springMVC中最后一步是初始化strategy，就是初始化各个解析器如HandlerMapping和handlerAdapter。这是springMVC各配件初始化的入口。而initStrategies方法中initHandlerMapping是handlerMapping初始化的入口，该方法逻辑比较简单，首先从ApplicationContext中找到所有的HandlerMapping，如果没有则使用默认的HandlerMapping --DefaultAnnonationHandlerMapping、BeanNameUrlHandlerMapping。具体请看下面源码
<!--more-->
```
private void initHandlerMappings(ApplicationContext context) {
		this.handlerMappings = null;

		if (this.detectAllHandlerMappings) {
			//从ApplicationContext中获取所有HandlerMapping包括父Context
			Map<String, HandlerMapping> matchingBeans =
					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
			if (!matchingBeans.isEmpty()) {
				this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
				// We keep HandlerMappings in sorted order.
				AnnotationAwareOrderComparator.sort(this.handlerMappings);
			}
		}
		else {
			try {
				HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
				this.handlerMappings = Collections.singletonList(hm);
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Ignore, we'll add a default HandlerMapping later.
			}
		}

        //如果handlerMapping为空那么给其初始化默认的handlerMapping
		if (this.handlerMappings == null) {
			this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
			if (logger.isDebugEnabled()) {
				logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
			}
		}
	}
```
在该方法中一般如果没有特别配置基本上ApplicationContext中handlerMapping是null，所以都需要调用getDefaultStrategies方法，该方法会将默认的handlerMapping实例化并注册进ApplicationContext中。具体请看以下代码：
```
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
		String key = strategyInterface.getName();
		//defaultStrategies是一个map，获取的是DispatchServlet.properties的属性，在这里获取的是系统默认的String类型的handlerMapping
		String value = defaultStrategies.getProperty(key);
		if (value != null) {
			String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
			List<T> strategies = new ArrayList<T>(classNames.length);
			for (String className : classNames) {
				try {
				//通过反射实例化类
					Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
					Object strategy = createDefaultStrategy(context, clazz);
					strategies.add((T) strategy);
				}
				catch (ClassNotFoundException ex) {
					throw new BeanInitializationException(
							"Could not find DispatcherServlet's default strategy class [" + className +
									"] for interface [" + key + "]", ex);
				}
				catch (LinkageError err) {
					throw new BeanInitializationException(
							"Error loading DispatcherServlet's default strategy class [" + className +
									"] for interface [" + key + "]: problem with class file or dependent class", err);
				}
			}
			return strategies;
		}
		else {
			return new LinkedList<T>();
		}
	}
```
该方法就是通过反射将默认的handlerMapping进行实例化并且通过createDefaultStrategy将controller也就是这里handlerMapping中的handler注册到handlerMapping中。而handler的注册是在BeanUrlHandlerMapping的父类AbstractDetectingUrlHandlerMapping中detectHandler实现的，该方法主要逻辑是将所有spring IoC容器初始化的bean进行匹配，查找到该bean的urls不为空我们就认为他是一个handler并通过registerHandler方法进行注册。具体逻辑代码如下：
```
protected void detectHandlers() throws BeansException {
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for URL mappings in application context: " + getApplicationContext());
		}
		String[] beanNames = (this.detectHandlersInAncestorContexts ?
				BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
				getApplicationContext().getBeanNamesForType(Object.class));

		// Take any bean name that we can determine URLs for.
		for (String beanName : beanNames) {
		//获取handler的所有url定义
			String[] urls = determineUrlsForHandler(beanName);
			if (!ObjectUtils.isEmpty(urls)) {
				// URL paths found: Let's consider it a handler.
				registerHandler(urls, beanName);
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Rejected bean name '" + beanName + "': no URL paths identified");
				}
			}
		}
	}

```
最终在registerHandler方法中将url注册进一个HandlerMapping的私有属性handlerMap中，该handlerMap是一个以url为key、handler为value的linkedHashMap。经过以上几个步骤handler注册已经完成了，其中handlerMap是这里面的核心，在handlerMap中配置好了url请求和对应的handler映射，这为springMVC响应http请求做好了基本映射数据的准备。