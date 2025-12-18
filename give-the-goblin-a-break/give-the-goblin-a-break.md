# Give the memory goblin a break! (Excessive memory consumption)
(The performance impact of observability and how we its cost can be reduced)
The article dives into finding the balance between fine grained / high-resolution observability and its performance hit.

Deep inside your computer lives a small creature called the Memory Goblin. It works tirelessly within your hardware to manage your computer’s memory. 
When you open a program, write a document, or watch a video; in the background, the goblin within your machine, streams and stores data in memory.

Memory (RAM) is designed (laid out) as a vast matrix of drawers, a large cabinet. Each drawer represents one byte of data.
When every drawer is empty, the goblin inserts data starting at the first drawer, and continues sequentially, until all data is stored in a drawer.
When every drawer contains an item, the goblin opens the least recently used drawer. Instead of removing the old data, it instead compresses it down with the new data until the former turns into dust, and voilà, the drawer only contains the new the data.
This goblin works relentlessly, never stopping for a break, vacation, or even a pay raise. But if it works too hard it’ll eventually overheat, so we must give it a break.


The idea of a memory goblin was brought to me by my most influential high school teacher, Anthony Viola, over 15 years ago. He mentioned that a prominent journal published an article about how computer memory works, and if I recall correctly the article described memory management by a goblin managing drawers or shelves. I don’t remember the article’s title, the publisher, or the author. I tried looking for it using Gemini and ChatGPT but they didn’t return any useful information.


## A note on observability

For the past two years, I’ve been working on an ad exchange written in Java. It’s an incredible piece of software which transacts millions of auctions every second in an environment where every millisecond counts.

The observability of a system can be directly linked to its reliability. If you can't see and error then how can you fix it? If you can't see trends then how can you use them to your advantage?
On the exchange, we have thousands of metrics, with hundreds of millions of measurements. We use these measurements to track error rates, ad spend, response times, number of messages produced, number of bids, etc.
Each metric plays an important role in observing the health of the exchange. For example, a sharp increate in the error rate of an operation can bring to light a bug. An increase in response times can hint to infrastructure scaling issues.
But metrics aren't free.
The tradeoff between performance and observability is delicate. The more data we have, the better we can make inference about state/health of the application, but this comes at a cost. 
Managing too many metrics can be detrimental to performance especially if the weight of recording a metric is not considered.


## Micrometer 
Within the exchange, we use the Micrometer observability facade, the Prometheus flavour.
Micrometer supports many types of meters; counters, gauges, timers, and distribution summaries. For simplicity, I'll use counters since they're used most often and are the simplest to understand.

At its simplest, `Counter` is, well, a counter. It keeps the count of "something".

Before continuing the reason must understand a couple of things.
1. `MeterRegistry` is the interface provided by Micrometer to produce meters, such as a counter.
2. In most modern applications, an instance of `MeterRegistry` is injected into the application, and for simplicity we're going to assume the same happens here.

For example;

```java
/**
 * A service, responsible for making an ad request. A {@link io.micrometer.core.instrument.Counter} is initialized every
 * time a request is made to keep count of the number of requests made. As you can imagine, this can be quite inefficient.
 */
public class AdRequestService {

    private final MeterRegistry meterRegistry;

    public AdRequestService(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public fetch() {
        // contains code to make an ad request

        meterRegistry.counter("number_of_request").increment();
    }
}
```
The code snippet above creates a counter and increments it. This operation is simple and relatively benign, but a lot is going on under the hood.
Every time a counter is initialized in this way, Micrometer creates an object to sort and store all the counter's tags. Then a `builder` object is built to hold the meter's name and tags.
Then in order to register the counter, an ID is constructed and used as a key to store the counter in a map. 
This single counter has the potential to create a tremendous amount of memory pressure if it created often enough. 
Some of the exchange's counters are created hundreds of millions of times per minute, accounting for about 3% of total memory usage. This is huge considering most of this memory pressure could be easily removed, giving the Garbage Collector (GC) a break.

### Efficient patterns for using meters

In the exchange, I was able to reduce Micrometer's total memory usage from 3% to 1%, reducing memory usage by 2%.
The rest of the article described the patterns I used to create counters efficiently to reduce CPU and memory usage.

#### A simple counter

I already showed the most simple way to create a counter.

```java
meterRegistry.counter("number_of_request").increment();
```
As mentioned above, this line of code can be quite inefficient when called a large number of times. To mitigate the creation of copious amounts of memory, create the counter once, store a reference to it in an instance variable, and use that refence when incrementing the counter.


```java
/**
 * A Service used to make an ad request. A Counter is initialized once in the constructor and a reference to it is re-used
 * to keep count of the number of requests made.
 */
public class AdRequestService {
    
    private final Counter numberOfAdRequest;
    
    public AdRequestService(MeterRegistry meterRegistry) {
        this.numberOfAdRequest = meterRegistry.counter("number_of_request");
    }
    
    public fetch() {
        // contains code to make an ad request
        
        numberOfAdRequest.increment();
    }
}
```

The code above is much more efficient than the previous example because the counter is only created once, releasing memory pressure.

#### Counters with enum tags

```java
public void increment(State state) {
    meterRegistry.counter("counter", "state", state.name()).increment();
}

public enum State {
    SENT,
    RECEIVED,
    REQUEST_NOT_RECEIVED,
    RESPONSE_NOT_RECEIVED,
}
```

```java
public class SimpleCounter {

    private final MeterRegistry meterRegistry;
    private final Map<EnumState, Counter> counters;

    public SimpleCounter(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        counters = getCounters();
    }

    private Map<EnumState, Counter> getCounters() {
        // note that we're using an EnumMap
        Map<EnumState, Counter> tempCounters = new EnumMap<>(EnumState.class);
        for(EnumState state : EnumState.values()){
            tempCounters.put(state, meterRegistry.counter("counter", "state", state.name()));
        }
        return tempCounters;
    }

    public void increment(EnumState state) {
        counters.get(state).increment();
    }

    public enum State {
        SENT,
        RECEIVED,
        REQUEST_NOT_RECEIVED,
        RESPONSE_NOT_RECEIVED,
    }
}
```


### JMH

Using JDK Mission Control, in production, I was able to calculate savings of about ~2% of memory. But in order to calculate CPU usage, I decided to setup a JMH test harness.

```
Benchmark                                      Mode  Cnt         Score         Error  Units
JMHSample_03_States.measureShared              avgt    5         9.387 ±       0.931  ns/op
JMHSample_03_States.measureSharedCreate        avgt    5        13.414 ±       1.899  ns/op
JMHSample_03_States.measureSharedCreateEnum    avgt    5  29577438.730 ± 2495721.829  ns/op
JMHSample_03_States.measureSharedEnum          avgt    5   5933216.139 ±  443411.755  ns/op
JMHSample_03_States.measureUnshared            avgt    5         8.757 ±       1.166  ns/op
JMHSample_03_States.measureUnsharedCreate      avgt    5        13.354 ±       0.803  ns/op
JMHSample_03_States.measureUnsharedCreateEnum  avgt    5  30121077.236 ± 6096476.216  ns/op
JMHSample_03_States.measureUnsharedEnum        avgt    5   6040832.152 ±  198174.826  ns/op
```

### Links

* https://github.com/micrometer-metrics/micrometer/wiki/Performance:-criticality-of-code-paths
* https://vertx.io/blog/micrometer-metrics-performance/
* https://vertx.io/docs/vertx-micrometer-metrics/java/ \[haven't read]
* https://docs.micrometer.io/micrometer/reference/concepts/meter-provider.html