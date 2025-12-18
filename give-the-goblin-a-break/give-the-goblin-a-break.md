# Give the memory goblin a break! (Excessive memory consumption)
This article dives into reducing the performance impact of observability measurements by evaluating its cost on memory and CPU.

Deep inside your computer lives a small creature called the Memory Goblin. It works tirelessly within your hardware to manage your computer’s memory. 
When you open a program, write a document, or watch a video; in the background, the goblin within your machine, streams and saves data in memory.

Memory is designed as a vast matrix of drawers, a large cabinet. Each drawer represents one byte of data.
When every drawer is empty, the goblin inserts data starting at the first drawer, and continues sequentially, until every byte of data is stored away.
When every drawer contains an item, the goblin opens the least recently used drawer. Instead of removing the old data, the goblin compresses it down with the new data until the former turns into dust, and voilà, the drawer only contains the new the data.
This goblin works relentlessly, never stopping for a break, vacation, or even a pay raise. But if it works too hard it’ll eventually overheat, so we must give it a break.

The idea of a memory goblin was brought to me by my most influential high school teacher, Anthony Viola, over 15 years ago. 
He mentioned that a prominent journal published an article about how computer memory works, and if I recall correctly, the article described memory management as a goblin managing drawers or shelves.
I don’t remember the article’s title, the publisher, or the author. I tried looking for it using Gemini and ChatGPT but they didn’t return any useful information.


## A note on observability

For the past two years, I’ve been working on an ad exchange written in Java. It’s an incredible piece of software which transacts millions of auctions every second in an environment where every millisecond counts.

The observability of a system can be directly linked to its reliability. If you can't see and error then how can you fix it? If you can't see trends then how can you use them to your advantage?
On the exchange, we have thousands of metrics, with hundreds of millions of measurements. We use these measurements to track error rates, ad spend, response times, number of bids, etc.
Each metric plays an important role in observing the overall health of the exchange. For example, a sharp increase in the error rate of an operation can bring to light a bug. An increase in response times can hint to infrastructure scaling issues.
But metrics aren't free.
The tradeoff between performance and observability is delicate. The more data we have, the better we can infer about state of the application, but this comes at a cost. 
Managing too many metrics can be detrimental to performance especially if the weight of recording a metric is not considered.


## Micrometer 
Within the exchange, we use the Micrometer observability facade, the Prometheus flavour.
Micrometer supports many types of *meters*; counters, gauges, timers, and distribution summaries. For simplicity, I'll use counters since they're used most often and are the simplest to understand.

At its simplest, a `Counter` is, well, a counter. It keeps count of "something".

Before continuing the reader must understand a couple of things.
1. `MeterRegistry` is the interface provided by Micrometer to produce and store meters, such as a counter.
2. In most modern applications, an instance of `MeterRegistry` is injected into the application, and for simplicity we're going to assume the same happens here.

Let's look at a simplified example:

```java
meterRegistry.counter("number_of_request").increment();
```
The code snippet above creates a counter and increments it. This operation looks simple and benign, but can be insidious in nature because a lot is going on under the hood.
Every time a counter is *initialized in this way*, Micrometer creates an object to sort and store all the counter's tags (This example doesn't contain tags). Then a `builder` object is instantiated to hold the meter's name and tags.
Then in order to register the counter, an ID is constructed and used as a key to store the counter in a map. 
This single counter has the potential to create a tremendous amount of memory pressure if it created often enough.
Some of the exchange's counters are created hundreds of millions of times per minute, accounting for about 3% of total memory usage. This is huge considering most of this memory pressure could be easily removed, giving the Garbage Collector (GC) a break.

### Efficient patterns for creating meters

In the exchange, I was able to reduce Micrometer's total memory usage from 3% to 1%, deleting 2% of redundant memory.
The rest of the article describes the patterns I used to create counters efficiently to reduce CPU and memory usage.

#### A simple counter

I already showed the most simple way to create a counter. The code example below will expand on this to a full working example.
The class below makes a request to an ad server. The way in which it makes the request is not important. What's important is how it keeps track of the number of requests made.

```java
/**
 * A service, responsible for making an ad request. A {@link io.micrometer.core.instrument.Counter} is initialized every
 * time a request is made to keep count of the number of requests made. As you can imagine, this can be quite inefficient.
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
As mentioned above, this line of code can be quite inefficient when called a large number of times because the amount of memory used by Micrometer is directly proportional to the number of times the counter is incremented.
To mitigate the creation of copious amounts of memory, create the counter once, store a reference to it in an instance variable, and use that refence when incrementing the counter.


```java
/**
 * A Service used to make an ad request. A Counter is initialized once in the constructor and a reference to it is re-used
 * to keep count of the number of requests made.
 */
public abstract class AdRequestService {
    
    private final Counter numberOfAdRequest;
    
    public AdRequestService(MeterRegistry meterRegistry) {
        this.numberOfAdRequest = meterRegistry.counter("number_of_request");
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

Counters using Enum tags offer a great way to keep track of a finite set of states.

```java
/**
 * Makes a request and keeps track of the number of requests in each possible state. A counter is initialized with every
 * request.
 */
public void fetch() {
    PayloadState state = makeRequest();
    meterRegistry.counter("number_of_request", "state", state.name()).increment();
}

private enum PayloadState {
    REQUEST_SENT,
    RESPONSE_RECEIVED,
    REQUEST_FAILED_TO_SEND,
    RESPONSE_NOT_RECEIVED,
}
```

To prevent re-initializing a counter for each request. Create a counter for each possible `PayloadState` and cache the results in an `EnumMap`.
An `EnumMap` is an implementation of `Map` specifically used for `Enum` keys. Since every possible key is known, it stores values in an array make insertion and retrieval much faster.

```java
public abstract class AdRequestService {

    private final MeterRegistry meterRegistry;
    private final Map<PayloadState, Counter> counters;

    public AdRequestService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        counters = getCounters();
    }

    /**
     * Upon construction, a instance of {@link Counter} is created for each possible state. These counters are stored in 
     * an {@link EnumMap} for fast retrieval.
     */
    private Map<PayloadState, Counter> getCounters() {
        Map<PayloadState, Counter> tempCounters = new EnumMap<>(PayloadState.class);
        for(PayloadState state : PayloadState.values()){
            tempCounters.put(state, meterRegistry.counter("number_of_request", "state", state.name()));
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

### Performance testing with Java Microbenchmark Harness (JMH)

JMH is a JVM tool which allows performance testing applications with nanosecond accuracy.
I created two projects to test various ways of creating and caching counters to check how performant each option is.
The [first project](https://github.com/LajosPolya/Micrometer-Performance) contains the code to be tested.
The [second project](https://github.com/LajosPolya/JMH-Test) contains the JMH testing harness.
JMH recommends this two project approach to ensure that the benchmarks are correctly initialized and produce reliable results.
I tested five options, these benchmarks are synonymous with the examples above, with the addition of one bonus test.

1. Using Micrometer to create a counter every time it is incremented
2. Creating a counter once, storing a reference to it, and using that reference to increment the counter.
3. Using Micrometer to create a counter with a tag every time it is incremented
4. Creating a counter once, for every possible tag value, storing a reference to the counters in an `EnumMap`, and using that map to increment the relevant counter. 
5. Creating a counter once, for every possible tag value, storing a reference to the counters in an `HashMap`, and using that map to increment the relevant counter.

I'm going to break down the single threaded benchmarks into two categories; "tagless" and "tagged".
Tagless counters are the simplest example of a counter, created like this; `meterRegistry.counter("counter")`, see, no tags.
Tagged counters contain one tag, created like this; `meterRegistry.counter("counter", "tag_key", "tag_value")`.
The reason the tagged benchmarks are many orders of magnitude slower than the untagged is because in order to randomly test counters with many different tags, I had to run the benchmark in a loop.
So each tagged benchmark is really testing 1,000,000 iterations, while the untagged benchmark only increments one counter.

| Benchmark (1 thread)                                   | Mode | Cnt |         Score |          Error | Units |
|--------------------------------------------------------|------|-----|--------------:|---------------:|-------|
| MicrometerCounterBenchmark.notCachedTaglessCounter     | avgt | 5   |        12.656 |  ±       0.354 | ns/op |
| MicrometerCounterBenchmark.cachedTaglessCounter        | avgt | 5   |         8.078 |  ±       0.161 | ns/op |
| MicrometerCounterBenchmark.notCachedTaggedCounters     | avgt | 5   |  27357777.939 |  ± 1053230.354 | ns/op |
| MicrometerCounterBenchmark.enumMapCachedTaggedCounters | avgt | 5   |   5719102.557 |  ±   19151.749 | ns/op |
| MicrometerCounterBenchmark.hashMapCachedTaggedCounters | avgt | 5   |   6421308.149 |  ±   24017.314 | ns/op |

#### Untagged Counters
It takes about 1/3 fewer CPU cycles to increment a counter when it is cached vs when it's not.

#### Tagged Counters
It takes about 5 times fewer CPU cycles to increment a counter with an `Enum` tag when it's cached in an `EnumMap` vs when it's not cached.
What's surprising is that using a `HashMap` is only about 18% slower than using an `EnumMap`.

| Benchmark (64 threads)                                 | Mode | Cnt |          Score |          Error | Units |
|--------------------------------------------------------|------|-----|---------------:|---------------:|-------|
| MicrometerCounterBenchmark.notCachedTaglessCounter     | avgt | 5   |        177.100 |  ±      23.376 | ns/op |
| MicrometerCounterBenchmark.cachedTaglessCounter        | avgt | 5   |         46.078 |  ±       1.516 | ns/op |
| MicrometerCounterBenchmark.notCachedTaggedCounters     | avgt | 5   |  496434854.701 |  ± 5020725.765 | ns/op |
| MicrometerCounterBenchmark.enumMapCachedTaggedCounters | avgt | 5   |   75517031.385 |  ± 2542030.000 | ns/op |
| MicrometerCounterBenchmark.hashMapCachedTaggedCounters | avgt | 5   |   93097015.965 |  ± 2702539.877 | ns/op |

The results are similar when testing with 64 threads.

### Links

* https://github.com/micrometer-metrics/micrometer/wiki/Performance:-criticality-of-code-paths
* https://vertx.io/blog/micrometer-metrics-performance/
* https://vertx.io/docs/vertx-micrometer-metrics/java/ \[haven't read]
* https://docs.micrometer.io/micrometer/reference/concepts/meter-provider.html