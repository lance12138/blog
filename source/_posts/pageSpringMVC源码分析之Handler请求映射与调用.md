---
title: SpringMVC源码分析之Handler请求映射与调用
date: 2016-10-30 21:50:42
tags: SpringMVC
---
在前两篇说了springMVC的初始化和HandlerMapping，接下来我们用最后一篇说说handler被映射和调用的过程。在说过初始化和HandlerMapping的注册过程后，springMVC的映射和调用就很简单了。
#### 请求映射Handler
当一个请求发出后，比如我们发出一个"http://localhost:8080/pay/payment"的请求，springMVC会先将路径封装到HttpServletRequest中，而spring容器会通过getHandler方法来通过路径找到上一篇说的已经注册好的HandlerMapping。请看如下代码：
```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// 找到当前请求的handler.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null || mappedHandler.getHandler() == null) {
				//未找到handler抛出异常
					noHandlerFound(processedRequest, response);
					return;
				}

				// 找到handler之后，为该handler匹配正确的适配器
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

................截取部分代码
```
<!-- more -->
通过request来匹配当前已经注册了的handlerMapping，再通过handlerMapping来获取handler。
```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
		for (HandlerMapping hm : this.handlerMappings) {
			if (logger.isTraceEnabled()) {
				logger.trace(
						"Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
			}
			HandlerExecutionChain handler = hm.getHandler(request);
			if (handler != null) {
				return handler;
			}
		}
		return null;
```
从下面的代码来看，spring容器是通过request封装的请求路径lookupPath来找到handler的，并且也是通过lookupPath来找到handler的handlerMethod的，代码就不重复贴了。
```
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
		String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
		if (logger.isDebugEnabled()) {
			logger.debug("Looking up handler method for path " + lookupPath);
		}
		this.mappingRegistry.acquireReadLock();
		try {
			HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
			if (logger.isDebugEnabled()) {
				if (handlerMethod != null) {
					logger.debug("Returning handler method [" + handlerMethod + "]");
				}
				else {
					logger.debug("Did not find handler method for [" + lookupPath + "]");
				}
			}
			return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
		}
		finally {
			this.mappingRegistry.releaseReadLock();
		}
	}

```
在找到handler之后返回的并不是仅仅是handler，而是HandlerExecutionChain，这一招应该是借鉴了struts的优良责任链模式，在handler的处理前后还有可能有多个处理逻辑，这其中包括各种拦截器或者其他的handler处理。
#### handler调用
handler获取之后接下来就是获取对应的adapter，在这使用的是适配器模式，在springMVC中有各种类型的controller bean，你比如使用注解的不使用注解的。为了方便扩展，spring容器使用适配器模式来对应不同controller调用，也就是通过实现adapter接口同时组合该handler来实现不同handler使用统一的调用方式。获取adapter的方式和获取handler的方式差不多，这里就不展开讲。
接下来也就是最后的一步--调用handler。调用handler主要三步，第一获取解析器，第二步获取具体方法，第三部反射调用方法并返回结果封装到ModelAndView中最终返回，详情看下面代码：
```

	protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
        //获取方法解析器
		ServletHandlerMethodResolver methodResolver = getMethodResolver(handler);
		//通过方法解析器通过request的url来调用具体方法
		Method handlerMethod = methodResolver.resolveHandlerMethod(request);
		ServletHandlerMethodInvoker methodInvoker = new ServletHandlerMethodInvoker(methodResolver);
		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		ExtendedModelMap implicitModel = new BindingAwareModelMap();
        //调用具体方法返回结果
		Object result = methodInvoker.invokeHandlerMethod(handlerMethod, handler, webRequest, implicitModel);
		//将结果封装到modelAndView中结果返回
		ModelAndView mav =
				methodInvoker.getModelAndView(handlerMethod, handler.getClass(), result, implicitModel, webRequest);
		methodInvoker.updateModelAttributes(handler, (mav != null ? mav.getModel() : null), implicitModel, webRequest);
		return mav;
	}
```
总的来说，springMVC方法调用还是比较简单的，通过请求的mapping路径匹配具体的handler，也就是我们说的controller，找到handler还要将找到对应的适配器，最后才是调用具体方法返回已经被封装成ModelAndView的结果。