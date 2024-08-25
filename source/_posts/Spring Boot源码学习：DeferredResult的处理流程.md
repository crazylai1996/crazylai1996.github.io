---
title: Spring Boot源码学习：DeferredResult的处理流程
date: 2024-04-20 15:02:00
tags: Spring Boot

---

### 前言

业务开发时，轮询可以被用于许多场景中，但把握轮询的频次不是一件容易的事情，频次过高会对服务端产生不小的压力，频次过低时，则无法保证实时性。而随着Servlet 3.0异步请求处理的特性支持，DeferredResult的出现很好的解决了这个问题。

<!--more-->

### DeferredResult是什么

DeferredResult是Spring基于Servlet 3.0的异步请求处理功能实现的，它可以迟早地释放Tomcat的请求线程，由业务线程去处理业务逻辑，处理完成再把结果返回到客户端，这可以使得服务端能够处理更多请求，以提升服务端的并发处理能力。

使用示例：

```Java
@RestController
@RequestMapping("/test")
@Slf4j
public class DeferredResultTestController {

    @GetMapping("/deferredResult")
    public DeferredResult<String> deferredResult(@RequestParam(defaultValue = "1000") long sleepMills) {
        //3s超时时间
        DeferredResult<String> deferredResult = new DeferredResult<>(3000L);
        deferredResult.onCompletion(() -> {
            log.info("请求完成");
        });
        deferredResult.onTimeout(() -> {
            deferredResult.setResult("请求超时了");
        });

        //使用新的业务线程去处理
        new Thread(() -> {
            try {
                Thread.sleep(sleepMills);
                deferredResult.setResult("处理成功");
            } catch (Exception e) {
                log.error("", e);
            }

        }).start();
        return deferredResult;
    }

}
```

### DeferredResult的处理流程

我们都知道，在Spring中，所有的请求都是由DispatcherServlet处理的，异步请求也不例外：

```Java
	protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
		HttpServletRequest processedRequest = request;
		HandlerExecutionChain mappedHandler = null;
		boolean multipartRequestParsed = false;
		//根据HttpServletRequest 创建或者从缓存中获取 WebAsyncManager 对象
		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

		try {
			ModelAndView mv = null;
			Exception dispatchException = null;

			try {
				processedRequest = checkMultipart(request);
				multipartRequestParsed = (processedRequest != request);

				// Determine handler for the current request.
				mappedHandler = getHandler(processedRequest);
				if (mappedHandler == null) {
					noHandlerFound(processedRequest, response);
					return;
				}

				// Determine handler adapter for the current request.
				HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

				// Process last-modified header, if supported by the handler.
				String method = request.getMethod();
				boolean isGet = "GET".equals(method);
				if (isGet || "HEAD".equals(method)) {
					long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
					if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
						return;
					}
				}

				if (!mappedHandler.applyPreHandle(processedRequest, response)) {
					return;
				}

				// Actually invoke the handler.
				mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

                //异步请求开始时，直接返回
				if (asyncManager.isConcurrentHandlingStarted()) {
					return;
				}

				applyDefaultViewName(processedRequest, mv);
				mappedHandler.applyPostHandle(processedRequest, response, mv);
			}
			//。。。
			processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
		}
		//...
	}

```

需要注意的是，这里而会获取/或新创建一个**WebAsyncManager**，该对象与当前请求绑定，用于对异步操作的管理，比如处理结果的传递、上下文的保存等。

对于返回结果为DeferredResult的Controller方法，Spring通过**DeferredResultMethodReturnValueHandler**来区分和处理：

```Java
public class DeferredResultMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
        
		Class<?> type = returnType.getParameterType();
		return (DeferredResult.class.isAssignableFrom(type) ||
				ListenableFuture.class.isAssignableFrom(type) ||
				CompletionStage.class.isAssignableFrom(type));
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		DeferredResult<?> result;
		
        //处理返回类型为DeferredResult、ListenableFuture、CompletionStage的结果
		if (returnValue instanceof DeferredResult) {
			result = (DeferredResult<?>) returnValue;
		}
		else if (returnValue instanceof ListenableFuture) {
			result = adaptListenableFuture((ListenableFuture<?>) returnValue);
		}
		else if (returnValue instanceof CompletionStage) {
			result = adaptCompletionStage((CompletionStage<?>) returnValue);
		}
		else {
			// Should not happen...
			throw new IllegalStateException("Unexpected return value type: " + returnValue);
		}

        //处理DeferredResult请求
		WebAsyncUtils.getAsyncManager(webRequest).startDeferredResultProcessing(result, mavContainer);
	}
	//...
}

```

来到**WebAsyncManager#startDeferredResultProcessing**方法：

```Java
	/**
	 * Start concurrent request processing and initialize the given
	 * {@link DeferredResult} with a {@link DeferredResultHandler} that saves
	 * the result and dispatches the request to resume processing of that
	 * result. The {@code AsyncWebRequest} is also updated with a completion
	 * handler that expires the {@code DeferredResult} and a timeout handler
	 * assuming the {@code DeferredResult} has a default timeout result.
	 * @param deferredResult the DeferredResult instance to initialize
	 * @param processingContext additional context to save that can be accessed
	 * via {@link #getConcurrentResultContext()}
	 * @throws Exception if concurrent processing failed to start
	 * @see #getConcurrentResult()
	 * @see #getConcurrentResultContext()
	 */
	public void startDeferredResultProcessing(
			final DeferredResult<?> deferredResult, Object... processingContext) throws Exception {

		Assert.notNull(deferredResult, "DeferredResult must not be null");
		Assert.state(this.asyncWebRequest != null, "AsyncWebRequest must not be null");

		Long timeout = deferredResult.getTimeoutValue();
		if (timeout != null) {
			this.asyncWebRequest.setTimeout(timeout);
		}

		List<DeferredResultProcessingInterceptor> interceptors = new ArrayList<>();
		interceptors.add(deferredResult.getInterceptor());
		interceptors.addAll(this.deferredResultInterceptors.values());
		interceptors.add(timeoutDeferredResultInterceptor);

		final DeferredResultInterceptorChain interceptorChain = new DeferredResultInterceptorChain(interceptors);

		this.asyncWebRequest.addTimeoutHandler(() -> {
			try {
				interceptorChain.triggerAfterTimeout(this.asyncWebRequest, deferredResult);
			}
			catch (Throwable ex) {
				setConcurrentResultAndDispatch(ex);
			}
		});

		this.asyncWebRequest.addErrorHandler(ex -> {
			try {
				if (!interceptorChain.triggerAfterError(this.asyncWebRequest, deferredResult, ex)) {
					return;
				}
				deferredResult.setErrorResult(ex);
			}
			catch (Throwable interceptorEx) {
				setConcurrentResultAndDispatch(interceptorEx);
			}
		});

		this.asyncWebRequest.addCompletionHandler(()
				-> interceptorChain.triggerAfterCompletion(this.asyncWebRequest, deferredResult));

		interceptorChain.applyBeforeConcurrentHandling(this.asyncWebRequest, deferredResult);
		//1. 开启异步处理
        startAsyncProcessing(processingContext);

		try {
			interceptorChain.applyPreProcess(this.asyncWebRequest, deferredResult);
			//2. 设置ResultHandler处理器
            deferredResult.setResultHandler(result -> {
				result = interceptorChain.applyPostProcess(this.asyncWebRequest, deferredResult, result);
				setConcurrentResultAndDispatch(result);
			});
		}
		catch (Throwable ex) {
			setConcurrentResultAndDispatch(ex);
		}
	}

```

从方法注释中，我们也可以看出，该方法：

1. 开启异步处理，将当前请求标记为异步请求，以便Tomcat能够识别；
2. 针对当前的DeferredResult，设置了一个结果处理器；

而当我们调用**DeferredResult#setResult**时，

```Java
	public boolean setResult(T result) {
		return setResultInternal(result);
	}

	private boolean setResultInternal(Object result) {
		if (isSetOrExpired()) {
			return false;
		}
		DeferredResultHandler resultHandlerToUse;
		synchronized (this) {
			if (isSetOrExpired()) {
				return false;
			}
			this.result = result;
			//取当前设置的ResultHandler
			resultHandlerToUse = this.resultHandler;
			if (resultHandlerToUse == null) {
				return true;
			}
			this.resultHandler = null;
		}
		//处理
		resultHandlerToUse.handleResult(result);
		return true;
	}
```

可以看到，实际调用的是在**WebAsyncManager#startDeferredResultProcessing**设置的进去的：

```Java
	private void setConcurrentResultAndDispatch(Object result) {
		synchronized (WebAsyncManager.this) {
			if (this.concurrentResult != RESULT_NONE) {
				return;
			}
            //设置结果
			this.concurrentResult = result;
		}

		if (this.asyncWebRequest.isAsyncComplete()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Async result set but request already complete: " + formatRequestUri());
			}
			return;
		}

		if (logger.isDebugEnabled()) {
			boolean isError = result instanceof Throwable;
			logger.debug("Async " + (isError ? "error" : "result set") + ", dispatch to " + formatRequestUri());
		}
        //将请求再次分发，该请求将重新进入DispatcherServlet的doDispatch方法进行处理
		this.asyncWebRequest.dispatch();
	}
```

再次进入**DispatcherServlet#doDispatch**方法，通过debug可以发现，asyncManager这里已经拿到了处理结果：

![image-20240420181952931](http://storage.laixiaoming.space/blog/image-20240420181952931.png)

而在后续通过**RequestMappingHandlerAdapter**调用具体的Controller方法时：

```Java
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		try {
			//省略...

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
			//当前异步请求是否有了结果
			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
                //替换原始反射方法，该方法返回最终结果
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		}
		finally {
			webRequest.requestCompleted();
		}
	}

```

可以看到，原来的Controller方法会被替换，替换后的invokeAndHandle将直接返回最终处理结果，返回类型也不再是DeferredResult，结果被返回给客户端。

### 小结

1.  在Spring中使用异步请求可以将返回值定义为**DeferredResult**；

2. Spring通过**DeferredResultMethodReturnValueHandler**针对返回类型为DeferredResult的Controller方法返回值进行特殊处理：

   ​	a. 开启异步请求；

   ​	b.设置结果回调处理器DeferredResultHandler； 

3. DeferredResult设置返回结果后，将触发DeferredResultHandler处理器的执行，该处理器将对原请求重新分发处理，并将最终结果 保存在了**WebAsyncManager**中，随后触发了**DispatcherServlet#doDispatch**的再次执行；

4. 第二次**DispatcherServlet#doDispatch**执行过程中，将通过**WebAsyncManager**拿到处理结果，将在后续替换掉原有Controller方法的调用，将最终结果返回给客户端。