---
title: Dubbo源码学习：对于异常的处理
date: 2024-12-22 14:19:13
tags: Dubbo
---



### 前言

在使用 Dubbo 进行服务间的调用时，如果 Provider 服务端抛出了特定异常，但在 Consumer 调用端可能拿到的并不是我们预期的异常类型，那Dubbo 对于服务端的异常是如何处理的呢？如果我们希望在调用端也能获取到对应的异常信息，可以怎么做呢？本文一起来看下。

<!--more-->

> 以下内容基于Dubbo 2.7.12版本



### 异常的包装

在服务端，Dubbo 是通过 **org.apache.dubbo.rpc.filter.ExceptionFilter** 进行处理的，当拿到调用结果后，会通过 **ExceptionFilter#onResponse** 对异常进行二次包装：

```Java
public void onResponse(Result appResponse, Invoker<?> invoker, Invocation invocation) {
    // 有异常的情况下执行
    if (appResponse.hasException() && GenericService.class != invoker.getInterface()) {
        try {
            Throwable exception = appResponse.getException();

            // 如果是受检异常，不处理
            // directly throw if it's checked exception
            if (!(exception instanceof RuntimeException) && (exception instanceof Exception)) {
                return;
            }
            // directly throw if the exception appears in the signature
            try {
                // 获取服务接口上的异常声明
                Method method = invoker.getInterface().getMethod(invocation.getMethodName(), invocation.getParameterTypes());
                Class<?>[] exceptionClasses = method.getExceptionTypes();
                for (Class<?> exceptionClass : exceptionClasses) {
                    // 如果是声明的异常，不处理
                    if (exception.getClass().equals(exceptionClass)) {
                        return;
                    }
                }
            } catch (NoSuchMethodException e) {
                return;
            }

            // for the exception not found in method's signature, print ERROR message in server's log.
            logger.error("Got unchecked and undeclared exception which called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + exception.getClass().getName() + ": " + exception.getMessage(), exception);

            // 异常和接口是不在同一个包下，如果是的话，不处理
            // directly throw if exception class and interface class are in the same jar file.
            String serviceFile = ReflectUtils.getCodeBase(invoker.getInterface());
            String exceptionFile = ReflectUtils.getCodeBase(exception.getClass());
            if (serviceFile == null || exceptionFile == null || serviceFile.equals(exceptionFile)) {
                return;
            }
            // 如果是 JDK 内置异常类型，不处理
            // directly throw if it's JDK exception
            String className = exception.getClass().getName();
            if (className.startsWith("java.") || className.startsWith("javax.")) {
                return;
            }
            // 如果是 RpcException，不处理
            // directly throw if it's dubbo exception
            if (exception instanceof RpcException) {
                return;
            }

            // otherwise, wrap with RuntimeException and throw back to the client
            // 其他异常，将其包装为 RunctionException 并返回
            appResponse.setException(new RuntimeException(StringUtils.toString(exception)));
        } catch (Throwable e) {
            logger.warn("Fail to ExceptionFilter when called by " + RpcContext.getContext().getRemoteHost() + ". service: " + invoker.getInterface().getName() + ", method: " + invocation.getMethodName() + ", exception: " + e.getClass().getName() + ": " + e.getMessage(), e);
        }
    }
}
```

从源码中，我们可以看到，Dubbo 对象异常的处理，主要分为几种情况：

1. 如果是受检异常，不处理，直接抛出；
2. 如果如果该异常在接口中声明了，不处理，直接抛出；
3. 如果异常和接口在同一个包下，不处理，直接抛出；
4. 如果是 JDK 内置的异常，不处理，直接抛出；
5. 如果是 RpcException，不处理，直接抛出；
6. 其他情况会将异常封装为 RuntimeException 。

总的来说，Dubbo 对于异常的处理，遵循了一个原则，如果该异常类在服务提供端和消费端都能够找到，那么该服务提供端异常会被原样抛出到消费端，否则，异常将封装为 RuntimeExcetion 后抛出。



### 小结

回到开头的问题，如果我们希望在服务消费端能够接收到提供端的异常，我们可以在接口上显式声明该异常，那么 Dubbo 就不会对该异常进行二次包装了。