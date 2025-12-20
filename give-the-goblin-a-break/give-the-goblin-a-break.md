# Give the memory goblin a break! (Excessive memory consumption)
This article explores reducing the performance impact of observability measurements by evaluating their cost in terms of memory and CPU usage.

Deep inside your computer lives a nano-creature by name of Memory Goblin. It works tirelessly within your hardware to manage your computer’s memory. 
When you boot an application, write a document, or watch a video, the goblin within your machine organizes and saves data in memory.

Memory is designed as a vast matrix of drawers, a large cabinet. Each drawer represents one byte of data.
When every drawer is empty, the goblin inserts data starting at the first drawer and continues sequentially until every byte of data is stored away.
When every drawer contains an item, the goblin opens the least recently used drawer. Instead of removing the old data, the goblin compresses it down with the new data until the former turns into dust, and voilà, the drawer only contains the new data.
This goblin works relentlessly, never stopping for a break, vacation, or even a pay raise. But if it works too hard, it’ll eventually breakdown, so we must give it a break.

## A note on observability
For the past two years, I’ve been working on an ad exchange written in Java. It’s an incredible piece of software that transacts millions of auctions every second in an environment where every millisecond counts.

The observability of a system can be directly linked to its reliability. If an error can't be seen, then how can it be fixed? If trends aren't visible, then how can they be used to gain an advantage?
Within the lifetime of an exchange deployment (a few days), thousands of metrics are registered, each having trillions of measurements. These measurements are used to track error rates, ad spend, response times, the number of bids, and other key metrics.
Each metric plays an important role in revealing the overall health of the exchange. For example, a sharp increase in the error rate of an operation can expose an otherwise hidden bug. An increase in the response time of an API can hint at infrastructure scaling issues.
But metrics aren't free.
There is a delicate tradeoff between performance and observability. Having more observation data can lead to better conclusions about the state of the application, but this comes at a cost.
Managing excessive metrics can be detrimental to performance, especially if the weight of recording a measurement is not considered.

## Micrometer 
The exchange employs the Micrometer observability facade as its observability engine, the Prometheus flavour.
Micrometer supports the following types of *meters*: counters, gauges, timers, and distribution summaries. Going forward, the examples only use counters since they're used most often and are the simplest to understand.

At its simplest, a `Counter` is, well, a counter. It keeps count of the number of times an event occurs.

Before continuing, the reader must understand a few definitions and pitfalls of the following examples.
1. `MeterRegistry` is the interface provided by Micrometer to produce and manage the application's meters, such as its counters.
2. In most modern applications, an instance of `MeterRegistry` is injected into the application, and for simplicity, assume the same happens here.
3. The example code is incomplete, but the complete code for each example will be linked via GitHub. 

Here's a simplified example of how an instance of `MeterRegistry` is used to create and increment a counter:

```java
meterRegistry.counter("number_of_request").increment();
```

The code snippet creates a counter and increments it. This operation may appear simple and benign, but it can be insidious in nature because a lot is happening the hood.
Every time a counter is *initialized in this way*, Micrometer constructs multiple holding objects that contain the counter's name, ID, and tags (this example doesn't contain tags).
After initialization, Micrometer attempts to register the counter if it hasn't already done so.
The construction of this single counter has the potential to create a tremendous amount of memory pressure if it is created repeatedly.
Counters that track the most frequently occurring events within the exchange are created hundreds of millions of times per minute, accounting for about 3% of total memory usage.
This is unacceptable, considering most of this memory could be easily removed by rethinking when and how counters are created, giving the Garbage Collector (GC) a break.

### Efficient patterns for creating meters

By rethinking the exchange's approach to counter management, the amount of memory used by Micrometer was reduced from occupying 3% of application memory to only 1%, freeing 2% of memory for other important operations. 
Tools such as Java Flight Recorder (JFR) and JDK Mission Control were used to calculate memory usage estimations.
The rest of the article describes the patterns used to create counters efficiently, reducing CPU and memory usage.

#### A simple counter

The simplest way to create a counter has already been shown. The next code example will expand on this to display a more complete example.
The class below makes a request to an ad server. The way in which it makes the request is not important. What's important is how it keeps track of the number of requests made.

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
To mitigate wasting copious amounts of memory, create the counter once, store a reference to it in an instance variable, and use that refence to increment the counter.

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
     * Makes a request via {@link #makeRequest}. A reference to the counter is used to keep track of the number of 
     * requests made.
     */
    public fetch() {
        makeRequest();
        numberOfAdRequest.increment();
    }

    protected abstract void makeRequest();
}
```

The code above is much more efficient than the previous example because the counter is only created once, releasing memory pressure.

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

To prevent re-initializing a counter on each request. Create a counter for each possible `PayloadState` and cache the results in an `EnumMap`.
An `EnumMap` is an implementation of `Map` specifically used for `Enum` keys. Since every possible key is known, it stores values in an array, making insertion and retrieval faster than the traditional `HashMap`.

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
     * Upon construction, a instance of Counter is created for each possible state. These counters are stored in an 
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
     * Makes a request via {@link #makeRequest}. Keeps track of the number of requests for each possible state. 
     * An existing counter is fetched from the cache ({@link #counters}) to prevent the creation of a counter for each
     * request.
     */
    public void fetch() {
        PayloadState state = makeRequest();
        counters.get(state).increment();
    }

    protected abstract PayloadState makeRequest();

    protected enum PayloadState {
        REQUEST_SENT,
        RESPONSE_RECEIVED,
        REQUEST_FAILED_TO_SEND,
        RESPONSE_NOT_RECEIVED,
    }
}
```

## Performance Testing

I created two projects to test various ways of creating and caching counters to check how performant each option is.
The [first project](https://github.com/LajosPolya/Micrometer-Performance) contains various example patterns used to create and increment counters.
The [second project](https://github.com/LajosPolya/JMH-Test) contains the JMH testing harness.
JMH recommends this two project approach to ensure that the benchmarks are correctly initialized and produce reliable results.
I tested five options, these benchmarks are synonymous with the examples above, with the addition of one bonus test.

1. Using Micrometer to create a counter every time it is incremented. :black_circle:
2. Creating a counter once, storing a reference to it, and using that reference to increment the counter. :white_circle:
3. Using Micrometer to create a counter with one tag every time it is incremented. :red_circle:
4. Creating each counter once, for every possible tag value, storing a reference to the counters in an `EnumMap`, and using that map to increment the relevant counter. :large_blue_circle:
5. Creating each counter once, for every possible tag value, storing a reference to the counters in an `HashMap`, and using that map to increment the relevant counter. :large_orange_diamond:

### Performance testing with Java Flight Recorder (JFR)

JFR is a tool that runs alongside an application while recording low-level metrics about it, such as, memory usage and CPU profiling.  

As mentioned previously, in a production environment, it was verified that the exchange's memory consumption relating to Micrometer was reduced by 2%.
To take this testing one step further, I set up a testing framework to test the memory consumption of an application that increments a counter 2<sup>31</sup>-1 (2,147,483,647) times.

| Benchmark                                                                                                                                                                                                                     | Total Memory Usage (MiB) | Most Memory Intensive Micrometer Classes |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------:|:-----------------------------------------|
| 1. [Uncached counter with zero tags](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainReinstantiateTagless.java) :black_circle:                                         |                   18.095 | unmeasurable                             |
| 2. [Cached counter with zero tags](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainCacheTagless.java) :white_circle:                                                   |               80,019.728 | `Meter$Id`                               |
| 3. [Uncached counter with one randomly chosen `enum` tag](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainReinstantiateTagged.java) :red_circle:                       |              223,974.042 | `Tags`, `Tag[]`, `Meter$id`              |
| 4. [Counter cahched in `EnumMap` with one randomly chosen `enum` tag](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainCacheEnumMapTagless.java) :large_blue_circle:    |                   17.687 | unmeasurable                             |
| 5. [Counter cahched in `HashMap` with one randomly chosen `enum` tag](https://github.com/LajosPolya/Micrometer-Performance/blob/main/src/main/java/com/github/lajospolya/MainCacheHashMapTagless.java) :large_orange_diamond: |                   18.183 | unmeasurable                             |

Every example that cached its counters used about `~18MiB` of memory. What's amazing is when the counters weren't cached, they used many orders of magnitude more memory.
A counter with zero tags utilized `~80GiB` of memory, mostly for the construction of `Meter$Id`. When a tag was introduced, the memory usage tripled to `~224GiB` because the construction of each counter introduced the instantiation of `Tags` and `Tag[]`.
These superfluous objects are short-lived so they won't cause to out-of-memory errors, but they can trigger excessive GC usage which can hinder the application's performance.
There is one caveat, this test is not at all indicative of how an application runs in the real world but it does exemplify the level of waste introduced when performance is not considered.

// Add note about GC count

### Performance testing with Java Microbenchmark Harness (JMH)

JMH is a JVM tool specializing in performance testing applications where nanosecond accuracy is a necessary.

The single threaded [benchmarks](https://github.com/LajosPolya/JMH-Test/blob/main/src/main/java/com/github/lajospolya/MicrometerCounterBenchmark.java) are broken down into two categories; "tagless" and "tagged".
Tagless counters are the simplest example of a counter, created like this; `meterRegistry.counter("counter")`, see, no tags.
Tagged counters contain one tag, created like this; `meterRegistry.counter("counter", "tag_key", "tag_value")`.
The reason the tagged benchmarks are many orders of magnitude slower than the untagged is because in order to randomly test counters with many different tags, I had to run the benchmark in a loop.
So each tagged benchmark is really testing 1,000,000 iterations, while the untagged benchmarks only increment one counter.

| Benchmark                                                                        | Mode | Cnt |         Score |          Error | Units |
|----------------------------------------------------------------------------------|------|-----|--------------:|---------------:|-------|
| 1. MicrometerCounterBenchmark.notCachedTaglessCounter :black_circle:             | avgt | 5   |        12.656 |  ±       0.354 | ns/op |
| 2. MicrometerCounterBenchmark.cachedTaglessCounter :white_circle:                | avgt | 5   |         8.078 |  ±       0.161 | ns/op |
| 3. MicrometerCounterBenchmark.notCachedTaggedCounters :red_circle:               | avgt | 5   |  27357777.939 |  ± 1053230.354 | ns/op |
| 4. MicrometerCounterBenchmark.enumMapCachedTaggedCounters :large_blue_circle:    | avgt | 5   |   5719102.557 |  ±   19151.749 | ns/op |
| 5. MicrometerCounterBenchmark.hashMapCachedTaggedCounters :large_orange_diamond: | avgt | 5   |   6421308.149 |  ±   24017.314 | ns/op |

#### Untagged Counters
It takes about 1/3 fewer CPU cycles to increment a counter when it is cached vs when it's not.

#### Tagged Counters
It takes about 5 times fewer CPU cycles to increment a counter with an `Enum` tag when it's cached in an `EnumMap` vs when it's not cached.
What's surprising is that using a `HashMap` in a single-threaded environment is only about 11% slower than using an `EnumMap`.

## Final Thoughts
It's necessary for an application to have an adequate amount of observability, but increasing observability can lead to significant performance degradation if the correct patterns aren't used.
Therefore, it's important always keep memory usage in mind to keep the application running as efficiently as possible.

### The Memory Goblin
The idea of a memory goblin was brought to me by my most influential high school teacher, Anthony Viola, over 15 years ago.
He mentioned that a prominent journal published an article about how computer memory works, and if I recall correctly, the article described memory management as a goblin managing drawers or shelves.
I don’t remember the article’s title, the publisher, or the author. I tried looking for it using Gemini and ChatGPT, but they didn’t return any useful information.

### Relevant links
* https://github.com/LajosPolya/Micrometer-Performance
* https://github.com/LajosPolya/JMH-Test
* https://github.com/micrometer-metrics/micrometer/wiki/Performance:-criticality-of-code-paths
* https://vertx.io/blog/micrometer-metrics-performance/
* https://docs.micrometer.io/micrometer/reference/concepts/meter-provider.html