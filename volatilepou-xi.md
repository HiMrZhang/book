# volatile剖析

实际上这组happens-before规则定义了操作之间的内存可见性，如果A操作happens-before B操作，那么A操作的执行结果（比如对变量的写入）必定在执行B操作时可见。

JMM 内部的实现通常是依赖于所谓的内存屏障，通过禁止某些重排序的方式，提供内存可见性保证，也就是实现了各种 happen-before 规则。



