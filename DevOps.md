
IPV4 IP's have 32 bits
(0-255).(0-255).(0-255).(0-255) -> 8+8+8+8 = 32
10.0.0.0/16 is CIDR Range.
Here 16 mean that 16 bits are reserved for network, leaving us with rest 16.
	10.0.(0-255).(0-255)

AWS reserves the **first 4 IPs** and the **last 1 IP** of every subnet for itself (network, router, DNS, and broadcast)

In EC2 a NAT Gateway (Residing in Public Subnet) is used for 1 way communication from Private Subnet to internet.

## LLM Routing

Task complexity
Cost/Token
Provider Latency
Safety Requirements

