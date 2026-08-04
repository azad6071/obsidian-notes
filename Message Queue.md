We use queue's in following scenarios.

Asyn Processing
Decoupling
Smooth Spike
Reliability
Backpressure


Things that matter:
Ordering: Credit 100 -> Withdraw 50.
			1.      			2.  
If a different worker consumes 2 first then it may say "Not Sufficient Amount"

Distribution:
Social Media App : Queue processing events. If we partition by City for workers, then it's a BAD strategy as Bangalore may have very high no of users compared to Surat.



There are 3 types of Delivery Guarantees
1. At most  once - when we can offord to lose messages
2. At lest once - (This is the system we generally try to make)
3. Exactly once


### Why we assume queues to be reliable:

We do not assume queue are reliable. We verify and configure them to be reliable. By
1. Require ACK from workers
2. Persist Message to disk
3. Replicate messages on multiple servers

Queue just buys us time. It doesn't make things faster.

Backpressure via TCP is not a "message" or a "flag." It is a **mechanical stall**—a direct physical consequence of a buffer filling up. The producer doesn't choose to slow down; it simply **cannot proceed** until the buffers drain.

In every TCP ACK, sent back to Producer, there's a field **Advertised Window.**
When consumer receive buffer fills up, it sends Window = 0.

### How to reason about the choice

1. Replayability: If you want this choose Kafka or Redis Streams. In RabbitMQ once a message is ACKed it gets deleted from the queue.
2. Scale: If very large scale - Kafka. Otherwise Redis or RabbitMQ works.
3. Advanced Routing: RabbitMQ. It supports advanced routing mechanism. For example routing-key (error, warning, info)
Example:
```
A log processing system where a queue binds with `*.error.#`. It will receive messages with routing keys like `api.error.database` or `frontend.error.404`.

A system that needs different handling based on message metadata (e.g., `format: pdf` AND `size: large`), where the routing logic is too complex for a topic pattern.

A multi-tenant application where all events for a specific `orderId` must be processed in order, but events for different tenant IDs can be processed in parallel across multiple queues.
```

