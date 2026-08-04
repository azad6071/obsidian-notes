

Git
Python
FastAPI
Redis Streams - Message Broker
React
AWS
EMR
ARGO-CD
Kubernetes

When data is collected in batches, it is almost always due to some manual step or lack of digitization or is a historical relic left over from the automation of some non-digital process.

Log-Table Duality

Reverse Proxy: Sit's in front of Backend Servers Hiding there identity. Can sit in front of single server.
Usage:  
	Caching
	Security
	Privacy.
Forward Proxy: Sit's in front of Client, hiding client Identify.
Usage: 
	Individuals to hide their IP
	Enterprises to block certain websites.
Load Balancer: Evenly forward traffic to Backend Servers.
API Gateway: Reverse proxy.
Usage: 
	Forward Traffic to correct micro-service.
	Rate-Limiting and Throttling
	Authentication and Authorization
	Protocol Translation

| Requirement                | Recommended Algorithm  | Storage                | Rationale                    |
| -------------------------- | ---------------------- | ---------------------- | ---------------------------- |
| Simple API (basic)         | Fixed Window           | Redis                  | Simplicity. Good enough.     |
| Financial transactions     | Sliding Window Log     | Redis + Sorted Sets    | Accuracy is mandatory.       |
| Social media feed (bursts) | Token Bucket           | Redis                  | Handles viral spikes.        |
| Database write throttling  | Leaky Bucket           | Queue (e.g., RabbitMQ) | Smooths out write load.      |
| Multi-tenant SaaS          | Sliding Window Counter | Redis Cluster          | Fairness + memory efficient. |
|                            |                        |                        |                              |
Connection Pooling
Exponential Backoff
Redundancy
Kafka can handle 1000 messages/sec/partition
Pagination
1 Database approximately serves 12K concurrent requests at max.

_We will use Apache Kafka as the central event bus to decouple the microservices. There will be three topics: `order-events`, `payment-events`, and `user-activity`. Each topic will have 24 partitions (to allow up to 24 concurrent consumers) with a replication factor of 3 spread across AZs. The `order-events` topic will be partitioned by `order_id` to preserve order per order. Data will be retained for 7 days in the hot tier and archived to S3 after that._

_The order service produces to `order-events` with `acks=all` to guarantee durability. The analytics service and notification service will be separate consumer groups reading from the same topic with at-least-once semantics. Failed messages (after 3 retries) will be sent to a dead-letter queue topic._

_Peak throughput is estimated at 15,000 messages/sec (~15 MB/sec). With 7-day retention, we need ~9 TB of storage (15 MB/s * 7 days * 3x replication). The cluster will consist of 5 brokers managed in KRaft mode, with Prometheus monitoring for consumer lag and broker health._