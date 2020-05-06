# java服务,响应慢如何定位?

工作中我们经常遇到服务响应慢情况,这时我们需要定位哪个方法的执行拖了后退,对症下药进行优化.

### 方案一

```
long start = System.currentTimeMillis();
// doSomthing
long useTime = System.currentTimeMillis() -start;
log.debug("execute method {} of {} use time {}ms",useTime);
```



