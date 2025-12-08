# Configuration induced performance issue

A tale about how a seemingly benign \[mis]configuration resulted in wasted CPU cycles.

For the past two years, I’ve been working on an ad exchange written with the Java Spring Framework. It’s an incredible piece of software which transacts millions of auctions every second in an environment where every millisecond counts.

My curiosity about the application’s performance has run me down the path of deciphering Java Flight Recorder (JFR) files via JDK Mission Control, diving deep into memory hotspots and operations that hog CPU cycles.

While reviewing the list of our worst-performing methods, I noticed something curious, one function in particular: `IdentityHashMapIterator.hasNext()` uses about 0.6% of CPU cycles. This caught my attention because, as far as I know, the exchange doesn’t ubiquitously use this specific implementation of Map.

I stepped through the flame chart and discovered that this usage doesn’t even come from our codebase; it’s contained within Micrometer’s `CompositeCounter.increment` method. 

![hasNext.png](attachments/hasNext.png)

Diving into the `CompositeMeterRegistry.increment` method. Where is `hasNext` actually called?

```java
@Override
public void increment(double amount) {
    for (Counter c : getChildren()) {
        c.increment(amount);
    }
}
``` 

In Java, the enhanced for loop is syntactic sugar for looping through a collection using its iterator’s `hasNext` and `next()` methods. To break down where `hasNext` is actually called, I disassembled the class file into assembly. In short, `getChildren` returns Map (which is an Iterable), and the enhanced for loop implicitly calls Iterator.hasNext and Iterator.next()

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

Zooming out a little further, why is `increment()` called in a loop?

Spring should inject the `CompositeMeterRegistry` only if the application supports multiple implementations of `MeterRegistry`. When CompositeMeterRegistry creates a Counter, it creates a CompositeCounter which stores its various registries in an IdentityHashMap. When `CompositeCounter.increment()` is called, it loops through each Counter implementation and calls increment on each of them. What’s unusual is the `IdentityHashMap` only contains one element, `PrometheusMeterRegistry`. If it’s the only registry present at runtime, why isn’t it being injected directly?

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
3. If a bean of type `MeterRegistry` doesn’t exist, the existing bean of type `MeterRegistry` contained in `MeterManager` is returned. This is confusing because, we already know that we have a bean of `MeterRegistry`. After all, `MeterManager` depends on it, making this completely redundant, but running the code confirms that this gets executed. Why?

When `CompositeMeterRegistry` is created, it identifies beans that implement `MeterRegistry`. Through this search, it finds the `PrometheusMeterRegistry` via Micrometer dependency, then it finds it again through the `MeterConfig` configuration file, for some reason ignoring the `@ConditionalOnMissingBean` annotation. Why?

As it turns out, `@ConditionalOnBean` should only be used on auto-configured configuration files or the condition may not be honoured. This is even stated in the https://docs.spring.io/spring-boot/api/java/org/springframework/boot/autoconfigure/condition/ConditionalOnMissingBean.html.

> The condition can only match the bean definitions that have been processed by the application context so far and, as such, it is strongly recommended to use this condition on auto-configuration classes only. If a candidate bean may be created by another auto-configuration, make sure that the one using this condition runs after.
  
To fix the issue I removed our redundant construction of `MeterRegistry`. I booted the exchange and verified that `PrometheusMeterRegistry` is injected into the application. Now that the correct implementation is injected, the `increment` method no longer contains a loop saving an additional 0.6% of CPU cycles and reduced memory pressure from not having to create iterators on each call to `increment`. This iterator normally wouldn’t cause any issues but the exchange produces hundreds of millions of metrics per minute leading to hundreds of millions of iterator instantiations and `hasNext` executions.
  
In a nutshell, a small configuration error introduced an almost unnoticed superfluous operation leading to wasted CPU cycles. To fix this, all that was needed was a bit of curiosity, a couple of hours of debugging, and the removal of 5 lines of code.


- Szóbör Bober  
