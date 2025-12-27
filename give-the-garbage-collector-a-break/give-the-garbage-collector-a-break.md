# Give the Garbage Collector a break!
This article explores reducing the performance impact of observability metrics by evaluating their cost in terms of memory and CPU usage.

## A note on observability
For the past two years, I’ve been working on an ad exchange written in Java. It’s an incredible piece of software that transacts millions of auctions every second in an environment where every millisecond counts.

Observability metrics are crucial for maintaining a reliable system. If the occurrence of an error isn't surfaced, then it can't be fixed. If auction trends aren't made visible, they can't be used to gain an advantage.
Over the course of a few days, trillions of measurements and thousands of metrics are registered in the exchange. These metrics track things like error rates, ad spend, response times, and the number of bids.
Each metric plays an important role in revealing the overall health of the exchange. A sharp increase in the error rate of an operation would be a loud signal that something is wrong. Likewise, an increase in the response time of an API could be a hint at infrastructure scaling issues.

Unfortunately, metrics aren't free. There's a delicate tradeoff between performance and observability. Having more observation data can lead to better conclusions about the state of the application, but creating an excessive number of metrics can itself be detrimental to performance.

## Micrometer 
The exchange employs Micrometer as its observability engine, specifically the Prometheus flavour.
Micrometer supports the following types of *meters*: counters, gauges, timers, and distribution summaries. Going forward, the examples will only use counters since they're used most often and are the simplest to understand.

*Counters* keep track of the number of times an event occurs.

Going forward, there are a few definitions and pitfalls to cover:
1. `MeterRegistry` is the interface provided by Micrometer to produce and manage the application's meters, such as its counters.
2. In most modern applications, an instance of `MeterRegistry` is injected into the application, for simplicity, assume the same happens here.
3. The code snippets shared here are incomplete, but the complete code for each example will be linked.

Here's a simplified example of how an instance of `MeterRegistry` is used to create and increment a counter:

```java
meterRegistry.counter("number_of_request").increment();
```

This operation may appear simple and benign, but it's insidious in nature because a lot is happening the hood.
Every time a counter is *initialized in this way*, Micrometer constructs multiple holding objects that contain the counter's name, ID, and tags (if used).
After initialization, Micrometer attempts to register the counter if it hasn't already done so.
When this single counter is repeatedly constructed, it has the potential to create a tremendous amount of memory pressure.
In fact, counters that track the most frequently occurring events within the exchange are created hundreds of millions of times per minute, accounting for about 3% of total memory usage.
Most of this memory could be easily removed by rethinking when and how counters are created, giving the Garbage Collector (GC) a break!

### Efficient patterns for creating meters

By rethinking the exchange's approach to counter management, the amount of memory used by Micrometer was reduced from occupying 3% of application memory to only 1%, freeing 2% of memory for other important operations. 
Tools such as Java Flight Recorder (JFR) and JDK Mission Control were used to calculate memory usage estimations.
The rest of the article describes the patterns used to create counters efficiently, reducing CPU and memory usage.

#### A simple counter

The next code example will expand on the first example. The class below makes a request to an ad server. The way a request is made isn't important. What's important is how the count of requests are tracked.

[Full example here:](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/meterRegistry/ReinstantiatedTaglessCounter.java)
```java
/**
 * A service, responsible for making an ad request. A Counter is initialized every time a request is made to keep count
 * of the number of requests made. As you can imagine, this can be quite inefficient.
 */
public abstract class AdRequestService {

    private final MeterRegistry meterRegistry;

    public AdRequestService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    /**
     * Makes a request via {@link #makeRequest}. A counter is initialized every time a request is made.
     */
    public fetch() {
        makeRequest();
        meterRegistry.counter("number_of_request").increment();
    }

    protected abstract void makeRequest();
}
```

As mentioned, this code can be quite inefficient when called repeatedly because the amount of memory used by Micrometer is directly proportional to the number of times the counter is incremented.
To mitigate wasting copious amounts of memory, the counter should be created once and stored in an instance variable. The stored reference can then be used to increment the counter without additional memory usage.

[Full example here:](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/meterRegistry/CachedTaglessCounter.java)
```java
/**
 * A service, responsible for making an ad request. A Counter is initialized once in the constructor and a reference to
 * it is re-used to keep count of the number of requests made.
 */
public abstract class AdRequestService {

    private final Counter numberOfAdRequests;

    public AdRequestService(MeterRegistry meterRegistry) {
        this.numberOfAdRequest = meterRegistry.counter("number_of_requests");
    }

    /**
     * Makes a request. A reference to the counter is used to keep track of the number of requests made.
     */
    public fetch() {
        makeRequest();
        numberOfAdRequest.increment();
    }
}
```

The code above is now much more efficient than the previous example since the counter is only created once!

#### Counters with Enum tags

Counters using `enum` tags offer a great way to keep track of a finite set of states, but introduce additional memory pressure through the creation and storage of tags.

[Full example here:](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/meterRegistry/ReinstantiatedTaggedCounter.java)
```java
/**
 * Makes a request and keeps track of the number of requests in each possible state. A counter is initialized with every
 * request.
 */
public void fetch() {
    PayloadState state = makeRequest();
    meterRegistry.counter("number_of_requests", "state", state.name()).increment();
}

private enum PayloadState {
    REQUEST_SENT,
    RESPONSE_RECEIVED,
    REQUEST_FAILED_TO_SEND,
    RESPONSE_NOT_RECEIVED,
}
```

To prevent re-initializing a counter on each request. A counter for each possible `PayloadState` should be created and cached in an `EnumMap`.
An `EnumMap` is an implementation of `Map` specifically used for `enum` keys. Since every possible key is known, values are stored in an array, making insertion and retrieval faster than the traditional `HashMap`.

[Full example here:](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/meterRegistry/CachedEnumMapTaggedCounter.java)
```java
public abstract class AdRequestService {

    private final MeterRegistry meterRegistry;
    private final Map<PayloadState, Counter> counters;

    public AdRequestService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        counters = getCounters();
    }

    /**
     * Upon construction, an instance of Counter is created for each possible state. These counters are stored in an 
     * EnumMap for fast retrieval.
     */
    private Map<PayloadState, Counter> getCounters() {
        Map<PayloadState, Counter> tempCounters = new EnumMap<>(PayloadState.class);
        for(PayloadState state : PayloadState.values()){
            tempCounters.put(state, meterRegistry.counter("number_of_requests", "state", state.name()));
        }
        return tempCounters;
    }

    /**
     * Makes a request and keeps track of the number of requests for each possible state. An existing counter is fetched
     * from the cache ({@link #counters}) to prevent the creation of a counter for each
     * request.
     */
    public void fetch() {
        PayloadState state = makeRequest();
        counters.get(state).increment();
    }
}
```

## Performance Testing
To compare the relative performance of these approaches, I created two projects to test different ways of creating and caching counters.
The [first project](https://github.com/LajosPolya/Micrometer-Performance) contains various example patterns used to create and increment counters.
The [second project](https://github.com/LajosPolya/JMH-Test) contains the Java Microbenchmark Harness (JMH) framework. JMH is a tool with the JVM specializing in performance testing applications where nanosecond accuracy is a necessary.
JMH recommends this two project approach to ensure the benchmarks are correctly initialized and produce reliable results.

I tested five scenarios:
1. Using Micrometer to create a counter every time it's incremented. :black_circle:
2. Creating a counter once, storing a reference to it, and using that reference to increment the counter. :white_circle:
3. Using Micrometer to create a counter with one tag every time it's incremented. :red_circle:
4. Creating each counter once, for every possible tag value, storing a reference to the counters in an `EnumMap`, and using that map to increment the relevant counter. :large_blue_circle:
5. Creating each counter once, for every possible tag value, storing a reference to the counters in an `HashMap`, and using that map to increment the relevant counter. :large_orange_diamond:

### Gathering performance metrics with Java Flight Recorder (JFR)
JFR is a tool that runs alongside an application while recording low-level metrics about it, such as, memory usage and CPU profiling.  

In a production environment, JFR verified the exchange's memory consumption relating to Micrometer was reduced by 2%.
To take this testing one step further, I set up a testing framework to test the memory consumption of an application that increments a counter 2<sup>31</sup>-1 (2,147,483,647) times.

| Benchmark                                                                                                                                                                                                                     | Total Memory Usage (MiB) | Most Memory Intensive Micrometer Classes |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------:|:-----------------------------------------|
| 1. [Uncached counter with zero tags](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainReinstantiateTagless.java) :black_circle:                                         |                       18 | insignificant                            |
| 2. [Cached counter with zero tags](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainCacheTagless.java) :white_circle:                                                   |                   80,020 | `Meter$Id`                               |
| 3. [Uncached counter with one randomly chosen `enum` tag](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainReinstantiateTagged.java) :red_circle:                       |                  223,974 | `Tags`, `Tag[]`, `Meter$id`              |
| 4. [Counter cahched in `EnumMap` with one randomly chosen `enum` tag](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainCacheEnumMapTagless.java) :large_blue_circle:    |                       18 | insignificant                            |
| 5. [Counter cahched in `HashMap` with one randomly chosen `enum` tag](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainCacheHashMapTagless.java) :large_orange_diamond: |                       18 | insignificant                            |

Every test that cached its counters used about `~18MiB` of memory. What's amazing is when the counters weren't cached, they used orders of magnitude more memory.
The repeated creation of a counter with zero tags needlessly utilized `~80GiB` of memory, mostly for the construction of `Meter$Id`. When a tag was introduced, the memory usage tripled to `~224GiB` because the construction of each counter introduced the instantiation of two more objects; `Tags` and `Tag[]`.
These superfluous objects won't cause out-of-memory errors because they are short-lived, but their existence may trigger the GC excessively, unnecessarily taking up resources and hindering the application's performance.
In the tests where metrics weren't cached, the GC was invoked hundreds of times, in a rather short period of time. Surprisingly, the GC was never invoked in tests that cached their metrics. 
This test isn't indicative of how an application runs in the real world, but it does exemplify the level of waste introduced when performance isn't considered!

### Benchmarking with JMH
The single threaded [benchmarks](https://github.com/LajosPolya/JMH-Test/blob/main/src/main/java/com/github/lajospolya/MicrometerCounterBenchmark.java) are broken down into two categories; "tagless" and "tagged".
Tagless counters are the simplest example of a counter; `meterRegistry.counter("counter")`.
Tagged counters contain at least one tag; `meterRegistry.counter("counter", "tag_key", "tag_value")`.
The reason the tagged benchmarks are many orders of magnitude slower than the untagged is because in order to randomly test counters with many different tags, I had to run the benchmark in a loop.
So each tagged benchmark is really testing 1,000,000 iterations, while the untagged benchmarks only increment one counter. The results of the two types of tests shouldn't be compared.

| Benchmark                                                                        |  Score (ns/op) |   Error (ns/op) |
|----------------------------------------------------------------------------------|---------------:|----------------:|
| 1. MicrometerCounterBenchmark.notCachedTaglessCounter :black_circle:             |         12.656 |   ±       0.354 |
| 2. MicrometerCounterBenchmark.cachedTaglessCounter :white_circle:                |          8.078 |   ±       0.161 |
| 3. MicrometerCounterBenchmark.notCachedTaggedCounters :red_circle:               | 27,357,777.939 | ± 1,053,230.354 |
| 4. MicrometerCounterBenchmark.enumMapCachedTaggedCounters :large_blue_circle:    |  5,719,102.557 |  ±   19,151.749 |
| 5. MicrometerCounterBenchmark.hashMapCachedTaggedCounters :large_orange_diamond: |  6,421,308.149 |  ±   24,017.314 |

#### Untagged Counters
It takes about 1/3 fewer CPU cycles to increment a counter when it's cached vs when isn't.

#### Tagged Counters
It takes about 5 times fewer CPU cycles to increment a counter with an `enum` tag when it's cached in an `EnumMap` vs when isn't cached.
Surprisingly, using a `HashMap` in a single-threaded environment is only about 11% slower than using an `EnumMap`.

## Final Thoughts
It's necessary for an application to have an adequate amount of observability, but increasing observability can lead to significant performance degradation if the correct patterns aren't used.
Therefore, it's important always keep memory usage in mind to keep the application running as efficiently as possible.

### Relevant links
* https://github.com/LajosPolya/Micrometer-Performance
* https://github.com/LajosPolya/JMH-Test
* https://github.com/micrometer-metrics/micrometer/wiki/Performance:-criticality-of-code-paths
* https://vertx.io/blog/micrometer-metrics-performance/
* https://github.com/micrometer-metrics/micrometer/pull/6670
* https://docs.micrometer.io/micrometer/reference/concepts/meter-provider.html