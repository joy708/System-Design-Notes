# System Design Deep Dive: HLD Interview Guide for Amazon SDE-2

## Table of Contents
1. Core Concepts with Examples
2. How Components Work Together
3. Real System Architectures
4. Interview Framework

---

## Part 1: Core Concepts with Step-by-Step Examples

### 1. Database Indexing

**What it is:** A data structure that improves query speed at the cost of additional writes and storage.

**Step-by-step example:**
Imagine a users table with 10 million records.

```
Without Index:
Query: SELECT * FROM users WHERE email = 'john@example.com'
- Database scans ALL 10M rows (O(n))
- Time: ~5 seconds

With B-Tree Index on email:
1. Database looks up 'john@example.com' in the index tree (O(log n))
2. Index points directly to row location on disk
3. Database reads that specific row
- Time: ~5 milliseconds
```

**Types and when to use:**
- **B-Tree Index:** Default, great for range queries (`WHERE age > 25`)
- **Hash Index:** Exact match queries only (`WHERE id = 123`), faster lookups
- **Bitmap Index:** Low cardinality columns (gender, status), data warehouses
- **Full-Text Index:** Text search (`WHERE description CONTAINS 'laptop'`)

**Trade-offs:**
- Writes become slower (must update index)
- Storage overhead (indexes can be 10-30% of table size)
- Too many indexes hurt performance

---

### 2. Partitioning (Horizontal vs Vertical)

**Horizontal Partitioning (Sharding):** Split rows across multiple databases
**Vertical Partitioning:** Split columns across multiple databases

**Step-by-step example - Horizontal:**

```
Original orders table (10M rows):
orders: [order_id, user_id, product_id, amount, created_at, ...]

Partition by range (created_at):
- DB1: orders from 2020-2021 (3M rows)
- DB2: orders from 2022-2023 (4M rows)  
- DB3: orders from 2024-2025 (3M rows)

Query: "Get orders from last month"
- Routes to DB3 only
- Scans 3M instead of 10M rows
```

**Step-by-step example - Vertical:**

```
User table split:
- Hot data DB: [user_id, name, email, last_login] - accessed frequently
- Cold data DB: [user_id, address, preferences, metadata] - rarely accessed

Query: "Show user profile page"
- Reads from Hot DB only (faster, smaller)
- Cold DB queried only when needed
```

**When to use:**
- Horizontal: Data growing too large, need parallel processing
- Vertical: Different access patterns (hot vs cold data)

---

### 3. Sharding

**What it is:** Distributing data across multiple databases based on a shard key.

**Step-by-step example - User sharding:**

```
100M users, 4 database shards
Shard key: user_id

Shard assignment:
- Shard 0: user_id % 4 == 0 (25M users)
- Shard 1: user_id % 4 == 1 (25M users)
- Shard 2: user_id % 4 == 2 (25M users)
- Shard 3: user_id % 4 == 3 (25M users)

Request flow:
1. User 12345 logs in
2. Application: 12345 % 4 = 1
3. Route to Shard 1
4. Query executes on 25M subset
```

**Sharding strategies:**

**1. Range-based:**
```
- Shard 1: user_id 1-25M
- Shard 2: user_id 25M-50M
- Shard 3: user_id 50M-75M
- Shard 4: user_id 75M-100M

Problem: Hotspots if new users always go to last shard
```

**2. Hash-based:**
```
hash(user_id) % num_shards

Pros: Even distribution
Cons: Adding shards requires resharding (expensive)
```

**3. Directory-based:**
```
Lookup table:
user_id_range → shard_id
1-10M → Shard1
10M-20M → Shard2

Pros: Flexible reassignment
Cons: Extra lookup overhead
```

**Sharding challenges:**
- **Cross-shard queries:** Need to query multiple shards and merge results
- **Cross-shard transactions:** Distributed transactions are complex (2PC)
- **Resharding:** Adding/removing shards requires data migration

---

### 4. Consistent Hashing

**Problem it solves:** When adding/removing cache servers, minimize data redistribution.

**Step-by-step example:**

```
Traditional hashing (BAD):
3 cache servers, 9 keys
key % 3 determines server

Keys: k1, k2, k3, k4, k5, k6, k7, k8, k9
k1 % 3 = 1 → Server1
k2 % 3 = 2 → Server2
k3 % 3 = 0 → Server0
...

Add 1 server (now 4 total):
k1 % 4 = 1 → Server1 (unchanged)
k2 % 4 = 2 → Server2 (unchanged)
k3 % 4 = 3 → Server3 (MOVED!)
k4 % 4 = 0 → Server0 (MOVED!)
...

Result: 75% of keys moved! Cache stampede!
```

**Consistent hashing (GOOD):**

```
1. Hash space: 0 to 2^32-1 (imagine a ring)
2. Hash servers onto ring:
   - hash("Server1") = 100
   - hash("Server2") = 200
   - hash("Server3") = 300

3. Hash keys onto ring:
   - hash("key1") = 50
   - hash("key2") = 150
   - hash("key3") = 250

4. Assignment rule: Move clockwise to find next server
   - key1 (50) → Server1 (100)
   - key2 (150) → Server2 (200)
   - key3 (250) → Server3 (300)

5. Add Server4 at position 225:
   Only keys between 200-225 move from Server3 to Server4
   ~25% of Server3's keys (not 75% of all keys!)
```

**Virtual nodes optimization:**

```
Problem: Servers may hash unevenly on ring
Solution: Each physical server gets 100-200 virtual nodes

Server1 virtual nodes: S1-1, S1-2, ..., S1-150
Server2 virtual nodes: S2-1, S2-2, ..., S2-150

Result: More evenly distributed load
```

---

### 5. Replication

**What it is:** Keeping copies of data on multiple machines for availability and performance.

**Master-Slave Replication:**

```
Step-by-step write flow:
1. Client sends: UPDATE users SET name='John' WHERE id=5
2. Request goes to Master DB
3. Master executes update locally
4. Master writes to replication log
5. Slave DBs read replication log
6. Slaves apply the same update
7. Master returns success to client

Read flow:
1. Client sends: SELECT * FROM users WHERE id=5
2. Load balancer routes to any Slave DB
3. Slave returns data

Benefits:
- Reads scale horizontally (add more slaves)
- Master focuses on writes
```

**Replication lag problem:**

```
Timeline:
t0: Master has balance=$100
t1: Client deposits $50
t2: Master updates balance=$150
t3: Replication to Slave (takes 200ms)
t4: Client immediately checks balance on Slave
    Slave still shows $100! (stale read)

Solutions:
1. Read-your-writes: Route user's reads to master for short time
2. Monotonic reads: Stick user to same slave (session affinity)
3. Synchronous replication: Wait for slaves (slower writes)
```

**Master-Master Replication:**

```
Two masters: both accept writes

Conflict example:
t0: Both masters have balance=$100
t1: Master1 receives: balance -= $50 (now $50)
t2: Master2 receives: balance += $30 (now $130)
t3: Replicate to each other
    Master1 sees: $50 then +$30 = $80
    Master2 sees: $130 then -$50 = $80
    (Last-write-wins resolves conflict)

Better: Use conflict-free data types (CRDTs)
```

**Multi-Leader Replication:**

Used in multi-datacenter setups.

```
DC1 (US): Leader1 + followers
DC2 (EU): Leader2 + followers
DC3 (ASIA): Leader3 + followers

Write in US → Leader1 → async replicate to Leader2, Leader3
Write in EU → Leader2 → async replicate to Leader1, Leader3

Benefits:
- Low latency writes (write to nearby DC)
- Tolerates datacenter failure

Challenges:
- Conflict resolution complexity
```

---

### 6. Quorum and Sloppy Quorum

**Quorum Basics:**

```
Setup: Data replicated across N nodes
W = write quorum (must write to W nodes)
R = read quorum (must read from R nodes)

Rule: W + R > N guarantees consistency

Example: N=3, W=2, R=2
Write:
1. Client writes key=value
2. System writes to 2 out of 3 nodes
3. Returns success

Read:
1. Client reads key
2. System reads from 2 out of 3 nodes
3. Returns latest version (by timestamp)
4. At least 1 node has latest write (overlap guaranteed)
```

**Scenarios:**

```
Strong consistency: W + R > N
- N=3, W=2, R=2
- Always read latest write

High write availability: W=1, R=N
- Writes succeed to any 1 node (fast!)
- Reads must check all nodes (slow)

High read availability: W=N, R=1
- Writes must succeed on all nodes (slow)
- Reads from any node (fast, but may be stale)

Eventual consistency: W=1, R=1
- Fast reads and writes
- May read stale data
```

**Sloppy Quorum:**

```
Problem: 3 replicas for key="user123": Node1, Node2, Node3
Node2 is down. Can't achieve W=2 quorum!

Sloppy Quorum solution:
1. Write to Node1 (primary)
2. Write to Node4 (temporary, not primary)
3. Return success
4. When Node2 recovers, Node4 hands off data to Node2

Use case: Favor availability over strict consistency
Example: Amazon Dynamo, Cassandra
```

**Hinted Handoff:**

```
Step-by-step:
1. Node2 goes down
2. Client writes "key1=value1"
3. System writes to Node1, Node4 (with hint: "belongs to Node2")
4. Node4 stores: {key: key1, value: value1, hint: Node2}
5. Node2 comes back online
6. Node4 sees Node2 is up
7. Node4 transfers key1=value1 to Node2
8. Node4 deletes local copy
```

---

### 7. Coordination Services (ZooKeeper, etcd)

**What they do:** Distributed configuration, leader election, distributed locks, service discovery.

**Leader Election Example:**

```
Scenario: 3 cache servers need to elect 1 leader to coordinate cache invalidation

Step-by-step with ZooKeeper:
1. All 3 servers connect to ZooKeeper
2. Each creates ephemeral sequential node in /election:
   - Server1 creates: /election/server-0000000001
   - Server2 creates: /election/server-0000000002
   - Server3 creates: /election/server-0000000003

3. Each server lists all nodes in /election
4. Server with lowest sequence number is leader
   - Server1 is leader!

5. Server2 and Server3 watch /election/server-0000000001
6. If Server1 crashes:
   - Ephemeral node /election/server-0000000001 deleted
   - Server2 and Server3 get notification
   - Server2 (now lowest) becomes new leader

Benefits:
- Automatic failover
- Prevents split-brain (two leaders)
```

**Distributed Lock Example:**

```
Scenario: Only 1 worker should process a job at a time

Step-by-step:
1. Worker1 tries to acquire lock for "job-123"
2. Worker1 creates: /locks/job-123 (ephemeral node)
3. Success! Worker1 has lock
4. Worker1 processes job
5. Worker2 tries to create /locks/job-123
6. Fails! Node already exists
7. Worker2 watches /locks/job-123
8. Worker1 finishes, deletes /locks/job-123 (or crashes, node auto-deleted)
9. Worker2 gets notification, acquires lock

Prevents:
- Duplicate job processing
- Race conditions
```

**Service Discovery:**

```
Microservices: user-service needs to find payment-service instances

1. Payment service instances register themselves:
   - payment-1: /services/payment/192.168.1.10:8080
   - payment-2: /services/payment/192.168.1.11:8080

2. User service queries /services/payment
3. Gets list: [192.168.1.10:8080, 192.168.1.11:8080]
4. User service calls one of them
5. If payment-2 crashes, ephemeral node deleted
6. User service gets notification, updates list
```

---

### 8. Push vs Pull Architecture

**Pull-based (Polling):**

```
Step-by-step: News feed

1. User opens app
2. App requests: GET /feed
3. Server queries:
   - Get user's followed accounts
   - Get recent posts from each
   - Rank and return top 50
4. Server returns feed
5. User scrolls, sees feed
6. Every 30 seconds, app polls again

Pros:
- Simple
- Works with any client

Cons:
- Wasted requests if no updates
- Delayed updates (up to polling interval)
- High server load at scale
```

**Push-based (WebSocket/SSE):**

```
Step-by-step: Real-time notifications

1. User opens app
2. App opens WebSocket to server
3. Server maintains connection
4. Friend posts new photo
5. Server immediately pushes: {type: 'new_post', data: {...}}
6. App shows notification instantly

Pros:
- Real-time updates
- No wasted requests
- Lower latency

Cons:
- Stateful connections (harder to scale)
- Firewall/proxy issues
- Reconnection logic needed
```

**Hybrid (Fan-out on Write):**

```
Step-by-step: Twitter/Instagram feed

Write (Push):
1. Celebrity with 10M followers posts tweet
2. System pushes tweet to Redis caches:
   - For each follower, append to their feed cache
   - feed:user_1 → [tweet_id_999, tweet_id_998, ...]
3. Background workers do this async

Read (Pull):
1. User opens app
2. App requests: GET /feed
3. Server reads from feed:user_1 cache (fast!)
4. Returns immediately (pre-computed)

Hybrid for celebrities:
- Don't fan-out to 10M users (too expensive)
- Compute their tweets on-read instead
- Mix cached feed + celebrity tweets at read time
```

---

### 9. CAP Theorem

**C (Consistency):** All nodes see the same data at the same time
**A (Availability):** Every request gets a response (success or failure)
**P (Partition Tolerance):** System works despite network splits

**You can only have 2 out of 3.**

**Step-by-step example - Network partition:**

```
Setup: 3 nodes in 2 datacenters
- DC1: Node1, Node2
- DC2: Node3
- Network split: DC1 and DC2 can't communicate

Write request to DC1: SET key=value1
Write request to DC2: SET key=value2

CP System (Consistency + Partition Tolerance):
- DC1 and DC2 both reject writes
- Return error: "Cannot guarantee consistency"
- Availability sacrificed
- Example: HBase, MongoDB (in certain modes)

AP System (Availability + Partition Tolerance):
- DC1 accepts: key=value1
- DC2 accepts: key=value2
- Both return success
- Consistency sacrificed (diverged data)
- When partition heals, conflict resolution needed
- Example: Cassandra, DynamoDB

CA System (Consistency + Availability):
- Only possible without partitions
- Not realistic in distributed systems
- Example: Traditional single-node RDBMS
```

---

### 10. Bloom Filters

**What it is:** Space-efficient probabilistic data structure to test set membership.

**Step-by-step example:**

```
Problem: Check if username exists (avoid DB query for non-existent users)

Setup:
- Bit array of size 10 (simplified)
- 2 hash functions: h1, h2

Add "alice":
1. h1("alice") = 2 → set bit[2] = 1
2. h2("alice") = 7 → set bit[7] = 1
Bit array: [0,0,1,0,0,0,0,1,0,0]

Add "bob":
1. h1("bob") = 2 → set bit[2] = 1 (already set)
2. h2("bob") = 5 → set bit[5] = 1
Bit array: [0,0,1,0,0,1,0,1,0,0]

Check "alice":
1. h1("alice") = 2 → bit[2] = 1 ✓
2. h2("alice") = 7 → bit[7] = 1 ✓
Result: Probably exists (check DB)

Check "charlie":
1. h1("charlie") = 3 → bit[3] = 0 ✗
Result: Definitely does NOT exist (skip DB!)

Check "dave":
1. h1("dave") = 2 → bit[2] = 1 ✓
2. h2("dave") = 5 → bit[5] = 1 ✓
Result: Probably exists (FALSE POSITIVE! not actually in set)

Properties:
- No false negatives (if it says "not exists", it's definitely not)
- Possible false positives (if it says "exists", might be wrong)
- Can't remove items
```

**Use cases:**
- Cassandra/HBase: Avoid disk reads for non-existent keys
- Web crawlers: Check if URL already crawled
- CDN: Check if content cached before upstream request

---

### 11. Load Balancing Algorithms

**Round Robin:**
```
Requests: R1, R2, R3, R4, R5, R6
Servers: S1, S2, S3

R1 → S1
R2 → S2
R3 → S3
R4 → S1
R5 → S2
R6 → S3

Pros: Simple, fair distribution
Cons: Ignores server load/capacity
```

**Weighted Round Robin:**
```
Servers: S1 (weight=3), S2 (weight=2), S3 (weight=1)
Pattern: S1, S1, S1, S2, S2, S3, repeat

Use: Different server capacities
```

**Least Connections:**
```
State:
S1: 5 active connections
S2: 3 active connections
S3: 7 active connections

New request → S2 (least loaded)

Pros: Adapts to actual load
Cons: Requires tracking state
```

**Consistent Hashing:**
```
Use client IP or session ID as key
hash(client_ip) → server

Pros: Sticky sessions (same client → same server)
Cons: May not balance evenly
```

---

### 12. Caching Strategies

**Cache-Aside (Lazy Loading):**
```
Read flow:
1. App checks cache: GET user:123
2. Cache miss
3. App queries DB: SELECT * FROM users WHERE id=123
4. App writes to cache: SET user:123 = {data}
5. App returns data to client

Write flow:
1. App updates DB
2. App invalidates cache: DELETE user:123
3. Next read will reload fresh data

Pros: Only cache what's needed
Cons: Cache miss penalty, possible stale data
```

**Write-Through:**
```
Write flow:
1. App writes to cache: SET user:123 = {data}
2. Cache synchronously writes to DB
3. Cache returns success
4. App returns success to client

Pros: Cache always consistent with DB
Cons: Write latency, writes even for rarely-read data
```

**Write-Behind (Write-Back):**
```
Write flow:
1. App writes to cache: SET user:123 = {data}
2. Cache returns success immediately
3. Cache asynchronously writes to DB (batched)

Pros: Fast writes
Cons: Risk of data loss if cache fails before DB write
```

**Refresh-Ahead:**
```
Background process:
1. Identify frequently accessed keys
2. Before expiration, reload from DB
3. Keep hot data always cached

Example: Product page views
- Product:123 gets 1000 req/sec
- TTL = 5 min
- At 4:30, background job reloads from DB
- No cache miss penalty for users
```

---

### 13. Rate Limiting Algorithms

**Token Bucket:**
```
Parameters:
- Capacity: 10 tokens
- Refill rate: 1 token/second

Timeline:
t=0: Bucket has 10 tokens
t=1: Request arrives, consume 1 token (9 left)
t=2: Request arrives, consume 1 token (8 left)
     Refill: +1 token (9 left)
...
t=10: Bucket full (10 tokens)
t=11: 15 requests arrive simultaneously
      - Process 10 (consume all tokens)
      - Reject 5 (no tokens left)

Pros: Allows bursts up to capacity
Cons: Implementation complexity
```

**Leaky Bucket:**
```
Parameters:
- Queue size: 10 requests
- Process rate: 1 request/second

Timeline:
t=0: Queue empty
t=1: 5 requests arrive → added to queue (5 in queue)
t=2: Process 1 request, queue = 4
t=3: Process 1 request, queue = 3
t=4: 10 requests arrive → queue fills (10 in queue), reject extras
t=5: Process 1 request, queue = 9

Pros: Smooth output rate
Cons: Can't handle legitimate bursts
```

**Fixed Window Counter:**
```
Window: 1 minute
Limit: 100 requests/minute

00:00 - 00:59: Request count = 0
00:30: Request arrives, count = 1
00:45: Request arrives, count = 2
...
00:58: 99th request, count = 99
01:00: Window resets, count = 0

Problem (boundary issue):
00:30 - 00:59: 100 requests (allowed)
01:00 - 01:29: 100 requests (allowed)
In 1-minute span (00:30-01:29): 200 requests! (2x limit)
```

**Sliding Window Log:**
```
Limit: 100 requests/minute

Timestamp log:
[12:00:10, 12:00:15, 12:00:20, ..., 12:01:05]

New request at 12:01:10:
1. Remove all timestamps before 12:00:10 (1 min ago)
2. Count remaining timestamps
3. If count < 100, allow request and add timestamp
4. Else, reject

Pros: Accurate, no boundary issue
Cons: Memory overhead (store all timestamps)
```

**Sliding Window Counter (Hybrid):**
```
Combines fixed window + sliding

Example at 12:00:30 (30% into current window):
- Previous window (11:00-12:00): 80 requests
- Current window (12:00-13:00): 20 requests so far
- Estimated count: 80 * (1 - 0.3) + 20 = 56 + 20 = 76

If limit is 100: Allow request (76 < 100)

Pros: Memory efficient, smooth
Cons: Approximate
```

---

## Part 2: How Components Work Together

### Combining Multiple Concepts

**Example: Distributed Cache System (Redis Cluster)**

```
Components used:
- Sharding (data distribution)
- Consistent hashing (minimize redistribution)
- Replication (availability)
- Quorum (consistency)

Architecture:
┌─────────────────┐
│  Load Balancer  │
└────────┬────────┘
         │
    ┌────┴────┬────────┬────────┐
    │         │        │        │
┌───▼────┐ ┌─▼─────┐ ┌▼──────┐ ┌▼──────┐
│Shard 1 │ │Shard 2│ │Shard 3│ │Shard 4│
│Master  │ │Master │ │Master │ │Master │
└───┬────┘ └───┬───┘ └───┬───┘ └───┬───┘
    │          │         │         │
┌───▼────┐ ┌───▼───┐ ┌───▼───┐ ┌───▼───┐
│Replica │ │Replica│ │Replica│ │Replica│
└────────┘ └───────┘ └───────┘ └───────┘

Write flow for key="user:12345":
1. Client: SET user:12345 = {data}
2. Consistent hash: hash("user:12345") → Shard 2
3. Write to Shard 2 Master
4. Async replicate to Shard 2 Replica
5. Return success (W=1, eventual consistency)

Read flow:
1. Client: GET user:12345
2. Consistent hash: hash("user:12345") → Shard 2
3. Read from Shard 2 (master or replica)
4. Return data

Node failure:
- Shard 2 Master crashes
- Replica promoted to Master (via ZooKeeper/Sentinel)
- Requests continue to Shard 2 (now at new master)

Adding Shard 5:
- Consistent hashing redistributes ~20% of keys
- Other 80% stay on original shards
- Minimal disruption
```

---

**Example: Distributed Database (Cassandra)**

```
Components:
- Partitioning (hash-based)
- Replication (multiple datacenters)
- Quorum (tunable consistency)
- Gossip protocol (cluster membership)
- Bloom filters (optimize reads)

Ring topology:
Nodes: N1, N2, N3, N4, N5, N6
Token ranges:
- N1: 0-100
- N2: 101-200
- N3: 201-300
- N4: 301-400
- N5: 401-500
- N6: 501-600

Write with replication factor=3:
1. Client writes: INSERT INTO users (id, name) VALUES (123, 'Alice')
2. Partition key: hash(123) = 250
3. Coordinator: N3 (owns token 250)
4. N3 writes locally
5. N3 replicates to N4, N5 (next 2 nodes clockwise)
6. W=2 quorum: Wait for 2 acks (N3 + N4)
7. Return success to client
8. N5 writes async (hinted handoff if down)

Read with R=2:
1. Client: SELECT * FROM users WHERE id=123
2. hash(123) = 250 → Coordinator N3
3. N3 sends read to N3, N4, N5 (all replicas)
4. Waits for 2 responses (R=2)
5. Compares timestamps, returns latest
6. Background repair: Updates stale replica

Consistency levels:
- ONE (W=1, R=1): Low latency, eventual consistency
- QUORUM (W=2, R=2 for RF=3): Balanced
- ALL (W=3, R=3): Strong consistency, low availability
- LOCAL_QUORUM: Quorum in local datacenter (multi-DC)
```

---

## Part 3: Real System Architectures

### Case Study 1: URL Shortener (like bit.ly)

**Requirements:**
- Generate short URLs
- Redirect to original URLs
- Track analytics
- 100M URLs created/month
- 1B redirects/month

**Architecture:**

```
┌──────────┐     ┌─────────────┐
│  Client  │────▶│ Load Balancer│
└──────────┘     └──────┬──────┘
                        │
              ┌─────────┴─────────┐
              │                   │
         ┌────▼────┐         ┌────▼────┐
         │  API    │         │  API    │
         │ Servers │         │ Servers │
         └────┬────┘         └────┬────┘
              │                   │
              └─────────┬─────────┘
                        │
              ┌─────────▼─────────┐
              │                   │
         ┌────▼────┐         ┌────▼────┐
         │  Redis  │         │ MySQL   │
         │  Cache  │         │  (main) │
         └─────────┘         └────┬────┘
                                  │
                        ┌─────────┴──────────┐
                   ┌────▼────┐         ┌─────▼────┐
                   │  MySQL  │         │  MySQL   │
                   │  (read) │         │  (read)  │
                   └─────────┘         └──────────┘
```

**Database schema (MySQL):**
```sql
CREATE TABLE urls (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  short_code VARCHAR(7) UNIQUE,
  long_url TEXT,
  created_at TIMESTAMP,
  user_id BIGINT,
  INDEX idx_short_code (short_code)  -- B-tree index
);

-- Sharding key: short_code
-- Partition by hash(short_code) % num_shards
```

**Short code generation:**
```
Approach 1: Base62 encoding of auto-increment ID
- ID 12345 → base62 → "dnh"
- ID space: 62^7 = 3.5 trillion URLs

Approach 2: MD5 hash + collision handling
- hash(long_url + timestamp) → take first 7 chars
- If collision, retry with different salt

Chosen: Approach 1 (simpler, no collisions)
```

**Write flow (create short URL):**
```
1. Client: POST /shorten {long_url}
2. API generates short_code from auto-increment ID
3. Write to MySQL master
4. Async replicate to read replicas
5. Return short_code to client
(No cache on write - lazy loading on read)
```

**Read flow (redirect):**
```
1. Client: GET /dnh
2. API checks Redis cache: GET url:dnh
3. Cache hit? Return long_url and redirect
4. Cache miss:
   - Query MySQL read replica: SELECT long_url WHERE short_code='dnh'
   - Write to Redis: SET url:dnh = {long_url} EX 3600
   - Return long_url and redirect
5. Async: Increment analytics counter in separate table
```

**Why this design:**
- **Redis cache:** 90%+ hit rate, reduce DB load
- **Read replicas:** Read-heavy (1000:1 read:write ratio)
- **Sharding:** When URLs > 1B, shard by hash(short_code)
- **Index on short_code:** O(log n) lookup vs O(n) scan
- **Auto-increment ID:** Simple, distributed via DB
- **TTL in cache:** Avoid stale data, manage memory

**Scaling considerations:**
- Cache stampede: Use Redis cluster (consistent hashing)
- Write bottleneck: Shard MySQL, use ID ranges per shard
- Analytics: Move to Kafka + data warehouse for heavy analysis

---

### Case Study 2: Notification Service

**Requirements:**
- Send email, SMS, push notifications
- Millions of notifications/day
- Delivery guarantees
- Rate limiting per user
- Priority handling

**Architecture:**

```
┌─────────┐     ┌─────────────┐      ┌──────────────┐
│Services │────▶│  API Gateway│─────▶│ Message Queue│
│(trigger)│     └─────────────┘      │   (Kafka)    │
└─────────┘                          └──────┬───────┘
                                            │
                                    ┌───────┴────────┐
                                    │                │
                              ┌─────▼─────┐   ┌──────▼──────┐
                              │  Consumer │   │  Consumer   │
                              │  Group 1  │   │  Group 2    │
                              └─────┬─────┘   └──────┬──────┘
                                    │                │
                              ┌─────▼─────┐   ┌──────▼──────┐
                              │   Redis   │   │  PostgreSQL │
                              │  (state)  │   │   (logs)    │
                              └───────────┘   └─────────────┘
                                    │
                              ┌─────▼─────┐
                              │  External │
                              │ Providers │
                              │(SNS,Twilio)│
                              └───────────┘
```

**Flow:**

```
1. Trigger event:
   - User signs up
   - Order placed
   - Friend request

2. Service publishes to Kafka:
   {
     type: 'email',
     user_id: 12345,
     template: 'welcome',
     priority: 'high',
     data: {...}
   }

3. Consumer reads from Kafka:
   - Check Redis: rate_limit:user:12345:email
   - If under limit:
     * Render template
     * Call external provider (SendGrid)
     * Log to PostgreSQL
     * Update Redis counter
   - If over limit:
     * Queue for later (separate topic)

4. Retry logic:
   - Provider fails? → Retry topic (exponential backoff)
   - Max retries: 3
   - Dead letter queue for manual review
```

**Components explained:**

**Kafka partitioning:**
```
Partitions: 10
Partition key: user_id

Benefits:
- User 12345 always goes to same partition
- Messages for same user processed in order
- Parallel processing across partitions
```

**Rate limiting (Token Bucket in Redis):**
```
Key: rate_limit:user:12345:email
Value: {tokens: 8, last_refill: timestamp}

Algorithm:
1. Get current state from Redis
2. Calculate tokens to add: (now - last_refill) * refill_rate
3. new_tokens = min(max_tokens, current_tokens + added)
4. If new_tokens >= 1:
   - Decrement 1 token
   - Send notification
   - Update Redis
5. Else:
   - Reject / queue for later
```

**Priority handling:**
```
Kafka topics:
- notifications-high (consumer group: 5 instances)
- notifications-medium (consumer group: 3 instances)
- notifications-low (consumer group: 1 instance)

High priority gets more consumer resources
```

**Why this design:**
- **Kafka:** Durability, replay, parallel processing
- **Redis:** Fast rate limit checks, ephemeral state
- **PostgreSQL:** Audit trail, analytics
- **Consumer groups:** Scale horizontally
- **Idempotency:** Track message ID in Redis to avoid duplicates

---

### Case Study 3: Distributed Task Scheduler

**Requirements:**
- Schedule jobs (one-time, recurring)
- Execute at specific times
- Retry failed jobs
- Handle millions of tasks
- No duplicate execution

**Architecture:**

```
┌─────────────┐     ┌───────────────┐
│   Clients   │────▶│  API Servers  │
└─────────────┘     └───────┬───────┘
                            │
                    ┌───────▼────────┐
                    │   PostgreSQL   │
                    │  (tasks table) │
                    └───────┬────────┘
                            │
                    ┌───────▼────────┐
                    │   ZooKeeper    │
                    │ (coordination) │
                    └───────┬────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         ┌────▼────┐   ┌────▼────┐   ┌────▼────┐
         │ Worker 1│   │ Worker 2│   │ Worker 3│
         │(partition│   │(partition│  │(partition│
         │  0-333) │   │ 334-666)│   │ 667-999)│
         └─────────┘   └─────────┘   └─────────┘
```

**Database schema:**
```sql
CREATE TABLE tasks (
  id BIGINT PRIMARY KEY,
  partition_key INT, -- hash(id) % 1000
  scheduled_time TIMESTAMP,
  status VARCHAR(20), -- pending, running, completed, failed
  retry_count INT DEFAULT 0,
  payload JSONB,
  lock_id VARCHAR(36), -- UUID of worker holding lock
  lock_expiry TIMESTAMP,
  INDEX idx_partition_time (partition_key, scheduled_time, status)
);
```

**Worker assignment (via ZooKeeper):**
```
Startup:
1. Worker1 connects to ZooKeeper
2. Creates ephemeral node: /workers/worker1
3. All workers list /workers, count total (e.g., 3 workers)
4. Self-assign partition range:
   - Worker1: 0-333 (first 1/3)
   - Worker2: 334-666 (second 1/3)
   - Worker3: 667-999 (third 1/3)

5. Each worker watches /workers for changes
6. If Worker2 crashes:
   - Ephemeral node deleted
   - Workers 1 & 3 rebalance:
     * Worker1: 0-499
     * Worker3: 500-999
```

**Task execution flow:**
```
Worker1 polling loop (every 5 seconds):

1. Query DB:
   SELECT * FROM tasks
   WHERE partition_key BETWEEN 0 AND 333
   AND scheduled_time <= NOW()
   AND status = 'pending'
   AND (lock_expiry IS NULL OR lock_expiry < NOW())
   LIMIT 100
   FOR UPDATE SKIP LOCKED; -- Prevent lock contention

2. For each task:
   - Acquire lock:
     UPDATE tasks
     SET status='running', lock_id='worker1-uuid', lock_expiry=NOW()+5min
     WHERE id=? AND status='pending'
   
   - If update affects 1 row (acquired lock):
     * Execute task payload
     * On success: UPDATE status='completed'
     * On failure: UPDATE status='failed', retry_count=retry_count+1
   
   - If update affects 0 rows:
     * Another worker got it, skip

3. Heartbeat:
   - Every 1 min, extend lock_expiry for running tasks
   - Prevents lock timeout if task takes long

4. Dead lock recovery:
   - Worker2 crashes while holding locks
   - lock_expiry passes
   - Other workers can now acquire those tasks
```

**Handling recurring tasks:**
```
Cron-like schedule: "0 0 * * *" (daily at midnight)

When completing recurring task:
1. Calculate next_scheduled_time
2. INSERT new task with next_scheduled_time
3. Mark current task as completed

Optimization: Use separate table for recurring configs
```

**Why this design:**
- **Partitioning:** Distribute load across workers, avoid scanning all tasks
- **ZooKeeper:** Dynamic worker assignment, automatic rebalancing
- **Pessimistic locking:** Prevent duplicate execution
- **Lock expiry:** Handle worker crashes without manual intervention
- **FOR UPDATE SKIP LOCKED:** High concurrency, no lock waits
- **Index on (partition, time, status):** Efficient polling query

---

## Part 4: Interview Framework (ANSWERED Framework)

When you get a system design question in an interview, follow this structure:

### A - Ask Clarifying Questions (3-5 minutes)

```
Functional requirements:
- What exactly does the system do?
- What are the core features?
- What can we deprioritize?

Non-functional requirements:
- Expected scale (users, requests/sec, data size)?
- Read vs write ratio?
- Latency requirements?
- Consistency vs availability?
- Geographic distribution?

Constraints:
- Technology preferences?
- Budget?
- Team size/expertise?
```

**Example for "Design Twitter":**
```
Questions:
- Features: Tweet, timeline, follow, like, retweet?
- Scale: 500M users, 100M daily active?
- Read:write ratio: 100:1?
- Timeline: Real-time or eventual consistency?
- Media: Support images/videos?
```

### N - Numbers (Back-of-Envelope Estimates) (5 minutes)

```
Calculate:
- Storage: QPS × data size × time
- Bandwidth: QPS × request/response size
- Memory (cache): Hot data size
- Servers: QPS / (single server capacity)

Example - Twitter:
- 100M DAU, each posts 2 tweets/day
- Writes: 100M × 2 / 86400 = 2,300 tweets/sec
- Peak (3x): ~7,000 tweets/sec
- Reads (100:1): 230,000 reads/sec
- Storage: 2,300 tweets/sec × 280 chars × 2 bytes × 86400 × 365
          = ~40 TB/year (text only)
- Cache: 20% of users generate 80% of traffic
         = 20M users × 500 tweets × 1KB = 10 TB cache
```

### S - System Interface (API Design) (3 minutes)

```
Define clean APIs:

POST /v1/tweets
{
  text: "Hello world",
  user_id: 12345,
  media_ids: [...]
}
Response: {tweet_id, timestamp, ...}

GET /v1/timeline/{user_id}?limit=50&offset=0
Response: {tweets: [...], next_offset: 50}

POST /v1/follow
{user_id: 123, followee_id: 456}
```

### W - Write High-Level Design (15 minutes)

```
1. Draw major components:
   - Clients (web, mobile)
   - Load balancers
   - Application servers
   - Databases
   - Caches
   - Message queues
   - Storage services

2. Show data flow:
   - Request path
   - Write path
   - Read path

3. Identify bottlenecks early
```

### E - Explain Key Components (10 minutes)

```
For each major component, explain:
- Why this choice?
- Alternatives considered?
- Trade-offs?

Example:
Database choice:
- Option 1: PostgreSQL
  + ACID, relations, complex queries
  - Harder to scale writes
  
- Option 2: Cassandra
  + Scales writes, multi-DC
  - Eventual consistency, no joins
  
- Chosen: PostgreSQL for users/metadata (rarely updated)
           Cassandra for tweets (high write volume)
```

### R - Review Design (Deep Dives) (15 minutes)

```
Interviewer will probe specific areas:

Common deep dives:
1. How do you handle hotspots? (celebrities)
2. How do you ensure consistency?
3. How do you handle failures?
4. How do you monitor and debug?
5. How do you scale 10x?

For each:
- State the problem clearly
- Propose solution with details
- Discuss trade-offs
```

**Example - Handle celebrity tweets:**
```
Problem: Celebrity with 50M followers posts
- Naive fan-out: Write to 50M user timelines (slow!)

Solution 1: Mixed fan-out
- Fan-out for regular users (< 1M followers)
- On-demand fetch for celebrities
- Merge at read time

Solution 2: Cache celebrity tweets separately
- Special "celebrity timeline" cache
- Merge with regular timeline at read

Chosen: Solution 1 (Twitter's approach)
- Balances write and read cost
```

### E - Extend Design (Bonus Topics) (5 minutes)

```
If time permits, discuss:
- Security (authentication, rate limiting)
- Monitoring (metrics, alerts, dashboards)
- Analytics (A/B testing, feature flags)
- Deployment (CI/CD, blue-green)
- Cost optimization
```

### D - Discuss Trade-offs (Throughout)

```
Always mention trade-offs:
- "We chose X over Y because..."
- "This approach sacrifices A for B"
- "In future, we might need to..."

Example:
"We're using eventual consistency for the timeline
 to achieve low latency and high availability.
 Trade-off: Users might briefly see stale data.
 Acceptable because Twitter is not a banking app."
```

---

## Quick Reference: Technology Choices

### Databases

**SQL (PostgreSQL, MySQL):**
```
Use when:
- ACID transactions needed
- Complex queries (joins, aggregations)
- Data has clear schema
- Moderate scale (< 10M rows per table with good indexing)

Examples:
- User accounts
- Financial transactions
- Inventory management
```

**NoSQL - Key-Value (Redis, DynamoDB):**
```
Use when:
- Simple key-based access
- Need millisecond latency
- Ephemeral data (sessions, cache)

Examples:
- Session storage
- Real-time leaderboards
- Rate limiting counters
```

**NoSQL - Document (MongoDB):**
```
Use when:
- Flexible schema
- Nested documents
- Rapid iteration

Examples:
- Product catalogs
- User profiles
- CMS
```

**NoSQL - Wide Column (Cassandra, HBase):**
```
Use when:
- Write-heavy workload
- Time-series data
- Massive scale (billions of rows)
- Multi-datacenter replication

Examples:
- IoT sensor data
- Messaging history
- Audit logs
```

**NoSQL - Graph (Neo4j):**
```
Use when:
- Relationship-heavy data
- Graph traversals
- Recommendations

Examples:
- Social networks
- Fraud detection
- Knowledge graphs
```

### Message Queues

**Kafka:**
```
Use when:
- High throughput (millions/sec)
- Log aggregation
- Event streaming
- Need replay capability

Examples:
- Activity tracking
- Log aggregation
- CDC (Change Data Capture)
```

**RabbitMQ:**
```
Use when:
- Task distribution
- Request-reply patterns
- Complex routing

Examples:
- Background jobs
- Microservice communication
```

**SQS:**
```
Use when:
- Managed service preferred
- Standard queue needs
- AWS ecosystem

Examples:
- Decoupling services
- Buffering requests
```

### Caching

**Redis:**
```
Use when:
- Need data structures (lists, sets, sorted sets)
- Pub/sub messaging
- Distributed locks
- Sub-millisecond latency

Patterns:
- Session storage
- Real-time analytics
- Leaderboards
```

**Memcached:**
```
Use when:
- Simple key-value cache
- Multi-threaded performance
- Large cache sizes

Simpler than Redis, but less features
```

**CDN (CloudFront, Akamai):**
```
Use when:
- Static content delivery
- Global distribution
- Reduce origin load

Examples:
- Images, videos, JS/CSS
- API responses (with cache headers)
```

---

## Common Patterns Cheat Sheet

### Pattern 1: Write-Heavy System
```
Problem: High write throughput (millions/sec)

Solution:
1. Use Kafka for ingestion (buffer writes)
2. Batch writes to database
3. Use Cassandra or HBase (optimized for writes)
4. Async processing

Example: Analytics system, IoT platform
```

### Pattern 2: Read-Heavy System
```
Problem: Read:write ratio 1000:1

Solution:
1. Multi-layer caching (browser, CDN, Redis, app-level)
2. Read replicas for database
3. Denormalize data (trade storage for speed)
4. Precompute expensive queries

Example: News site, e-commerce product pages
```

### Pattern 3: Low Latency Requirement
```
Problem: p99 latency < 10ms

Solution:
1. Cache hot data in memory (Redis)
2. Use SSD for cold data
3. Geographically distributed (reduce network latency)
4. Precompute and cache results
5. Async processing for non-critical path

Example: Real-time bidding, gaming
```

### Pattern 4: Strong Consistency Requirement
```
Problem: Must see own writes, no stale reads

Solution:
1. Use ACID database (PostgreSQL)
2. Synchronous replication (W=all, R=one)
3. Read from master for critical reads
4. Distributed transactions (if cross-service)

Example: Banking, booking systems
```

### Pattern 5: High Availability Requirement
```
Problem: 99.99% uptime (< 1 hour downtime/year)

Solution:
1. Multi-region deployment
2. Async replication
3. Eventual consistency
4. Circuit breakers
5. Graceful degradation

Example: Social media, messaging
```

---

## Final Tips for Amazon Interviews

### 1. Leadership Principles Alignment

When discussing design choices, relate to Amazon's principles:

**Customer Obsession:**
```
"We prioritize read latency because customers expect instant search results."
```

**Ownership:**
```
"I'd set up monitoring and alerts to ensure we catch issues before customers do."
```

**Invent and Simplify:**
```
"Instead of a complex consensus algorithm, we can use a simpler master-slave setup given our consistency requirements."
```

**Frugality:**
```
"We can use reserved instances for baseline load and spot instances for peak handling to reduce costs by 60%."
```

### 2. Think Out Loud

Don't go silent. Say things like:
- "I'm thinking about the trade-off between..."
- "One approach would be X, but that has the downside of Y..."
- "Let me validate this number..."

### 3. Draw Clearly

- Use boxes for services
- Use cylinders for databases
- Use arrows with labels
- Number the steps in a flow
- Keep it organized

### 4. Handle Ambiguity

If a requirement is unclear:
- Make an assumption
- State it clearly
- Ask if it's reasonable

"I'm assuming we need single-digit millisecond latency. Is that correct?"

### 5. Know When to Dive Deep

Interviewer says: "How would you implement X?"
This is a signal to go deep. Provide:
- Algorithm/data structure
- Code-level details
- Time/space complexity

### 6. Discuss Monitoring

Always mention:
- Key metrics (latency, error rate, throughput)
- Alerts (thresholds, escalation)
- Dashboards (what to visualize)
- Logging (what to log, retention)

### 7. Consider Failure Scenarios

For each component, ask:
- What if this fails?
- How do we detect it?
- How do we recover?
- What's the user impact?

---

## Practice Problems

Work through these end-to-end:

1. **URL Shortener** (covered above)
2. **Design Twitter**
3. **Design YouTube**
4. **Design Uber**
5. **Design WhatsApp**
6. **Design Amazon.com Search**
7. **Design Rate Limiter**
8. **Design Distributed Cache**
9. **Design Web Crawler**
10. **Design Notification Service** (covered above)

For each:
- Apply ANSWERED framework
- Time yourself (45 minutes total)
- Draw the architecture
- Write down numbers
- Practice explaining out loud

Good luck with your Amazon interviews!
