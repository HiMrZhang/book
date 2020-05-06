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



