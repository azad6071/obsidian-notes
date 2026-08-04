Interfaces are meant to be pure **contracts** - they define what an object can do, not how it does it or what data it stores i.e. we can't add variables in interfaces. For that we need to use **abstract class**. 

Every field declaration in the body of an interface is implicitly public, static, and final. It is permitted to redundantly specify any or all of these modifiers for such fields.
```java

public interface item{
	String id; //Compilation Error
	private final String price; // Comilation error
	final String name; // OK
	static final String name; // OK Redundant specification
	public static final String name; // OK Redundant specification

	void start(); //OK
}
```

The state of the item cannot be trusted until you hold the lock!
 

```java 
public void transfer(Account from, Account to, double amount) {

	ReentrantLock fromLock = from.lock.tryLock();
	ReentrantLock toLock = to.lock.tryLock();
	
	if(fromLock && toLock){
		try{
			from.withdraw(amount);
			to.deposit(amount);
			}finaly{
			from.lock.unlock();
			to.lock.unlock();
			}
		}
	if(fromLock) from.lock.unlock();
	if(toLock) to.lock.unlock();

}
```

ReentrantReadWriteLock

- **Read Lock:** Multiple threads can hold this lock simultaneously, provided no thread holds the write lock.
    
- **Write Lock:** Only a single thread can hold this lock, completely excluding all readers and other writers.
You cannot seamlessly upgrade a lock from a read lock to a write lock. If a thread reads data, decides it needs to modify it, and tries to acquire the write lock without first releasing its read lock, it will cause an immediate **deadlock** with itself.

## StampedLock

```java

public class StockPrice {  
    private double price;  
    private long lastUpdatedTime;  
  
    private final StampedLock sl = new StampedLock();  
  
    public void updatePrice(double newPrice, long timestamp) {  
        long stamp = sl.writeLock();  
        try{  
            price = newPrice;  
        }finally {  
            sl.unlockWrite(stamp);  
        }  
    }  
  
    public double[] getPriceData() {  
        long stamp = sl.tryOptimisticRead();  
        double currentPrice = price;  
        if(!sl.validate(stamp)){  
            stamp = sl.readLock();  
            try{  
                currentPrice = price;  
            }finally {  
                sl.unlockRead(stamp);  
            }  
        }  
        return new double[]{currentPrice};  
    }  
}
```

ExecutorService → Task execution and thread management (No of workers)
Semaphore → Resource access control and rate limiting (Concurrent access to a resource to at-most N threads)

A static nested class does NOT need an instance of the outer class,
but an inner class ALWAYS needs one.

```java

class Outer {
    static int x = 10;

    static class Helper {
        void print() {
            System.out.println(x);
        }
    }
    
	 class Inner {
        void print() {
            System.out.println(x);
        }
    }
}
```

Static nested class can only access static member variables of parent class.

```java
Outer.Helper helper = new Helper();
helper.print();

Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();
inner.print();
```

---
Java code is executed sequentially within a thread, but the Java Memory Model allows the JVM and CPU to reorder or delay the visibility of memory writes across threads unless we syncronize explicitly. Without a happens-before relationship, other threads may observe writes in a different order or not observe them at all, even though the original thread executed the code line by line.

**Synchronized

When a thread exits a synchronized block, all its writes are flushed to main memory, and when another thread enters a synchronized block on the same lock, its local caches are invalidated and reloaded. This is why synchronization guarantees visibility and prevents stale reads.

So synchronized:
`synchronized` is not just mutual exclusion.  
It is also a _memory visibility and cache-coherence contract_.



## Comparable

```java

public class Player implements Comparable<Player> {

    // same as before

    @Override
    public int compareTo(Player otherPlayer) {
        return Integer.compare(getRanking(), otherPlayer.getRanking());
    }

}
```

Now we can use Collections.sort(playerList); 

## Comparator

```java

public class PlayerComparator implements Comparator<Player> {

    @Override
    public int compare(Player p1, Player p2) {
        return Integer.compare(p1.getRanking(), p2.getRanking());
    }

}


PlayerRankingComparator playerComparator = new PlayerRankingComparator();
Collections.sort(footballTeam, playerComparator);

```


Sometimes, we don't have the luxury to modify original class definition, in that case, comparator becomes useful.
Moreover, we Can define multiple comparators in case we're using comparators.


## Future

When we invoke a asynchronous task., then it acts as a reference for the result of this task.

```java

Future<T> future = asyncProcess.method();
future.get(); 
// Blocking Request - Invocation is paused till result is available


/*Example*/

Future<Integer> future = executor.submit(() -> {
	Thread.sleep(2000);
    return 10 + 20;
});

Integer result = future.get();  // blocks until result is ready


```


## Runnable
```java
@FunctionalInterface  
public interface Runnable {  
    /**  
     * When an object implementing interface {@code Runnable} is used  
     * to create a thread, starting the thread causes the object's     * {@code run} method to be called in that separately executing  
     * thread.     * <p>  
     * The general contract of the method {@code run} is that it may  
     * take any action whatsoever.     *     * @see     java.lang.Thread#run()  
     */    public abstract void run();  
}
```

## Callable
```java
@FunctionalInterface  
public interface Callable<V> {  
    /**  
     * Computes a result, or throws an exception if unable to do so.     *     * @return computed result  
     * @throws Exception if unable to compute a result  
     */    V call() throws Exception;  
}
```


How Threads communicate in Java?
Using shared memory and coordination mechnisms.

When a method has exception thrown in it's signature, then caller is forced to handle it or re-throw.

If it's a checked exception, either catch it, or throw it.

What if you catch but never throw?
-- you are swallowing the exception (Bad Practice)

https://newsletter.francofernando.com/p/designing-a-url-shortener

why in a system only 3 of 2 can be available in CAP

## 1. Collision Handling

- **What happens when two different long URLs produce the same hash?** How do you detect and resolve this (chaining, linear probing, rehashing, append salt)?
    
- **How do you design the hash function to minimize collision probability** given a fixed short code length (e.g., 6-7 characters)?
    
- **Birthday paradox impact**: With 62 chars (a-z, A-Z, 0-9), how many URLs can you store before collision probability exceeds 1%? (Calculate: ~√(2 × 62^6 × 0.01) ≈ 8.4 million URLs)
    
- **Should you use deterministic collision resolution** (always append a counter) or **probabilistic** (regenerate with different hash inputs)?
    

## 2. Hash Function Selection

- **Why not use cryptographic hashes like SHA-256 or MD5?** What are the computational cost trade-offs?
    
- **What properties make a hash "good enough" for URL shortening?** (Speed > cryptographic security)
    
- **How do non-cryptographic hashes compare?** (MurmurHash, xxHash, CityHash, FarmHash) - which has lowest collision rate and fastest speed?
    
- **Base conversion vs hashing**: Why do some services use base-62 encoding of database IDs instead of hashing? What are the advantages?
    

## 3. Uniformity and Distribution

- **How do you ensure uniform distribution** across your short code space to avoid hotspots in database sharding?
    
- **What statistical tests would you run** to verify your hash distributes evenly? (Chi-square test, Kolmogorov-Smirnov)
    
- **How does input URL structure affect distribution**? If users submit mostly similar URLs (same domain, different paths), how does your hash handle that?
    

## 4. Custom Transformation Techniques

- **What's the impact of reversing the URL string before hashing?** Does it improve distribution for URLs with common prefixes?
    
- **How would you incorporate domain name weighting** - treat "[google.com](https://google.com/)" differently than "[randomblog.blogspot.com](https://randomblog.blogspot.com/)"?
    
- **Should you strip query parameters** (e.g., `?utm_source=...`) before hashing? How to canonicalize URLs to avoid different hashes for identical content?
    

## 5. Performance at Scale

- **Computational cost analysis**: For 10,000 URLs/second, what hash throughput do you need? Compare MD5 (~400 MB/s) vs xxHash (~10 GB/s) vs CRC32 (~500 MB/s)
    
- **Memory footprint**: Does your hash function require lookup tables (e.g., CRC32) and does that matter in high-concurrency environments?
    
- **How does hash computation compare to database write latency**? Is hashing ever the bottleneck?
    

## 6. Length vs Collision Trade-offs

- **Short code length optimization**: Given your projected URL volume (e.g., 1 billion total), what's the minimum characters needed to keep collision probability under 0.1%?
    
- **Variable-length codes**: Would you ever use different lengths (e.g., 6 chars for popular URLs, 8 chars for long tail)? How does that affect hash design?
    
- **What's the amortized cost of length increase?** Adding 1 character expands keyspace by 62× but increases storage/cache overhead.
    

## 7. Advanced Techniques

- **Two-tier hashing**: First-level hash maps to bucket, second-level within bucket - how does this affect lookup speed?
    
- **Perfect hashing for URL shorteners**: Is it possible to precompute a collision-free function given a fixed set of URLs? (Yes, using tools like `gperf` for static sets, but URL set is dynamic)
    
- **Consistent hashing**: Could you use consistent hashing to distribute short codes across multiple database shards while minimizing reshuffling?
    

## 8. Security Considerations

- **Can attackers reverse your hash** to enumerate all shortened URLs? How do you prevent this? (Add secret salt, use keyed hash like HMAC)
    
- **Hash extension attacks**: If your hash is predictable (e.g., base62(CRC32(url))), can an attacker guess other valid short codes?
    
- **How do you protect against hash flooding attacks** where millions of URLs deliberately collide with the same hash?
    

## 9. Real-World Design Decisions

- **Why does YouTube short code (e.g., `dQw4w9WgXcQ`) look random but isn't a pure hash?** (They use base-64 encoding of auto-increment IDs with obfuscation)
    
- **TinyURL vs [bit.ly](https://bit.ly/) approach**: Research and compare their actual hash strategies (published in engineering blogs)
    
- **Database index implications**: Hash collisions cause secondary lookups - how does this affect your query plan and index design?
    

## 10. Testing and Validation

- **How would you empirically measure collision rates** for your chosen hash function with real URL data from production logs?
    
- **What's the worst-case collision scenario** (e.g., all URLs from the same domain with numeric IDs) and how does your hash perform?
    
- **A/B test design**: How would you compare two hash functions in production for collision rate and latency?
    

## Example Problem to Solve:

> Design a hash function that takes a URL string and returns a 7-character short code using [a-zA-Z0-9]. Requirements: <50ns average compute time, <1% collision rate at 10M URLs, uniform distribution for sharding. Which hash function do you choose and how do you handle collisions?

These questions test deep understanding of hash function theory, real-world constraints, and distributed systems design.
 