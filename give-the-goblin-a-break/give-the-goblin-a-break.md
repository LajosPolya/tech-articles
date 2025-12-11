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

