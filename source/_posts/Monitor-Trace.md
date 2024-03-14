---
title: 分布式链路追踪
date: 2021-03-14 14:53:22
categories: [Monitor]
tags: [IT, trace]
---
### 背景：
分布式系统往往会进行模块的拆分，如：基础模块、网关模块、订单模块、库存模块等，那么当一个请求耗时过长时，我们怎么去更直观地定位问题所在呢？请求在每个模块的执行过程是怎样的？模块与模块之间是怎样的调用顺序？请求究竟卡在哪一步？
从而引发一个思考：如何能清晰的描述一个`请求的生命周期`，帮助我们快速地定位问题。

### 基本思路：
给每个请求生成一个唯一标识的traceId，在日志打印的加上这个traceId，模块/服务间调用时将traceId往下传，直至整个请求处理完成返回；

<!--more-->

### 做法：（java实现）
- 模块内部对controller做AOP切面拦截，生成traceId存入线程上下文，修改日志包在日志输出时打印threadLocal的traceId
``` java
@Aspect
@Component
public class ControllerAspect {

  @Pointcut("execution(* com.xxx.xxx.controller..*(..))")
  public void controllerLog() {
  }

  @Around("controllerLog()")
  public Object doAround(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
  HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();

      // 生成唯一的请求traceId,并加入日志链和上下文
      BizContextUtil.setReqStartTime(System.currentTimeMillis());
      String traceId = TraceUtil.buildTraceId();
      MDC.put(BizContextUtil.TRACE_ID, traceId);
      BizContextUtil.setTraceId(traceId);

      Object result = null;
      try {
        result = proceedingJoinPoint.proceed();
        LoggerUtil.SYS_LOG.info("uri:{},args:{},userId:{},ms:{}", request.getRequestURI(), JsonUtil.to(proceedingJoinPoint.getArgs()), BizContextUtil.getPortalUserId(), BizContextUtil.getUseTime());
      } catch (Exception e) {
          LoggerUtil.SYS_LOG.error("ControllerLogAspect error,uri:{},args:{},userId:{}", request.getRequestURI(), JsonUtil.to(proceedingJoinPoint.getArgs()), BizContextUtil.getPortalUserId());
          throw e;
      }
      return result;
  }
}
```

- 日志配置traceId打印
``` xml
<appender name="SYS_LOG" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <File>logs/sys.log</File>
    <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
      <fileNamePattern>logs/sys-%d{yyyy-MM-dd}.log</fileNamePattern>
      <maxHistory>30</maxHistory>
    </rollingPolicy>
    <encoder>
      <pattern> %d %p (%file:%line\)- [%X{traceId}] %m%n</pattern>
      <charset>UTF-8</charset>
    </encoder>
</appender>
```

- 跨线程调用传递：在跨线程调用时取出前一个线程的traceId，传递给下一个线程
``` java
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor) {
  BizContext bizContext = BizContextUtil.getBizContext();
  if (bizContext == null) {
    return CompletableFuture.supplyAsync(supplier::get, executor);
  }

  return CompletableFuture.supplyAsync(
    SupplierWrapper.of(() -> {
      BizContextUtil.setBizContext(bizContext);
      if (StringUtil.isNotEmpty(BizContextUtil.getTraceId())) {
        MDC.put(BizContextUtil.TRACE_ID, BizContextUtil.getTraceId());
      }
      U u = supplier.get();
      return u;
  }), executor);
}
```

- 跨服务调用传递：调用者往http的header设置traceId，服务提供者取出traceId放入threadLocal，如注入feign的RequestInterceptor；

``` java
@Component
public class FeignRequestInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        if (StringUtil.isNotEmpty(BizContextUtil.getTraceId())) {
            template.header(BizContextUtil.TRACE_ID, BizContextUtil.getTraceId());
        }
        if (BizContextUtil.getPortalUserId() > 0) {
            template.header(BizContextUtil.PORTAL_USER_ID, "" + BizContextUtil.getPortalUserId());
        }
        if (BizContextUtil.getUserId() > 0) {
            template.header(BizContextUtil.USER_ID, "" + BizContextUtil.getUserId());
        }
    }
}
```

一些微服务框架如阿里的dubbo，可以利用dubbo的`RpcContext`分別实现`DubboConsumerFilter`和`DubboProviderFilter`，前者负责将traceId放入RPC上下文，后者则取出traceId放入threadLocal；

### 拓展：
- nodejs版实现（待补充）
- go版实现（待补充）
- [SkyWalking][skywalking]
  ![](/images/skywalking.png)

[skywalking]: https://github.com/apache/skywalking