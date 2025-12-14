# Chapter 6: Partitioning - Detailed Explanation

## What Is Partitioning?

**Partitioning** (also called **sharding**) means breaking a large dataset into smaller subsets called **partitions**.

**Each partition** is a small database of its own.

### Why Partition Data?

**Scalability**: The main reason for partitioning.

- A single machine has limits (disk, memory, CPU)
- Large datasets or high query throughput require multiple machines
- Partitioning spreads data and query load across multiple machines

**Goal**: Distribute data and queries evenly across nodes
- Avoid **hot spots** (nodes with disproportionately high load)
- Each node handles fair share of data and requests

---

## Terminology

Different databases use different terms:

| Term | Used by |
|------|---------|
| **Shard** | MongoDB, Elasticsearch |
| **Region** | HBase |
| **Tablet** | Bigtable |
| **vnode** | Cassandra, Riak |
| **vBucket** | Couchbase |

All mean the same thing: **partition**.

---

## Partitioning and Replication

**Partitioning and replication** are independent concepts that work together.

**Typical setup**:
- Data is partitioned across multiple nodes
- Each partition is replicated to multiple nodes for fault tolerance

**Example** (3 nodes, 2 replicas per partition):
```
Node 1: Partition A (leader), Partition B (follower), Partition C (follower)
Node 2: Partition A (follower), Partition B (leader), Partition C (follower)
Node 3: Partition A (follower), Partition B (follower), Partition C (leader)
```

Each node may be the leader for some partitions and a follower for others.

**Chapter 5 (replication) and Chapter 6 (partitioning) are complementary.**

---

## Partitioning of Key-Value Data

**Core question**: How do you decide which records go to which partition?

**Goal**: Distribute data evenly

**Two main approaches**:
1. Partitioning by key range
2. Partitioning by hash of key

---

## Partitioning by Key Range

**Idea**: Assign continuous range of keys to each partition

Similar to how a paper encyclopedia is divided into volumes (A-B, C-D, etc.)

### Example

**Data**: User records with user IDs

**Partitioning**:
```
Partition 1: user_id from 1 to 1,000,000
Partition 2: user_id from 1,000,001 to 2,000,000
Partition 3: user_id from 2,000,001 to 3,000,000
...
```

**Used by**: Bigtable, HBase, RethinkDB, MongoDB (before 2.4)

---

### Advantages of Key Range Partitioning

**1. Range queries are efficient**

**Example**: Fetch all records where timestamp is between 2024-01-01 and 2024-01-31

```
Partition 1: Jan 1-10    ← Query this
Partition 2: Jan 11-20   ← Query this
Partition 3: Jan 21-31   ← Query this
Partition 4: Feb 1-10    ← Skip this
```

You only need to query partitions that contain the relevant range.

**2. Keys are ordered within and across partitions**

Can efficiently scan keys in sorted order.

---

### Disadvantages of Key Range Partitioning

**Problem: Risk of hot spots**

If keys are not uniformly distributed, some partitions get more data/queries.

**Example 1**: Partitioning sensor data by timestamp
```
Partition for today: Gets ALL writes (hot spot!)
Partitions for past dates: Get no writes (idle)
```

**Solution**: Use compound key
- Primary key: `(sensor_name, timestamp)`
- First partition by sensor name, then by timestamp
- Spreads writes across multiple partitions

**Example 2**: Partitioning users by last name
```
Partition A-B: Many users (Smith, Brown, etc.)
Partition Q-R: Few users
```

Some partitions much larger than others.

---

### Determining Partition Boundaries

**Question**: How to choose the split points?

**Option 1**: Manually by administrator
- Tedious and error-prone

**Option 2**: Automatically by database
- Monitor data distribution
- Adjust boundaries to keep partitions roughly equal size

---

## Partitioning by Hash of Key

**Idea**: Use hash function to determine partition

**Hash function**:
- Takes a key (string, number, etc.)
- Returns a number
- Distributes keys uniformly

**Example**:
```
hash("user123") = 47823947
hash("user124") = 2938475
hash("user125") = 99237482

Partition = hash(key) mod N
```

**Used by**: Cassandra, MongoDB, Riak, Elasticsearch, Couchbase, Voldemort

---

### Hash Function Requirements

**NOT cryptographic hash** (like SHA-256)
- Don't need cryptographic strength
- Just need good distribution

**Examples**:
- Cassandra, MongoDB: MD5
- Voldemort: Fowler-Noll-Vo function

**Programming language hash functions** (Java's `hashCode()`, Python's `hash()`) are NOT suitable!
- Same key may have different hash in different processes
- Need consistent hash across all nodes

---

### Example of Hash Partitioning

**Setup**:
- 4 partitions (0, 1, 2, 3)
- Hash function returns 32-bit integer
- Partition = hash(key) mod 4

**Data distribution**:
```
Partition 0: Records where hash(key) mod 4 = 0
Partition 1: Records where hash(key) mod 4 = 1
Partition 2: Records where hash(key) mod 4 = 2
Partition 3: Records where hash(key) mod 4 = 3
```

**Result**: Keys distributed fairly uniformly

---

### Advantages of Hash Partitioning

**1. Avoids hot spots**

Even if keys are skewed (many users named "Smith"), hashing distributes evenly.

**2. Simple and effective**

Works well for most use cases.

---

### Disadvantages of Hash Partitioning

**Problem: Lose ability to do efficient range queries**

**Example**: Find all records with keys between "user100" and "user200"

```
hash("user100") = 47823947 → Partition 3
hash("user150") = 2938475  → Partition 1
hash("user200") = 99237482 → Partition 2

Must query ALL partitions!
```

Adjacent keys are scattered across different partitions.

**MongoDB**: If using hash-based sharding, any range query goes to all partitions.

**Riak, Couchbase, Voldemort**: Don't support range queries on primary key at all.

---

### Consistent Hashing

**Concept** (slightly different from partitioning):

**Traditional modulo hashing**:
```
Partition = hash(key) mod N
```

**Problem**: If N changes (add/remove nodes), almost all keys need to move!

**Consistent hashing**:
- Hash function output is a range (e.g., 0 to 2^32-1)
- This range forms a circle (hash ring)
- Each node is assigned a random position on the ring
- Each key goes to the next node clockwise

**Visualization**:
```
        0
    A       B
  
D             C
       2^31
```

**Advantage**: When node added/removed, only nearby keys need to move.

**Used by**: Cassandra, Riak

**Note**: "Consistent hashing" terminology is confusing
- Not same as "consistency" in CAP theorem
- Better term: "hash partitioning with dynamic partition assignment"

---

## Skewed Workloads and Relieving Hot Spots

**Problem**: Even with hash partitioning, can still get hot spots.

**Example**: Social media
```
Celebrity user posts → millions of reads/writes to same key
Hash always sends that key to same partition → hot spot!
```

**Current solutions** (as of book publication):
- Most systems can't automatically detect and compensate
- Application must handle it

**Workarounds**:

1. **Add random number to key**
   ```
   key = "celebrity_user_id_" + random(0, 99)
   
   Spreads writes across 100 keys
   But now reads must read all 100 keys and combine results!
   ```

2. **Split hot keys** only (don't add overhead for all keys)
   - Track which keys are hot
   - Apply splitting only to those
   - Add metadata to indicate which keys are split

**Trade-off**: Additional complexity in application code.

---

## Partitioning and Secondary Indexes

**So far**: Partitioning key-value data (simple)

**Real applications**: Often need to search by multiple criteria (secondary indexes)

**Problem**: Secondary indexes don't map neatly to partitions.

**Two main approaches**:
1. Document-partitioned indexes (local indexes)
2. Term-partitioned indexes (global indexes)

---

## Document-Partitioned Indexes (Local Indexes)

**Idea**: Each partition maintains its own secondary indexes, covering only documents in that partition.

**Example**: E-commerce site selling cars

**Partitioning**: By document ID (car ID)
```
Partition 1: car_id 1-500
Partition 2: car_id 501-1000
Partition 3: car_id 1001-1500
```

**Each partition has local index** on color and make:
```
Partition 1:
  color:red → [car_1, car_7, car_23]
  color:blue → [car_5, car_9]
  make:Honda → [car_1, car_5]
  make:Toyota → [car_7, car_9]

Partition 2:
  color:red → [car_501, car_507]
  color:blue → [car_505]
  make:Honda → [car_501, car_505]
  
Partition 3:
  (similar structure)
```

**Writing**: Only update index in one partition (where document resides)

---

### Querying with Local Indexes

**Query**: Find all red cars

**Problem**: Index is local to each partition!

**Solution**: **Scatter/gather**
- Send query to ALL partitions
- Each partition searches its local index
- Combine results

**Example**:
```
Query: color = red

→ Partition 1: returns [car_1, car_7, car_23]
→ Partition 2: returns [car_501, car_507]
→ Partition 3: returns [car_1001, car_1050]

Combined result: [car_1, car_7, car_23, car_501, car_507, car_1001, car_1050]
```

---

### Advantages and Disadvantages

**Advantages**:
- Writes are fast (only one partition to update)
- Consistency is easier (one partition to keep in sync)

**Disadvantages**:
- Reads are expensive (must query all partitions)
- Even if only need one result, must check everywhere
- **Tail latency amplification**: Slow partition slows down entire query

**Also called**: Local index, document-partitioned index

**Used by**: MongoDB, Riak, Cassandra, Elasticsearch, Solr, VoltDB

---

## Term-Partitioned Indexes (Global Indexes)

**Idea**: Partition the index itself, separately from the documents.

**Global index** covering data in all partitions, but the index itself is partitioned.

---

### Example

**Same e-commerce site**

**Documents partitioned by car_id** as before:
```
Partition 1: car_id 1-500
Partition 2: car_id 501-1000
Partition 3: car_id 1001-1500
```

**Indexes partitioned by term** (the indexed value):
```
Index Partition A: color a-m
  color:blue → [car_5, car_9, car_505]
  make:Honda → [car_1, car_5, car_501, car_505]
  
Index Partition B: color n-z
  color:red → [car_1, car_7, car_23, car_501, car_507, car_1001, car_1050]
  
Index Partition C: make n-z
  make:Toyota → [car_7, car_9, car_1050]
```

**Partitioning the index by term** (the value being indexed):
- color a-m go to partition A
- color n-z go to partition B
- etc.

Could also partition index by hash of term (more even distribution).

---

### Querying with Global Indexes

**Query**: Find all red cars

**Process**:
1. Determine which index partition contains "red" → Partition B
2. Query only Partition B
3. Get complete result from one index partition!

**Advantage**: More efficient reads
- Query touches only relevant index partition
- Not a scatter/gather across all data partitions

---

### Writing with Global Indexes

**Write**: Add new red Honda car (car_id = 1600)

**Process**:
1. Car document goes to Partition 4 (car_id based)
2. Update index for color:red → Partition B
3. Update index for make:Honda → Partition A

**Problem**: Write touches multiple partitions!

**Disadvantage**: Writes slower and more complex
- Distributed transaction across partitions
- Or asynchronous updates (eventual consistency)

---

### Advantages and Disadvantages

**Advantages**:
- Reads are efficient (query one index partition)
- Better for read-heavy workloads

**Disadvantages**:
- Writes are slower (must update multiple partitions)
- Writes are more complex
- Usually asynchronous updates (index may be stale)

**Also called**: Global index, term-partitioned index

**Used by**: Amazon DynamoDB, Riak, Oracle

---

### Global Indexes: Synchronous vs Asynchronous

**Synchronous updates**:
- Write updates all relevant index partitions before acknowledging
- Guarantees index is up-to-date
- Requires distributed transaction (slow, complex)

**Asynchronous updates**:
- Write updates document immediately
- Index updated in background
- Fast writes
- But index may lag behind (eventual consistency)

**In practice**: Most databases use asynchronous updates for global indexes.

**Trade-off**: Fast writes vs. up-to-date indexes.

---

## Rebalancing Partitions

**Rebalancing**: Moving data from one node to another.

**Reasons for rebalancing**:
1. Query throughput increases → add more CPUs
2. Dataset size increases → add more disks/RAM
3. Machine fails → need to take over its responsibilities

**Requirements**:
- After rebalancing, load should be shared fairly
- Database should continue accepting reads/writes during rebalancing
- No more data than necessary should move (minimize network/disk I/O)

---

## Strategies for Rebalancing

### Strategy 1: Hash Mod N (DON'T DO THIS!)

**Idea**: `partition = hash(key) mod N`

**Problem**: When N changes, most keys need to move!

**Example**:
```
Initial: 3 nodes
  hash(key) mod 3
  
Add 1 node: 4 nodes total
  hash(key) mod 4
  
Almost every key goes to different partition!
```

**Result**: Massive data movement, expensive.

**Better approaches** below.

---

### Strategy 2: Fixed Number of Partitions

**Idea**: Create many more partitions than nodes at the start.

**Example**:
```
10 nodes
1000 partitions (100 partitions per node)

Each node handles 100 partitions
```

**When adding node**:
```
Add 11th node
Steal a few partitions from existing nodes
Each existing node gives ~9 partitions to new node
New node ends up with ~90 partitions
```

**Visualization**:
```
Before (10 nodes):
Node 1: [P1, P2, ..., P100]
Node 2: [P101, P102, ..., P200]
...
Node 10: [P901, P902, ..., P1000]

After adding Node 11:
Node 1: [P1, P2, ..., P91]       ← gave away 9 partitions
Node 2: [P101, P102, ..., P191]   ← gave away 9 partitions
...
Node 10: [P901, P902, ..., P991]  ← gave away 9 partitions
Node 11: [P92-100, P192-200, ..., P992-1000]  ← received 90 partitions
```

**Only partitions move**, not individual keys!

**Advantages**:
- Simple
- Only entire partitions move
- Can plan partition assignment in advance

**Disadvantage**:
- Must choose number of partitions at beginning
- If choose too few: partitions become large, expensive to rebalance
- If choose too many: management overhead

**Used by**: Riak, Elasticsearch, Couchbase, Voldemort

**Choosing number of partitions**:
- High enough to accommodate future growth
- Not so high as to incur overhead
- Rule of thumb: size of each partition = max dataset size / partitions per node
- Example: 10 TB total, 10 nodes, 1 TB per node → 100 partitions of 100 GB each

---

### Strategy 3: Dynamic Partitioning

**Idea**: Automatically split or merge partitions based on size.

**Similar to B-trees** in Chapter 3.

**Process**:

**Partition grows too large** (e.g., > 10 GB):
- Split into two partitions
- Half the data goes to each

**Partition shrinks too much** (e.g., < 5 GB after deletes):
- Merge with adjacent partition

**Example**:
```
Initial: 1 partition (all data)

Partition 1 grows to 15 GB → split
  → Partition 1a (7.5 GB)
  → Partition 1b (7.5 GB)

Partition 1a grows to 12 GB → split
  → Partition 1a1 (6 GB)
  → Partition 1a2 (6 GB)

Now have: Partition 1a1, Partition 1a2, Partition 1b
```

**Transfer partitions to balance load**:
- Move Partition 1a2 to different node

**Advantages**:
- Number of partitions adapts to data volume
- Works well for both small and large datasets

**Disadvantages**:
- Empty database starts with single partition (all data on one node initially)
- Can set **pre-splitting**: Create initial set of partitions

**Used by**: HBase, MongoDB, RethinkDB

**Note**: Works with both key-range and hash partitioning.

---

### Strategy 4: Partitioning Proportionally to Nodes

**Idea**: Fixed number of partitions **per node**.

**Example**:
```
10 nodes, 10 partitions per node = 100 partitions total

Add 11th node:
Now have 110 partitions
Randomly choose 10 existing partitions to split
New node takes one half of each split
```

**Process when adding node**:
1. Choose random partitions to split
2. Split each in half
3. New node takes one half, original node keeps other half

**Advantages**:
- Partition size remains stable as nodes are added
- Keeps partition size proportional to dataset size

**Used by**: Cassandra (consistent hashing variant)

---

## Operations: Automatic vs Manual Rebalancing

**Automatic rebalancing**:
- Database decides when and how to move partitions
- Convenient, less operational work

**Risks**:
- Rebalancing is expensive (network, disk I/O)
- Can overload network or nodes
- Combined with automatic failure detection → dangerous
  - Node slow due to overload → detected as dead → triggers rebalancing → makes overload worse!

**Manual rebalancing**:
- Administrator decides when to rebalance
- System generates suggested partition assignment
- Administrator commits the change

**Many databases**: Manual rebalancing by default

**Good practice**: Have a human in the loop, even with automatic rebalancing
- Prevents surprises
- Prevents cascading failures

---

## Request Routing (Service Discovery)

**Problem**: Partitions move between nodes. How does client know which node to contact?

**More general problem**: **Service discovery** - finding the right instance in a distributed system.

---

### Three Approaches to Request Routing

#### Approach 1: Contact Any Node

**Process**:
1. Client contacts any node (random)
2. Node checks: "Do I have this partition?"
   - Yes → Handle request
   - No → Forward to correct node
3. Return result to client

**Visualization**:
```
Client → Node 1 (doesn't have partition) → Node 3 (has partition) → Response
```

---

#### Approach 2: Routing Tier

**Process**:
1. Client contacts routing tier (load balancer)
2. Routing tier knows partition assignment
3. Forwards request to correct node
4. Return result to client

**Visualization**:
```
Client → Routing Tier → Correct Node → Response
```

---

#### Approach 3: Client-Side Routing

**Process**:
1. Client knows partition assignment
2. Client contacts correct node directly

**Visualization**:
```
Client → Correct Node (direct) → Response
```

---

### How Does Each Component Learn About Changes?

**Challenge**: Partition assignments change. How does everyone stay updated?

**Common solution**: Use a **coordination service** like ZooKeeper.

---

### Using ZooKeeper for Coordination

**Architecture**:
```
ZooKeeper Cluster (maintains partition → node mapping)
       ↓ (subscribe to changes)
    Nodes / Routing Tier / Clients
```

**Process**:
1. Each node registers itself in ZooKeeper
2. ZooKeeper maintains authoritative partition assignment
3. Components subscribe to changes in ZooKeeper
4. When partitions move, ZooKeeper notifies subscribers

**Used by**:
- **HBase, Kafka**: Routing tier (or coordinator) uses ZooKeeper
- **MongoDB**: Clients connect to `mongos` routing tier, which queries config servers
- **Cassandra, Riak**: Gossip protocol (nodes share state with each other)

---

### ZooKeeper Example

**Setup**:
```
ZooKeeper tracks:
  Partition 1 → Node A
  Partition 2 → Node B
  Partition 3 → Node A
```

**Routing tier subscribes** to ZooKeeper

**When partition moves**:
```
Partition 2 moves: Node B → Node C

ZooKeeper updates: Partition 2 → Node C
ZooKeeper notifies routing tier
Routing tier updates its local cache
New requests for Partition 2 → go to Node C
```

---

### Database Examples

**Cassandra and Riak**: Gossip protocol
- Nodes communicate directly with each other
- Eventually all nodes learn about changes
- Client can contact any node

**MongoDB**: Config servers + mongos
- Config servers track cluster metadata
- `mongos` instances act as routers
- Clients connect to `mongos`

**HBase, Kafka, SolrCloud**: ZooKeeper
- ZooKeeper tracks partition assignments
- Components subscribe to ZooKeeper

**Espresso (LinkedIn)**: Helix
- Uses ZooKeeper for coordination
- Helix manages partition assignment

---

## Parallel Query Execution

**So far**: Focused on simple queries (read/write single key).

**More complex**: **Massively parallel processing (MPP)** databases.

**Use case**: Analytical queries scanning large portions of dataset.

**Example**: Data warehouse query
```sql
SELECT product, SUM(sales) 
FROM sales_data 
WHERE date >= '2024-01-01' 
GROUP BY product
```

**Execution**:
1. Break query into stages
2. Execute stages in parallel across partitions
3. Combine results

**Example stages**:
```
Stage 1 (parallel): Each partition scans its sales_data, filters by date
Stage 2 (parallel): Each partition groups by product and sums
Stage 3 (serial): Combine sums from all partitions
```

**Techniques**:
- Partition-wise joins
- Shuffle data between nodes
- Map-side vs reduce-side joins (Chapter 10)

**Used by**: Teradata, Netezza, Vertica, SQL-on-Hadoop engines (Impala, Presto, Drill)

**Fast-changing field**: Chapter 10 covers in detail.

---

## Summary: Key Concepts

### 1. Partitioning Methods

| Method | Pros | Cons | Best For |
|--------|------|------|----------|
| **Key Range** | Efficient range queries, keys ordered | Risk of hot spots, manual boundary adjustment | Ordered data, range scans |
| **Hash** | Even distribution, avoids hot spots | No range queries | Uniform access patterns |

---

### 2. Secondary Indexes

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Local (Document-Partitioned)** | Fast writes | Slow reads (scatter/gather) | Write-heavy workloads |
| **Global (Term-Partitioned)** | Fast reads | Slow writes, eventual consistency | Read-heavy workloads |

---

### 3. Rebalancing Strategies

| Strategy | Pros | Cons | Used By |
|----------|------|------|---------|
| **Fixed Partitions** | Simple, predictable | Must choose number upfront | Riak, Couchbase, Voldemort |
| **Dynamic** | Adapts to data size | Empty DB starts with one partition | HBase, MongoDB |
| **Proportional to Nodes** | Stable partition size | More complex | Cassandra |

---

### 4. Request Routing

| Approach | Pros | Cons |
|----------|------|------|
| **Any Node** | Simple client | Extra hop possible |
| **Routing Tier** | Centralized logic | Single point of failure |
| **Client-Side** | Fewest hops | Complex client |

---

## Key Takeaways

1. **Partitioning is essential for scalability** - no single machine can handle unlimited data or queries

2. **Goal is even distribution** - avoid hot spots where one partition gets disproportionate load

3. **Key-range partitioning** enables range queries but risks hot spots

4. **Hash partitioning** distributes load evenly but loses range query efficiency

5. **Secondary indexes complicate partitioning**:
   - Local indexes: fast writes, slow reads
   - Global indexes: fast reads, slow/complex writes

6. **Rebalancing is necessary** but should be done carefully to avoid overloading the system

7. **Coordination is critical** - all components must agree on partition assignments
   - ZooKeeper commonly used
   - Or gossip protocols

8. **Partitioning works with replication** - each partition typically replicated to multiple nodes

9. **Choose partitioning strategy based on access patterns**:
   - Range queries? → Key-range partitioning
   - Uniform access? → Hash partitioning
   - Read-heavy? → Global indexes
   - Write-heavy? → Local indexes

10. **No perfect solution** - every approach involves trade-offs between read performance, write performance, complexity, and operational flexibility
