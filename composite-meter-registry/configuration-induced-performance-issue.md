# Always read the docs

A tale about how a seemingly benign \[mis]configuration resulted in wasted CPU cycles... and why you should always read the docs.

For the past two years, I’ve been working on an ad exchange written with the Java Spring Framework. It’s an incredible piece of software which transacts millions of auctions every second in an environment where every millisecond counts.

My curiosity about the application’s performance led me down a path of deciphering Java Flight Recorder (JFR) files via JDK Mission Control, diving deep into memory hotspots and operations that hog CPU cycles.

While reviewing the list of our worst-performing methods, I noticed something curious. One method in particular, `IdentityHashMapIterator.hasNext`, was using about 0.6% of CPU cycles. This caught my attention because the exchange doesn’t ubiquitously use this specific implementation of `Map`.

I stepped through the flame chart and discovered that this wasn't even coming from our codebase; it was contained within Micrometer’s `CompositeCounter.increment` method. 

![hasNext.png](attachments/hasNext.png)

Next, I dove into the `CompositeCounter.increment` method to determine where `hasNext` was actually being called. In Java, the enhanced for-loop is syntactic sugar for traversing through a collection using its iterator’s `hasNext` and `next` methods. 

```java
@Override
public void increment(double amount) {
    for (Counter c : getChildren()) {
        c.increment(amount);
    }
}
``` 

To break down where `hasNext` was called, I disassembled the class file into assembly. In short, `getChildren` returns `Map` (which is an `Iterable`), and the enhanced for-loop implicitly calls `Iterator.hasNext` and `Iterator.next`.

```
public void increment(double);
  Code:
     0: aload_0
     1: invokevirtual #7                  // Method getChildren:()Ljava/lang/Iterable;
     4: invokeinterface #13,  1           // InterfaceMethod java/lang/Iterable.iterator:()Ljava/util/Iterator;
     9: astore_3
    10: aload_3
    11: invokeinterface #19,  1           // InterfaceMethod java/util/Iterator.hasNext:()Z
    16: ifeq          41
    19: aload_3
    20: invokeinterface #25,  1           // InterfaceMethod java/util/Iterator.next:()Ljava/lang/Object;
    25: checkcast     #29                 // class io/micrometer/core/instrument/Counter
    28: astore        4
    30: aload         4
    32: dload_1
    33: invokeinterface #31,  3           // InterfaceMethod io/micrometer/core/instrument/Counter.increment:(D)V
    38: goto          10
    41: return
```

Zooming out a little further, why is `increment` called in a loop?

Spring should inject the `CompositeMeterRegistry` only if the application supports multiple implementations of `MeterRegistry`. When `CompositeMeterRegistry` creates a `Counter`, it constructs a `CompositeCounter`, which stores its various registries in an `IdentityHashMap`. When `CompositeCounter.increment` is called, it loops through each `Counter` implementation and calls `increment` on each of them. What was unusual was that the `IdentityHashMap` only contained one element, `PrometheusMeterRegistry`. If it’s the only registry present at runtime, why isn’t it being injected directly?

In our code, I found something that didn’t make any sense:

```java
@Configuration
public class MeterConfig {
    // 1
    @Bean
    protected MeterManager meterManager(PrometheusMeterRegistry prometheusMeterRegistry) {
        // 2
        return new MeterManager(prometheusMeterRegistry);
    }
    
    // 3
    @Bean
    @ConditionalOnMissingBean
    protected MeterRegistry meterRegistry(MeterManager meterManager) {
        return meterManager.getMeterRegistry();
    }
}
```

Let’s break down the sequence of events. Before continuing, I will assume the reader has a basic understanding of Spring.

1. `PrometheusMeterRegistry` implements `MeterRegistry`. Spring creates a bean of type `PrometheusMeterRegistry` and injects it into the `MeterConfig.meterManager` bean method
2. A bean of type `MeterManager` is created
3. If a bean of type `MeterRegistry` doesn’t exist, the existing bean of type `MeterRegistry` contained in `MeterManager` is returned. This is confusing because we already know that we have a bean of type `MeterRegistry`. After all, `MeterManager` depends on it, making this completely redundant, but running the code confirms that this gets executed. Why?

When `CompositeMeterRegistry` is created, it identifies beans that implement `MeterRegistry`. Through this search, `PrometheusMeterRegistry` is found in the Micrometer dependency; subsequently, the same instance is found in the `MeterConfig` configuration file. For some reason, the `@ConditionalOnMissingBean` annotation was ignored. Why?

As it turns out, `@ConditionalOnBean` should only be used on auto-configured configuration files, or the condition may not be honoured. This is even stated in the [javadoc](https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnMissingBean.html).

> The condition can only match the bean definitions that have been processed by the application context so far and, as such, it is strongly recommended to use this condition on auto-configuration classes only. If a candidate bean may be created by another auto-configuration, make sure that the one using this condition runs after.

This iterator normally wouldn’t cause any issues, but the exchange produces hundreds of millions of metrics per minute, meaning hundreds of millions of iterator instantiations and `hasNext` executions.

To fix the issue, I removed our redundant construction of `MeterRegistry`. After a restart, I could see that `PrometheusMeterRegistry` was now injected into the application as expected. The `increment` method no longer contained a loop, saving an additional 0.6% of CPU cycles and reducing memory pressure as the application no longer needed to create an iterator on each call to `increment`.
A 0.6% performance hit may not seem significant, but in an environment where every millisecond matters, we'll take all that we can get. Not to mention, the operation was completely redundant, and therefore, it had to be removed.

In a nutshell, a minor configuration error introduced an unnecessary operation that went unnoticed for three years, resulting in numerous wasted CPU cycles. All that was needed to fix it was a bit of curiosity, a couple of hours of debugging, and the removal of 5 lines of code.

- Szóbör Bober  
