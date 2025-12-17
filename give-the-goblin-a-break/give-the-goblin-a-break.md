# Give the memory goblin a break! (Excessive memory consumption)
(The performance impact of observability and how we can reduce its cost)
The article dives into finding the balance between fine grained / high-resolution observability and its performance hit.

The Memory Goblin works relentlessly/tirelessly to manage your computer’s memory. 

1. When you open a program, write a document, or watch a video; without even knowing it, the goblin is instructed to create all the necessary objects and places then at new addresses.
2. When you open a program, write a document, or watch a video; in the background, the goblin within you machine is instructed to stream and store your data in memory.

\[// maybe make the use of LRU more obvious to make the writing more concrete and less abstract?]

1. Items are never removed from the memory, instead, when memory becomes full, the Goblin uses the latest and greatest cache replacement policy/algorithm to choose an address where it can pack down its contained object and put a new object on top of it. When the computer runs out of memory, the goblin use the greatest and latest cache replacement policy/algorithm to choose an address to pack down and put a new object on top of it.
2. The goblin lives in a massive wearhouse, managing an almost never ending wall of drawers. When room is plenty, the goblin stores your data as it pleases (or maybe have an analogy for LRU algorithm... stores your data next to where it left off). But when memory is full, it finds the least recently used memory blocks (or any other cache replacement policy/algorithm) and packs the old data down until it becomes dust and fill the drawer with the most recent data. (god forbid it go to the hard-drive)
3. The goblin packs the memory (data?) down until it turn into dust, making room for new data \[storage]

Memory isn't infinite but the drawers live on infinitely 

This goblin works \[tirelessly, is this redundant since I said "without a break"] never stopping for a break, vacation, or even a pay raise. But if it works too hard it’ll eventually melt, so we must give it a break (reduce it's workload). But how?


Not sure I like this: In true capitalist fashion, by reducing memory consumption we were able to increase ad request throughput, making the goblin more efficient, and working harder 


The idea of a memory goblin was brought to me by my most influential high school teacher, Anthony Viola, over 15 years ago. He mentioned that a prominent journal published an article about how computer memory works, and if I recall correctly the article described memory management by a goblin managing drawers or shelves. I don’t remember the article’s title, the publisher, or the author. I tried looking for it using Gemini and ChatGPT but they didn’t return any useful information.


## A note on observability

The observability of a system can directly be linked to its reliability. If you can't see and error then how can you fix it? If you can't see trends then how can you use them to your advantage?
One the exchange, we have thousands of metrics, with hundreds of millions of measurements (data-points?).

The tradeoff/balance between performance and observability is delicate. The more data we have, the better we can make inference about state/health of the exchange, but this comes at a cost. Managing too many metrics can be detrimental to performance especially if we don't keep in mind the weight of each counter or metric operation.


## Micrometer 
Within the exchange, we use the Micrometer observability facade, the Prometheus flavour. Micrometer supports many types of metrics/measurements but most use the same interface so I'll stick to `Counter` for simplicity.
At its simplest, `Counter` is, well, a counter. It keeps track of the number of times it was incremented.
For example;

```java
// Do I have to explain what MeterRegistry is? Or can I just simply say "MeterRegistry is the factory used to create and store meters.
/**
 * A Service used to make an ad request. A Counter is initialized every time a request is made to keep count of the 
 * number of requests made. As you can imagine, this can be quite inefficient.
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
For every counter, or any other type of meter, Micrometer must create an object to hold all the tags, then the tags must be sorted. Then a `builder` object is built to hold the meter's name and tags.
Then in order to register the meter, and ID must be constructed and stored in a map. This single counter has the potential to create a tremendous amount of memory pressure if it created often enough. 
Some of our counters may be created hundreds of millions of times every minute, accounting for about ~3% of total memory usage, which is huge considering most of this memory pressure could be easily removed, giving the Garbage Collector (GC) a break.

### Efficient patterns for using meters

#### A simple counter

I've already showed the most simple way to create a counter.

```java
meterRegistry.counter("number_of_request").increment();
```
As said above, this line of code can be quite inefficient when called a large number of times. To mitigate the creation of copious amount of memory, create the counter once and increment a reference to it.


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

#### Counters with fixed tags

Another common pattern is to use a set of fixed tags. I'm simplifying the rest of the examples by removing the initialization of `MeterRegistry`.


```java
public fetch() {
    // make ad request
    boolean success = makeRequest();
    
    if(success) {
        meterRegistry.counter("number_of_request", Tags.of("status", "success")).increment();
    } else {
        meterRegistry.counter("number_of_request", Tags.of("status", "failed")).increment();
    }
}
```
The above example allows us to filter on `status`, so we get a count of the number of successful vs failed ad requests.


```java
public class AdRequestService {
    
    private final Counter numberOfSuccessfulAdRequest;
    private final Counter numberOfFailedAdRequest;
    
    public AdRequestService(MeterRegistry meterRegistry) {
        this.numberOfSuccessfulAdRequest = meterRegistry.counter("number_of_request", Tags.of("status", "success"));
        this.numberOfFailedAdRequest = meterRegistry.counter("number_of_request", Tags.of("status", "failed"));
    }
    
    public fetch() {
        // make ad request
        boolean success = makeRequest();

        if(success) {
            numberOfSuccessfulAdRequest.increment();
        } else {
            numberOfFailedAdRequest.increment();
        }
    }
}
```

#### Counters with enum tags



### JMH

```
Throughput (64 Threads) (Score = ops/ns over the 64 threads)
Benchmark                                   Mode  Cnt  Score   Error   Units
JMHSample_03_States.measureShared          thrpt    5  1.168 ± 0.178  ops/ns
JMHSample_03_States.measureSharedCreate    thrpt    5  0.360 ± 0.003  ops/ns
JMHSample_03_States.measureUnshared        thrpt    5  1.247 ± 0.049  ops/ns
JMHSample_03_States.measureUnsharedCreate  thrpt    5  0.362 ± 0.006  ops/ns

Average Time (64 Threads)
Benchmark                                  Mode  Cnt    Score   Error  Units
JMHSample_03_States.measureShared          avgt    5   49.784 ± 3.977  ns/op
JMHSample_03_States.measureSharedCreate    avgt    5  178.361 ± 1.538  ns/op
JMHSample_03_States.measureUnshared        avgt    5   50.473 ± 0.229  ns/op
JMHSample_03_States.measureUnsharedCreate  avgt    5  179.979 ± 5.793  ns/op

Throughput (1 Thread)
Benchmark                                   Mode  Cnt  Score   Error   Units
JMHSample_03_States.measureShared          thrpt    5  0.117 ± 0.005  ops/ns
JMHSample_03_States.measureSharedCreate    thrpt    5  0.077 ± 0.003  ops/ns
JMHSample_03_States.measureUnshared        thrpt    5  0.117 ± 0.002  ops/ns
JMHSample_03_States.measureUnsharedCreate  thrpt    5  0.076 ± 0.003  ops/ns

Average Time (1 Thread)
Benchmark                                  Mode  Cnt   Score   Error  Units
JMHSample_03_States.measureShared          avgt    5   8.575 ± 0.154  ns/op
JMHSample_03_States.measureSharedCreate    avgt    5  13.624 ± 0.589  ns/op
JMHSample_03_States.measureUnshared        avgt    5   8.604 ± 0.202  ns/op
JMHSample_03_States.measureUnsharedCreate  avgt    5  13.158 ± 0.430  ns/op
```

### Links

* https://github.com/micrometer-metrics/micrometer/wiki/Performance:-criticality-of-code-paths
* https://vertx.io/blog/micrometer-metrics-performance/
* https://vertx.io/docs/vertx-micrometer-metrics/java/ \[haven't read]
* https://docs.micrometer.io/micrometer/reference/concepts/meter-provider.html