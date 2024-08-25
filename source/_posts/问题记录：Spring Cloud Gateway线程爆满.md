---
title: 问题记录：Spring Cloud Gateway线程爆满
date: 2024-08-25 19:52:15
tags: 问题记录

---

### 问题描述

某日告警群突发OOM日志告警：

```Java
java.lang.OutOfMemoryError: unable to create new native thread
```

开发、运维、运营相关人员立马被召集了起来，首先确认了业务未受到明显的影响。

<!--more-->



### 定位问题

我首先回想最近该服务更新的内容，但改动并不多，而且距离最近一次发版也有一定的时间了。

运维人员则确认告警的服务后，打算登入对应的服务器一探究竟，但无奈发现服务器也登录不进去，一时范了难。

回想告警的内容，是因为无法创建线程，合理猜测会不会是服务器的线程已经爆满了，导致物理机也受到影响。于是建议运维首先将部署在该物理机上的其他一个服务杀掉（考虑到保留线程信息），以释放掉部分线程出来，尝试后终于登进去了。

登录进去后，查看该服务器总线程数，达到了近3w，确认了问题是因为线程爆满导致，再针对告警的服务，使用**top**指定对应的进程：

```bash
top -Hp 进程id
```



但发现该服务占用线程数并不高，只有不到200个；于是针对部署在该物理机上的服务，逐个进行排查，最终发现是服务网关gateway导致，其线程数达到了2w多个。

知道是gateway导致后，则立马进入到该gateway所在pod，使用**jstack**输出该服务进程的堆栈信息：

```bash
jstack 1 > 1.log
```

导出堆栈信息后，重启gateway服务。

这边开始分析堆栈信息，发现有大量的命名为**boundedElastic-evictor-xxx**的线程，而且都处理**TIMED_WAITING**状态。

于是打开gateway服务的代码工程（spring-cloud-starter-gateway为3.0.0版本）：

尝试全局搜索**boundedElastic**关键字的类，发现了**BoundedElasticScheduler**这个类，查看其源代码，发现里面有这么一个线程工厂静态对象，其创建的线程命名与堆栈信息输出的一致：

```Jaava
static final ThreadFactory EVICTOR_FACTORY = r -> {
   Thread t = new Thread(r, Schedulers.BOUNDED_ELASTIC + "-evictor-" + EVICTOR_COUNTER.incrementAndGet());
   t.setDaemon(true);
   return t;
};
```

于是逐一查看其调用的位置，一路找一路排除，来到了**DefaultPartHttpMessageReader**：

```Java
private Scheduler blockingOperationScheduler = Schedulers.newBoundedElastic(Schedulers.DEFAULT_BOUNDED_ELASTIC_SIZE,
      Schedulers.DEFAULT_BOUNDED_ELASTIC_QUEUESIZE, IDENTIFIER, 60, true);
```

又一路找，来到了创建**DefaultPartHttpMessageReader**的来源**ServerDefaultCodecsImpl#extendTypedReaders**：

```Java
@Override
protected void extendTypedReaders(List<HttpMessageReader<?>> typedReaders) {
   if (this.multipartReader != null) {
      addCodec(typedReaders, this.multipartReader);
      return;
   }
   DefaultPartHttpMessageReader partReader = new DefaultPartHttpMessageReader();
   addCodec(typedReaders, partReader);
   addCodec(typedReaders, new MultipartHttpMessageReader(partReader));
}
```

继续排查代码，发现来到**DefaultServerWebExchange**这里：

```
private static Mono<MultiValueMap<String, String>> initFormData(ServerHttpRequest request,
      ServerCodecConfigurer configurer, String logPrefix) {

   try {
      MediaType contentType = request.getHeaders().getContentType();
      if (MediaType.APPLICATION_FORM_URLENCODED.isCompatibleWith(contentType)) {
         return ((HttpMessageReader<MultiValueMap<String, String>>) configurer.getReaders().stream()
               .filter(reader -> reader.canRead(FORM_DATA_TYPE, MediaType.APPLICATION_FORM_URLENCODED))
               .findFirst()
               .orElseThrow(() -> new IllegalStateException("No form data HttpMessageReader.")))
               .readMono(FORM_DATA_TYPE, request, Hints.from(Hints.LOG_PREFIX_HINT, logPrefix))
               .switchIfEmpty(EMPTY_FORM_DATA)
               .cache();
      }
   }
   catch (InvalidMediaTypeException ex) {
      // Ignore
   }
   return EMPTY_FORM_DATA;
}

@SuppressWarnings("unchecked")
private static Mono<MultiValueMap<String, Part>> initMultipartData(ServerHttpRequest request,
      ServerCodecConfigurer configurer, String logPrefix) {

   try {
      MediaType contentType = request.getHeaders().getContentType();
      if (MediaType.MULTIPART_FORM_DATA.isCompatibleWith(contentType)) {
         return ((HttpMessageReader<MultiValueMap<String, Part>>) configurer.getReaders().stream()
               .filter(reader -> reader.canRead(MULTIPART_DATA_TYPE, MediaType.MULTIPART_FORM_DATA))
               .findFirst()
               .orElseThrow(() -> new IllegalStateException("No multipart HttpMessageReader.")))
               .readMono(MULTIPART_DATA_TYPE, request, Hints.from(Hints.LOG_PREFIX_HINT, logPrefix))
               .switchIfEmpty(EMPTY_MULTIPART_DATA)
               .cache();
      }
   }
   catch (InvalidMediaTypeException ex) {
      // Ignore
   }
   return EMPTY_MULTIPART_DATA;
}
```

看了代码后，初步确认是在处理**application/x-www-form-urlencoded**或**multipart/form-data**请求时发生的。

于是在测试环境，找了个这种请求，使用jmeter简单跑了下，果然会导致线程爆满的问题，至此问题元凶终于确认。



### 解决方法

解决方法也比较简单，就是使用WebFlux的自定义接口，配置上默认的multipart解析器：

```Java
@Configuration
public class WebConfig implements WebFluxConfigurer {

    @Override
    public void configureHttpMessageCodecs(ServerCodecConfigurer configurer) {
        DefaultPartHttpMessageReader partReader = new DefaultPartHttpMessageReader();
        MultipartHttpMessageReader multipartReader = new MultipartHttpMessageReader(partReader);
        configurer.defaultCodecs().multipartReader(multipartReader);
    }

}
```

针对该类问题，后续可以针对每个pod单独限制线程的上限，避免服务之间的相互影响。

对于服务创建的线程数，可以设定阈值，并添加到告警项，避免业务受损问题才暴露出来。