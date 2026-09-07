# Load Balancing Algorithms

## Concept
Algorithms determine which backend receives traffic.

## Round Robin
Requests are distributed sequentially.
```text
Request 1 → App 1
Request 2 → App 2
Request 3 → App 3
Request 4 → App 1
```

## Least Connections
Traffic goes toward the backend with fewer active connections. Useful when request durations vary.

## Weighted
Servers receive traffic according to assigned weights.
```text
App 1 → Weight 3
App 2 → Weight 1
```
Useful when servers have different capacities.

## IP Hash
Uses client IP to consistently select a backend. Can provide basic session affinity.

## Selection Considerations
Consider:
- Request duration
- Backend capacity
- Session requirements
- Traffic distribution
- Application behavior

## SRE Relevance
Uneven distribution can overload one backend while others are underutilized, causing latency and errors.

## Quick Revision
```text
Round Robin       → sequential
Least Connections → fewer active connections
Weighted          → more traffic to stronger servers
IP Hash           → consistent backend based on client IP
```

## Real-World Example
One backend is at 90% CPU while others are around 40%. Investigate LB traffic distribution before simply adding more application capacity.
