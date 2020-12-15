[![Build Status](https://travis-ci.org/DanielYWoo/fast-object-pool.svg?branch=master)](https://travis-ci.org/DanielYWoo/fast-object-pool)

fast-object-pool
================
FOP, a lightweight partitioned object pool, you can use it to pool expensive and non-thread-safe objects like thrift clients etc.

Github repo: https://github.com/DanielYWoo/fast-object-pool

Site page: https://danielw.cn/fast-object-pool/

Why yet another object pool
--------------

FOP is implemented with partitions to avoid thread contention, the performance test shows it's much faster than Apache commons-pool. 
This project is not to replace Apache commons-pool, this project does not provide rich features like commons-pool, this project mainly aims on:
1. Zero dependency
2. High throughput with many concurrent requests
3. Less code so everybody can read it and understand it.

Configuration
-------------
First of all you need to create a FOP config:


```java
PoolConfig config = new PoolConfig();
config.setPartitionSize(5);
config.setMaxSize(10);
config.setMinSize(5);
config.setMaxIdleMilliseconds(60 * 1000 * 5);
```


The code above means the pool will have at least 5x5=25 objects, at most 5x10=50 objects, if an object has not been used over 5 minutes it could be removed.

Then define how objects will be created and destroyed with ObjectFactory


```java
ObjectFactory<StringBuilder> factory = new ObjectFactory<>() {
    @Override public StringBuilder create() {
        return new StringBuilder(); // create your object here
    }
    @Override public void destroy(StringBuilder o) {
        // clean up and release resources
    }
    @Override public boolean validate(StringBuilder o) {
        return true; // validate your object here
    }
};
```


Now you can create your FOP pool and just use it


```java
ObjectPool pool = new ObjectPool(config, factory);
try (Poolable<Connection> obj = pool.borrowObject()) {
    obj.getObject().sendPackets(somePackets);
}
```

If you want best performance, you need add disruptor queue to your dependency and use DisruptorObjectPool. 
Note, the DisruptorPool has not been fully tested in production yet.

Shut it down


```java
pool.shutdown();

```

How it works
--------------
The pool will create multiple partitions, in most cases a thread always access a specified partition, 
so the more partitions you have, the less probability you run into thread contentions. Each partition has a 
blocking queue to hold poolable objects; to borrow an object, the first object in the queue will be removed; 
returning an object will append that object to the end of the queue. The idea is from ConcurrentHashMap's segments 
design and BoneCP connection pool's partition design. This project started since 2013 and has been deployed to many projects without any problem.

How fast it is
--------------
The source contains a benchmark test, you can run it on your own machine. On my Macbook Pro (8-cores i9), it's 50 times faster than commons-pool 2.2. 

Test Case 1: we have a pool of 256 objects, use 50/100/400/600 threads to borrow one object each time in each thread, then return to the pool.

The x-axis in the diagram below is the number of threads, you can see Stormpot provides the best throughput. FOP is closely behind Stormpot. Apache common pool is very slow, Furious is slightly faster than it. 
You see the throughput drops after 200 threads because there are only 256 objects, so there will be data race and timeout with more threads.
![](docs/b1-throughput.png?raw=true)
When you have 600 threads contending 256 objects, Apache common pool reaches over 85% error rate (probably because returning is too slow), basically it cannot be used in high concurrency.
![](docs/b1-error-rate.png?raw=true)

The diagram above if you only borrow one object at a time per thread, if you need to borrow two objects in a thread, things are getting more interesting.

Test Case 2: we have a pool of 256 objects, use 50/100/400/600 threads to borrow two objects each time each thread, then return to the pool.

In this case, we could have 600x2=1200 objects required concurrently but only 256 is available, so there will be contention and timeout error. 
FOP keeps error rate steadily but stormpot starts to see 7% error rate with 100 threads. (I cannot test stormpot with 600 threads because it's too slow) 
Furious seems not working because I cannot find a way to set timeout, without timeout I see circular deadlock in Furious, 
so I exclude Furious from the diagram below . Both FOP and Stormpot provides timeout configuration, in the test I set timeout to 10ms. 
FOP throws an exception, Stormpot returns null, so we can mark it as failed (show in the error rate plot). 
Again, Apache common pool reaches almost 80% error rate, basically not working.
![](docs/b2-error-rate.png?raw=true)

The throughput of FOP is also much better than other pools. Interestingly, in this case, Apache common pool is slightly faster than Stormpot. 
I don't know why Stormpot degrade so fast with two borrows in one thread. If you know how to optimize Stormpot configuration for this case please let me know.
![](docs/b2-throughput.png?raw=true)

So, in short, if you can ensure borrow at most one object in each thread, Stormpot is the best choice. If you cannot ensure that, use FOP which is more consistent in all scenarios.

Maven dependency
---------------
To use this project, simply add this to your pom.xml


```xml
<dependency>
    <groupId>cn.danielw</groupId>
    <artifactId>fast-object-pool</artifactId>
    <version>2.1.1</version>
</dependency>
```

If you want disruptor object pool, add this optional dependency

JDK 8,9,10
```xml
<dependency>
    <groupId>com.conversantmedia</groupId>
    <artifactId>disruptor</artifactId>
    <version>1.2.15</version>
</dependency>
```
JDK 11+
```xml
<dependency>
    <groupId>com.conversantmedia</groupId>
    <artifactId>disruptor</artifactId>
    <version>1.2.19</version>
</dependency>
```

JDK 8/11 is required. By default the debug messages are logged to JDK logger because one of the goals of this project is ZERO DEPENDENCY. However we have two optional dependencies, disruptor and SLF4j.
