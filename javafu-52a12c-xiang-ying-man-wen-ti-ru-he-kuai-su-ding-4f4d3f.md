# java服务,响应慢如何定位?

工作中我们经常遇到服务响应慢情况,这时我们需要定位哪个方法的执行拖了后退,对症下药进行优化.

### 方案一:通过系统时间

```
long start = System.currentTimeMillis();
// doSomthing
long usedTime = System.currentTimeMillis() -start;
log.debug("execute method XXX use time {}ms",usedTime);
```

### 方案二:通过guava工具类

```
<!-- 引入guava包 -->
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>20.0</version>
</dependency>
```

```
Stopwatch stopWatch = Stopwatch.createStarted();
// doSomthing
log.debug("execute method XXX use time {}ms",stopWatch.elapsed(TimeUnit.MILLISECONDS));
```

### 方案三:Spring AOP

* 定义注解

```
package com.easysoft.annotation;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ExecuteTime {
}
```

* 实现日志切面

```
import com.google.common.base.Stopwatch;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;

@Aspect
@Component
@Slf4j
public class ExecuteTimeAspect {


    @Pointcut("@annotation(com.easysoft.annotation.ExecuteTime)")
    public void serviceExecutionTimeLog() {
    }


    @Around(value = "serviceExecutionTimeLog()", argNames = "proceedingJoinPoint")
    public Object doAfter(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        Stopwatch stopWatch = Stopwatch.createStarted();
        Object proceed = proceedingJoinPoint.proceed();
        log.debug("execute method {} of {} use time {}ms",
                proceedingJoinPoint.getSignature().getName(),
                proceedingJoinPoint.getTarget().getClass().getName(),
                stopWatch.elapsed(TimeUnit.MILLISECONDS)
        );
        return proceed;
    }
}
```

* 添加注解

```
@ExecuteTime
public void call(){
// TODO
}
```



