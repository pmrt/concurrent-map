# Concurrent-map

Concurrent sharded map forked and based on the popular project [here](https://github.com/orcaman/concurrent-map). The documentation and API is the same except for the documented changes here.

## Changes

- Changed RWMutex to Mutex.
- Added NewWithConcurrencyLevel() to specify level of concurrency, which determines the number of shards which will be used.
- Prevents false sharing.

This version of concurrent sharded map is optimized for use cases which require many new disjoint keys and the number of reads/writes is similar.

## Why change RWMutex to Mutex?

Recommended reading:
- ReadWriter locks and their lack of applicability to finegrained synchronization - https://joeduffyblog.com/2009/02/11/readerwriter-locks-and-their-lack-of-applicability-to-finegrained-synchronization/
- ReadwriteLock java docs, which explain differences of RWMutexes and plain Mutexes - https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/locks/ReadWriteLock.html

Golang 1.9 brought sync.Map but the use cases are limited: 

> The Map type is optimized for two common use cases: (1) when the entry for a given
key is only ever written once but read many times, as in caches that only grow,
or (2) when multiple goroutines read, write, and overwrite entries for disjoint
sets of keys. In these two cases, use of a Map may significantly reduce lock
contention compared to a Go map paired with a separate Mutex or RWMutex.

So sync.Map is ideal for append-only use cases, like caches but it is [not that good for plain new disjoint keys](https://github.com/golang/go/issues/21035) or when the number of reads/writes is similar like in filters or rate-limiters which are constantly storing new objects (e.g.: IPs) and checking if they are present in the filter right before writing them otherwise. This is mainly because sync.Map uses a single mutex for the entire internal map structure and when using structures which are not read predominant, sync.Map never promotes the elements to the read-only map, which is way faster.

Here is when sharded concurrent maps become handy. Splitting the map structure into shards where every shard has its own mutex is a good way to reduce contention when there are many new disjoint keys. Elements are stored in a shard depending on the hash which, with the appropiate hash function, are expected to be equally distributed across the map. Now reading/writing keys only locks elements in the same shard.

Now, the original concurrent-map version used ReadWrite mutexes and a RWMutex is not free. It is more expensive than a plain Mutex because it has to keep track of the readers and writers, which leads to cache contention (hardware contention). Sometimes this cost is worth it, especially when there are more readers than writers like in use cases where the data is created and then searched frequently or when the duration of the lock is relatively long and expensive. But in cases where updates are frequent and locks are very short a simple mutex is likely to outperform ReadWrite mutexes. 

And that's why where using a Mutex instead of a ReadWrite mutex. So, this concurrent-map is better suited for cases when you need frequent updates and new disjoint keys are created. Also, the duration of locks in this type of maps is always very short so the overhead of a ReadWriter mutex can exceed the execution cost.

## Benchmarks

> old = original versio; new = this version

```name                              old time/op  new time/op  delta
Items-4                           4.20ms ±17%  3.79ms ± 5%    -9.81%  (p=0.000 n=49+44)
MarshalJson-4                     9.74ms ±10%  9.35ms ± 8%    -4.03%  (p=0.000 n=50+49)
Strconv-4                         37.3ns ± 1%  37.3ns ± 2%      ~     (p=0.722 n=45+47)
SingleInsertAbsent-4               456ns ± 5%   432ns ± 5%    -5.32%  (p=0.000 n=49+50)
SingleInsertAbsentSyncMap-4       1.08µs ± 5%  1.06µs ± 5%    -2.11%  (p=0.000 n=49+47)
SingleInsertPresent-4             49.1ns ± 1%  33.7ns ± 6%   -31.27%  (p=0.000 n=43+45)
SingleInsertPresentSyncMap-4       104ns ±11%   107ns ±21%      ~     (p=0.885 n=49+48)
MultiInsertDifferentSyncMap-4     5.98µs ± 7%  6.19µs ±13%    +3.54%  (p=0.000 n=50+48)
MultiInsertDifferent_1_Shard-4    4.16µs ± 3%  3.32µs ± 4%   -20.33%  (p=0.000 n=49+43)
MultiInsertDifferent_16_Shard-4    833ns ± 4%   713ns ± 9%   -14.41%  (p=0.000 n=44+48)
MultiInsertDifferent_32_Shard-4    777ns ± 3%   690ns ±21%   -11.20%  (p=0.000 n=45+50)
MultiInsertDifferent_256_Shard-4  1.27µs ± 8%  1.26µs ±14%      ~     (p=0.684 n=46+50)
MultiInsertSame-4                 3.37µs ± 3%  2.50µs ± 2%   -25.84%  (p=0.000 n=48+47)
MultiInsertSameSyncMap-4          3.59µs ± 4%  3.57µs ± 4%      ~     (p=0.153 n=50+49)
MultiGetSame-4                     587ns ± 6%  1583ns ± 6%  +169.53%  (p=0.000 n=38+45)
MultiGetSameSyncMap-4              496ns ±11%   408ns ± 6%   -17.78%  (p=0.000 n=45+46)
MultiGetSetDifferentSyncMap-4     9.80µs ± 8%  8.11µs ±10%   -17.18%  (p=0.000 n=50+49)
MultiGetSetDifferent_1_Shard-4    5.01µs ± 3%  5.68µs ± 3%   +13.33%  (p=0.000 n=45+50)
MultiGetSetDifferent_16_Shard-4   6.24µs ±11%  1.38µs ±11%   -77.84%  (p=0.000 n=49+49)
MultiGetSetDifferent_32_Shard-4   5.10µs ±26%  1.29µs ± 8%   -74.71%  (p=0.000 n=47+47)
MultiGetSetDifferent_256_Shard-4  1.67µs ±11%  1.29µs ± 9%   -22.67%  (p=0.000 n=45+46)
MultiGetSetBlockSyncMap-4         2.44µs ± 6%  1.92µs ± 3%   -21.52%  (p=0.000 n=45+42)
MultiGetSetBlock_1_Shard-4        5.58µs ±21%  4.59µs ± 3%   -17.78%  (p=0.000 n=50+48)
MultiGetSetBlock_16_Shard-4       5.57µs ± 8%  1.07µs ±13%   -80.73%  (p=0.000 n=46+46)
MultiGetSetBlock_32_Shard-4       4.40µs ± 7%  0.93µs ±27%   -78.95%  (p=0.000 n=48+49)
MultiGetSetBlock_256_Shard-4      1.07µs ± 6%  0.81µs ± 4%   -23.93%  (p=0.000 n=43+39)
Keys-4                            1.83ms ± 2%  1.67ms ± 2%    -8.48%  (p=0.000 n=42+45)```

As expected, the performance seems generally better for cases where reads and writes are used concurrently like MultiGetSetDifferent and MultiGetSetBlock while the performance is generally the same for map.Sync because it is using the same implementation and is slower for MultiGetSame, where there are multiple concurrent reads.

So, in conclusion, use this version if you need frequent updates and/or you create many new disjoint keys or if your number of reads/writes is similar. If the proportion of readers to writers is way higher in the read side you're probably better off sticking with sync.Map, especially if you have device has many threads.
