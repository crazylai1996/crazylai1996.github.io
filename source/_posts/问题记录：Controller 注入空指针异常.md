---
title: 问题记录：Controller 注入空指针异常
date: 2025-06-29 22:09:19
tags: 问题记录

---

### 问题描述

在最近开发过程中，遇到了一个让人疑惑的问题，现有一个 Controller ，有且只有一个私有方法：

```Java
@RestController
@RequestMapping("/test")
@Api(tags = "测试")
public class TestPubController {

    @Resource
    private TestService testService;

    @GetMapping("/pri")
    @ApiOperation(value = "测试私有方法")
    private String testPrivate() {
        System.out.println("private");
        testService.say();
        return "private";
    }
}
```

此时，接口调用是没有任何问题的。但当在这个 Controller 方法中增加了一个公有方法后：

```Java
    @GetMapping("/pub")
    @ApiOperation(value = "测试公共方法")
    public String testPublic(@RequestParam(required = false) Integer p) {
        System.out.println("public");
        testService.say();
        return "public";
    }
```

问题出现了，原有的 "/test/pri" 接口在调用 "testService#say" 方法时报了“空指针异常”，而新增的 "/test/pub" 却能正常使用。这乍一看，着实有点诡异，于是便打开 IDEA 准备 debug 一波。



### 定位问题

首先是 "test/pri" 接口：

![image-20250629222842692](http://storage.laixiaoming.space/blog/image-20250629222842692.png)

可以看到 testService 确实是空的。

再看下 "test/pub" 接口，注入正常：

![image-20250629223030460](http://storage.laixiaoming.space/blog/image-20250629223030460.png)

为什么会出现这种情况呢？

仔细一看会发现，这两个地方的 this 对象并不是同一个！在 testPrivate 方法中，this 指向的是一个 "TestPubController" 的 CGLIB 代理对象，而在 testPublic 方法中，this 则指向的是一个 "TestPubController" 对象。

了解过 Spring 代理的可能已经知道端倪了，TestPubController 是被代理过的，在进入 testPrivate 方法时，当前对象是代理对象，而进入 testPublic 方法时，当前对象是目标对象，而目标对象才是完成了自动注入流程的。

那为什么 testPrivate 方法中不会调用目标对象呢？

使用 arthas 查看这个代理类的源码中，这两个方法的具体区别：

```Java
public class TestPubController$$EnhancerBySpringCGLIB$$ee1c3a5f
        extends TestPubController
        implements SpringProxy,
        Advised,
        Factory {
        // 省略...
    public final String testPublic(Integer n) {
        MethodInterceptor methodInterceptor = this.CGLIB$CALLBACK_0;
        if (methodInterceptor == null) {
            TestPubController$$EnhancerBySpringCGLIB$$ee1c3a5f.CGLIB$BIND_CALLBACKS(this);
            methodInterceptor = this.CGLIB$CALLBACK_0;
        }
        if (methodInterceptor != null) {
            return (String) methodInterceptor.intercept(this, CGLIB$testPublic$0$Method, new Object[]{n}, CGLIB$testPublic$0$Proxy);
        }
        return super.testPublic(n);
    }
    // 省略...
}
        
```

发现代理类中只代理实现了 testPublic 方法，此时出现这个问题的原因便清晰了。

1. 该项目中有一个切面类，针对所有 Controller 的方法增加了日志打印的处理逻辑；
2. 当 Controller 中只有私有方法时，CGLIB 不会生成该 Controller 的代理类，此时不会有任何问题；
3. 当  Controller 中存在仅有方法时，CGLIB 将会生成该 Controller 的代理类，此时对公有方法的调用将转化为目标对象的调用；但另一方面，又不会代理私有方法，此时问题便出现了。

所以在项目开发中，对于 Controller 中的接口方法，由于并不是所有的框架都会忽略方法可见性，建议还是设置为公有，以此可以避免一些其他不可预知的问题。