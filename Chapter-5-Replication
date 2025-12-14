# Chapter 5: Replication - Detailed Explanation

## What Is Replication?

**Replication** means keeping a copy of the same data on multiple machines connected via a network.

### Why Replicate Data?

1. **Reduce latency**: Keep data geographically close to users
2. **Increase availability**: System continues working even if some parts fail
3. **Increase read throughput**: Serve more read queries by scaling out

### The Challenge

The difficulty in replication is **handling changes to replicated data**. All the complexity comes from ensuring that changes are applied to all replicas.

---

## Three Main Replication Approaches

1. **Single-leader** (master-slave)
2. **Multi-leader** (master-master)
3. **Leaderless** (peer-to-peer)

---

## Single-Leader Replication

### Architecture

**Components**:
- **Leader (master/primary)**: Accepts all writes
- **Followers (slaves/replicas/secondaries)**: Receive changes from leader

**Flow**:
```
Client ──write──> Leader
                    │
                    ├──> Follower 1
                    ├──> Follower 2
                    └──> Follower 3

Clients ──read──> Any replica
```

**Used by**: PostgreSQL, MySQL, MongoDB, RabbitMQ, Kafka

### How It Works

1. Client sends write to leader
2. Leader writes to local storage
3. Leader sends change to followers via **replication log** or **change stream**
4. Followers apply changes in same order as leader
5. Clients can read from any replica

---

### Synchronous vs Asynchronous Replication

#### Synchronous Replication

Leader waits for confirmation from follower before reporting success.

**Example**:
```
Client ──write──> Leader ──waits for──> Follower 1 ✓
                    │                    Follower 2 (async)
                    └──────────────────> Follower 3 (async)
                    
Leader confirms to client only after Follower 1 acknowledges
```

**Advantages**:
- Follower guaranteed to have up-to-date copy
- If leader fails, data is not lost

**Disadvantages**:
- If synchronous follower doesn't respond, write blocks
- Even one node failure can halt all writes

#### Asynchronous Replication

Leader sends change to followers but doesn't wait for acknowledgment.

**Advantages**:
- Leader can continue processing writes even if all followers fail
- Lower latency for writes
- Better write throughput

**Disadvantages**:
- If leader fails, writes not yet replicated are lost
- Followers may lag behind leader significantly

#### Semi-Synchronous

**Practical compromise**: One follower is synchronous, others are asynchronous.

If the synchronous follower becomes unavailable, make another follower synchronous.

**Guarantees**:
- At least two nodes have up-to-date copy
- Doesn't block writes if one node is slow

---

### Setting Up New Followers

**Challenge**: How to set up a new follower without downtime?

**Cannot** simply copy files (data is constantly changing).

**Solution**:
1. Take a **consistent snapshot** of leader's database
2. Copy snapshot to new follower
3. Follower requests all changes since snapshot (using snapshot's position in replication log)
4. Follower "catches up" by applying these changes
5. Now follower is ready to serve reads

**Example positions**:
- PostgreSQL: log sequence number
- MySQL: binlog coordinates

---

### Handling Node Outages

#### Follower Failure: Catch-up Recovery

**Scenario**: Follower crashes or network is interrupted

**Recovery**:
1. Follower knows last transaction it processed (from local log)
2. Reconnects to leader
3. Requests all changes since that point
4. Applies changes to catch up

**Simple and works well!**

#### Leader Failure: Failover

**Much more complex!** One of the followers must be promoted to leader.

**Failover process**:
1. **Determine leader has failed**: Use timeout (no response in 30 seconds)
2. **Choose new leader**: 
   - Usually replica with most up-to-date data
   - Or election algorithm (consensus)
   - Or appointed by administrator
3. **Reconfigure system**: 
   - Clients send writes to new leader
   - Old leader becomes follower when it recovers

#### Problems with Failover

**1. Data Loss (Asynchronous Replication)**

**Example**:
```
Old leader:  writes W1, W2, W3, W4 (only W1, W2 replicated)
             crashes before W3, W4 reach followers

Follower promoted to leader (only has W1, W2)
Old leader recovers with W3, W4 in its log

What to do with W3, W4?
- Discard them? → Data loss!
- Keep them? → Conflicts with new leader's writes!
```

Most systems discard unreplicated writes → **data loss**.

**2. Split Brain**

**Scenario**: Both old and new leader think they're the leader

```
Network partition

Old leader ──X──[network partition]──X── Followers
(still alive)                            (elect new leader)

Both accept writes! → Data divergence!
```

**Solution**: 
- **STONITH** (Shoot The Other Node In The Head)
- Shutdown mechanism if two leaders detected
- Otherwise, data corruption risk

**3. Timeout Too Short or Too Long**

- **Too short**: Unnecessary failovers (temporary glitch triggers it)
- **Too long**: Longer recovery time

No perfect setting - depends on your system.

**Reality**: Many ops teams prefer manual failover due to complexity.

---

### Replication Logs Implementation

#### Statement-Based Replication

**Idea**: Leader logs every write statement (INSERT, UPDATE, DELETE) and sends to followers.

**Example**:
```sql
Leader executes: UPDATE products SET views = views + 1 WHERE id = 123
Followers execute same statement
```

**Problems**:
- **Nondeterministic functions**: `NOW()`, `RAND()` produce different results on each replica
- **Auto-incrementing columns**: Order-dependent
- **Side effects**: Triggers, stored procedures may have different effects

**Solution**: Leader can replace nondeterministic functions with fixed values.

**Used historically by**: MySQL (before v5.1)

#### Write-Ahead Log (WAL) Shipping

**Idea**: Leader sends its append-only log of all writes to followers.

**How databases use logs**:
- B-trees: Write-ahead log records all modifications
- Log-structured storage: Log is the main storage

**In replication**: Send same log to followers.

**Used by**: PostgreSQL, Oracle

**Disadvantage**: Log describes data at very low level (which bytes changed in which disk blocks)
- **Tightly coupled to storage engine**
- Can't run different database versions on leader and followers
- Makes zero-downtime upgrades difficult

#### Logical (Row-Based) Log Replication

**Idea**: Use different log format for replication (decoupled from storage engine)

**Logical log** describes writes at row level:
- For INSERT: new values of all columns
- For DELETE: information to uniquely identify deleted row
- For UPDATE: information to identify row + new values

**Example**:
```
INSERT INTO users: id=5, name='Alice', email='alice@example.com'
UPDATE users WHERE id=5: email='alice@newdomain.com'
DELETE FROM users WHERE id=5
```

**Advantages**:
- Decoupled from storage engine internals
- Easier to keep backward compatible
- Easier for external applications to parse
- Can send to data warehouses, caches, search indexes

**Used by**: MySQL binlog (in row-based mode)

#### Trigger-Based Replication

**All previous methods**: Implemented by database system

**Alternative**: Move replication to application layer

**Approach**: Use **triggers** and **stored procedures**
- Trigger logs change to separate table
- External process reads that table and replicates

**Tools**: Oracle GoldenGate, Databus (for Oracle), Bucardo (for PostgreSQL)

**Advantages**:
- Flexible: can replicate subset of data, transform data, replicate to different database
- Can add custom conflict resolution logic

**Disadvantages**:
- Greater overhead
- More prone to bugs
- More limited than built-in replication

---

## Problems with Replication Lag

With asynchronous replication, followers may lag behind leader. Eventually they catch up (**eventual consistency**), but temporarily they're inconsistent.

**"Eventual"** is deliberately vague - might be 1 second or 1 minute!

### Three Main Problems

---

### 1. Reading Your Own Writes

**Problem**: User makes a write, then immediately reads from a follower that hasn't caught up yet. Appears as if write was lost!

**Example**:
```
User posts comment ──> Leader (write succeeds)
User refreshes page ──> Follower (doesn't have new comment yet)
User: "Where's my comment?!"
```

**Solution: Read-After-Write Consistency**

Guarantee that user sees their own updates immediately.

**Implementation strategies**:

1. **Read user's own data from leader**
   ```
   if (reading user's own profile):
       read from leader
   else:
       read from follower
   ```

2. **Track time of last update**
   - For 1 minute after update, read from leader
   - After that, read from follower

3. **Client remembers timestamp of recent write**
   - Ensure replica serving read is caught up to that timestamp
   - If replica too far behind, read from another or wait

4. **Cross-device consistency**
   - User updates on laptop, views on phone
   - Timestamp approach works
   - Route user's requests to same datacenter

---

### 2. Monotonic Reads

**Problem**: User makes multiple reads from different replicas, sees data moving backward in time!

**Example**:
```
Time 1: User reads from Follower 1 (up-to-date)
        Sees comments: A, B, C

Time 2: User reads from Follower 2 (lagging)
        Sees comments: A, B
        
User: "Where did comment C go?!"
```

**Solution: Monotonic Reads**

If a user makes several reads in sequence, they won't see time go backwards.

**Weaker than strong consistency**, but stronger than eventual consistency.

**Implementation**:
- Each user always reads from same replica (could be based on hash of user ID)
- If that replica fails, reroute to another

---

### 3. Consistent Prefix Reads

**Problem**: Violation of causality - seeing effects before causes.

**Example** (conversation):
```
Mr. Poons: "How far into the future can you see?"
Mrs. Cake: "About ten seconds usually."
```

**If writes go to different partitions**:
```
Partition 1: Mrs. Cake's answer arrives first
Partition 2: Mr. Poons' question arrives second

Observer sees:
Mrs. Cake: "About ten seconds usually."
Mr. Poons: "How far into the future can you see?"

Makes no sense!
```

**Solution: Consistent Prefix Reads**

If sequence of writes happens in certain order, anyone reading them sees them in that same order.

**Particularly a problem in partitioned databases** where different partitions operate independently.

**Implementation**:
- Ensure causally related writes go to same partition
- Or use algorithms that explicitly track causal dependencies (version vectors)

---

### Solutions for Replication Lag

1. **Pretend replication is synchronous** when it's not → Eventually causes problems
2. **Make application code more complex** to handle inconsistency
3. **Use transactions** with stronger guarantees (but impacts performance/availability)

The right choice depends on your application requirements.

---

## Multi-Leader Replication

### Motivation

Single-leader limitation: All writes must go through one leader.

**Use cases for multiple leaders**:

---

### 1. Multi-Datacenter Operation

**Single-leader architecture**:
```
Datacenter 1: Leader
Datacenter 2: Followers
Datacenter 3: Followers

All writes go to Datacenter 1!
```

**Multi-leader architecture**:
```
Datacenter 1: Leader + Followers
Datacenter 2: Leader + Followers
Datacenter 3: Leader + Followers

Writes go to local datacenter's leader
```

**Comparison**:

| Aspect | Single-Leader | Multi-Leader |
|--------|---------------|--------------|
| Performance | Inter-datacenter latency on every write | Each datacenter processes writes locally |
| Tolerance of outages | Failover promotes follower in another datacenter | Each datacenter operates independently |
| Tolerance of network problems | Very sensitive to inter-datacenter link | Can tolerate datacenter disconnection |

**Disadvantage**: Must handle write conflicts (two datacenters modify same data)

**Used by**: MySQL (Tungsten Replicator), PostgreSQL (BDR), Oracle GoldenGate

---

### 2. Clients with Offline Operation

**Example**: Calendar app, note-taking app

**Architecture**:
```
Each device acts as a "datacenter" with its own local leader

Phone ──┐
Laptop ─┼──> Sync when online
Tablet ─┘
```

When device is online, it syncs with other devices. Every device is a "leader."

**Same problem**: Handling write conflicts when syncing

**Technologies**: CouchDB, PouchDB designed for this

---

### 3. Collaborative Editing

**Example**: Google Docs, Etherpad

**Challenge**: Multiple people editing simultaneously

**Approach 1**: Single-leader (lock document while editing)
- Poor experience

**Approach 2**: Multi-leader (each user's changes are immediately applied locally)
- Must handle conflicts when syncing
- Uses algorithms like **Operational Transformation** or **CRDTs**

---

### Handling Write Conflicts

**Core problem of multi-leader replication!**

**Example**:
```
User 1 in Datacenter 1:  UPDATE wiki_page SET title = 'A' WHERE id = 123
User 2 in Datacenter 2:  UPDATE wiki_page SET title = 'B' WHERE id = 123

(Both execute simultaneously in their local datacenter)

When replicating, what should title be?
```

#### Conflict Avoidance (Best Strategy!)

**Idea**: Ensure writes for a particular record always go through same leader.

**Example**:
- User's data always goes to datacenter closest to them
- As long as user doesn't change location, no conflicts

**Problem**: If datacenter fails or user moves, might need to reroute → conflicts possible

---

#### Converging Toward a Consistent State

Single-leader: Writes are serialized (have sequential order)

Multi-leader: No defined ordering!

**Database must resolve conflict in convergent way**: All replicas must arrive at same final value.

**Strategies**:

1. **Last Write Wins (LWW)**
   - Assign each write a unique ID (timestamp, UUID, hash)
   - Pick write with highest ID
   - **Data loss!** Other writes discarded

2. **Replica with Higher ID Wins**
   - Arbitrary but deterministic

3. **Merge Values**
   - Title becomes "A/B" (concatenate)
   - Works for some data types

4. **Record Conflict**
   - Preserve both versions
   - Application code resolves later (or prompts user)

---

#### Custom Conflict Resolution

**On write**: When conflict detected, call conflict handler
- Example: Bucardo allows Perl scripts

**On read**: Store all conflicting versions, application resolves when reading
- Example: CouchDB returns all versions to application

**Conflict-Free Replicated Datatypes (CRDTs)**:
- Data structures that automatically resolve conflicts in sensible ways
- Examples: Sets that can be merged, counters that preserve increments
- Used in Riak

**Operational Transformation**:
- For collaborative editing (Google Docs)
- Designed for concurrent editing of ordered lists (text documents)

---

### Multi-Leader Replication Topologies

**How leaders send changes to each other?**

#### All-to-All Topology
```
     Leader 1
    /    |    \
Leader 2  |  Leader 3
    \    |    /
     (everyone connects to everyone)
```

**Most general and commonly used**

---

#### Circular Topology
```
Leader 1 → Leader 2 → Leader 3 → Leader 1
```

Each node forwards writes to next node.

**Problem**: If one node fails, chain breaks

---

#### Star Topology
```
        Leader 1 (root)
        /      \
    Leader 2  Leader 3
```

One designated root forwards changes.

**Problem**: If root fails, topology breaks

---

#### Problems with Topologies

**Circular and Star**: Single point of failure

**All-to-All**: Some network links faster than others → **causality problems**

**Example**:
```
Leader 1: INSERT INTO users (id=1, name='Alice')
Leader 1: INSERT INTO posts (user_id=1, text='Hello')

          Insert user
Leader 1 ──────────────> Leader 2 (slow path)
    │
    │       Insert post
    └────────────────────> Leader 3 (fast path)

Leader 3 receives post before user!
Foreign key constraint violation!
```

**Solution**: Version vectors track causality (discussed later)

---

## Leaderless Replication

No leader! Any replica can accept writes from clients.

**Used by**: Amazon's Dynamo (internal), Riak, Cassandra, Voldemort

Sometimes called **Dynamo-style** databases.

---

### Writing to the Database

**Client sends write to several replicas in parallel** (or coordinator node does it).

**Example** (n=3 replicas):
```
Client ──write──┬──> Replica 1 ✓
                ├──> Replica 2 ✓
                └──> Replica 3 ✗ (unavailable)

Write succeeds! (2 out of 3 acknowledged)
```

---

### Reading from the Database

**Client sends read to several replicas in parallel**.

**Why?** Some replicas might have stale data.

**Example**:
```
Client ──read──┬──> Replica 1: returns value v1
               ├──> Replica 2: returns value v1
               └──> Replica 3: returns value v2 (stale)

Client uses version numbers to determine which is newest
```

---

### Quorums for Reading and Writing

**Configuration**:
- `n` = number of replicas
- `w` = number of writes that must succeed
- `r` = number of reads required

**Rule**: As long as `w + r > n`, expect to get up-to-date value when reading.

**Common configuration**: `n=3, w=2, r=2`

**Why it works**:
- `w + r > n` means at least one of the `r` replicas must be up-to-date
- There's overlap between `w` write replicas and `r` read replicas

**Example**:
```
n=3, w=2, r=2

Write succeeds to Replicas 1 and 2
Read queries Replicas 2 and 3
Replica 2 is in both sets → has latest value!
```

---

### Quorum Configurations

**Typical choices**:

1. **`n=3, w=2, r=2`**: Tolerate 1 unavailable node
2. **`n=5, w=3, r=3`**: Tolerate 2 unavailable nodes

**Reads and writes always sent to all `n` replicas**, but only wait for `w` or `r` responses.

---

#### Handling Unavailable Nodes

**Scenario**: Write sent to 3 replicas, only 2 available

```
Replica 1: ✓ (write succeeds)
Replica 2: ✓ (write succeeds)
Replica 3: ✗ (temporarily unavailable)
```

**When Replica 3 comes back online**, how does it catch up?

---

### Catching Up: Read Repair and Anti-Entropy

#### 1. Read Repair

**Process**:
- Client reads from several replicas
- Detects stale responses (by version number)
- Writes newer value back to stale replicas

**Good for**: Frequently read values

**Example**:
```
Client reads from Replicas 1, 2, 3
Replica 3 has stale value v1
Client writes latest value v2 to Replica 3
```

---

#### 2. Anti-Entropy Process

**Background process** constantly looks for differences between replicas and copies missing data.

**Unlike replication log**: Doesn't copy writes in any particular order, may have significant delay.

**Used by**: Cassandra, Riak, Voldemort

---

### Sloppy Quorums and Hinted Handoff

**Problem**: Network interruption cuts off client from replicas

**Example**:
```
Client can reach Replicas 1, 2 but not Replica 3
Needs w=2 confirmations
Should write succeed?
```

**Trade-off**:
1. **Return errors** (refuse write) → Availability reduced
2. **Accept write anyway** to reachable replicas → May not be the "designated" replicas

---

#### Sloppy Quorum

**Accept writes** on nodes that are reachable, even if they're not among the `n` "home" nodes.

**Example**:
```
Designated nodes: Replicas 1, 2, 3
Replica 3 unreachable

Write temporarily goes to Replica 4
When Replica 3 recovers, Replica 4 sends data to Replica 3
```

This temporary handoff is called **hinted handoff**.

**Advantage**: Increases write availability

**Disadvantage**: Even with `w + r > n`, can't guarantee reading latest value
- Might read from designated nodes that don't have latest write yet

**Optional in**: Riak, Cassandra

---

### Detecting Concurrent Writes

**Problem**: Multiple clients writing to same key simultaneously

**Example**:
```
Client 1: SET cart = {milk, eggs}
Client 2: SET cart = {flour, sugar}

(Both writes happen concurrently to different replicas)

Final value should be...?
```

---

#### Last Write Wins (LWW)

**Approach**: Attach timestamp, keep value with latest timestamp.

**Problem**: **Data loss!** Concurrent writes, one arbitrarily discarded.

**Only safe if** values are immutable (never updated after creation).

**Used by**: Cassandra (default), Riak (optional)

---

#### The "Happens-Before" Relationship

**Key insight**: Events are **concurrent** if neither happens before the other.

**Example**:
```
Client 1: Write A
Client 1: Write B (saw A first)

A happens-before B → Not concurrent
```

```
Client 1: Write A
Client 2: Write B (hasn't seen A)

A and B are concurrent → Must preserve both!
```

---

#### Version Numbers and Causality

**Solution**: Use version numbers to track causality.

**Algorithm**:

1. Server maintains version number for each key
2. Increments version number every time key is written
3. Stores new value along with version number

**Client reads**:
- Server returns all current values with version numbers

**Client writes**:
- Must include version number from prior read
- Must merge together all values received in prior read

**Example flow**:
```
Client 1: GET cart
  Server: returns [] (version 1)

Client 1: PUT cart = [milk] (version 1)
  Server: stores [milk] (version 2)

Client 2: GET cart
  Server: returns [milk] (version 2)

Client 2: PUT cart = [milk, eggs] (version 2)
  Server: stores [milk, eggs] (version 3)

Client 1: GET cart (still has version 1!)
  Server: returns [milk, eggs] (version 3)
  
Client 1: PUT cart = [milk, flour] (version 1)
  Server: stores [milk, flour] (version 4)
  Server: Detects conflict! (version 1 < version 3)
  Server: Keeps both: [milk, eggs] and [milk, flour]

Client 3: GET cart
  Server: returns both [milk, eggs] (v3) and [milk, flour] (v4)
  Client 3: Must merge! → [milk, eggs, flour]
  
Client 3: PUT cart = [milk, eggs, flour] (versions [3,4])
  Server: stores [milk, eggs, flour] (version 5)
```

**Key point**: Version numbers determine **happens-before** relationships.

---

#### Version Vectors

**Problem**: Previous algorithm works with single replica.

**Multiple replicas**: Need version number **per replica**.

**Version vector**: Collection of version numbers from all replicas.

**Example** (3 replicas):
```
Replica A: version 3
Replica B: version 1
Replica C: version 2

Version vector: [A:3, B:1, C:2]
```

**Used by**: Riak (called "dotted version vectors"), Cassandra (used internally)

**Client must read** version vector, merge conflicts, send back merged value with version vector.

---

## Summary: Comparison of Replication Methods

### Single-Leader Replication

**Pros**:
- Simple to understand and implement
- No write conflicts
- Reads can scale out

**Cons**:
- All writes go through one node (bottleneck)
- Leader failure requires failover (complex)
- Replication lag causes read inconsistency

**Best for**: 
- Traditional applications
- When writes are less frequent than reads
- When strong consistency matters

---

### Multi-Leader Replication

**Pros**:
- Better performance in multi-datacenter setups
- Tolerance of datacenter failures
- Tolerance of network problems

**Cons**:
- Write conflicts!
- Complex to reason about
- Weak consistency guarantees

**Best for**:
- Multi-datacenter deployments
- Offline-first applications
- Collaborative editing

---

### Leaderless Replication

**Pros**:
- High availability (no failover needed)
- Low latency (read/write locally)
- Tolerates node failures well

**Cons**:
- Concurrent writes cause conflicts
- Read repair and anti-entropy overhead
- Version vectors complex for clients

**Best for**:
- High availability requirements
- Can tolerate eventual consistency
- Need resilience to network partitions

---

## Key Takeaways

1. **Replication lag is inevitable** with asynchronous replication
   - Read-after-write consistency
   - Monotonic reads
   - Consistent prefix reads

2. **Multi-leader and leaderless** avoid single point of failure but introduce **write conflicts**

3. **Conflict resolution** is hard:
   - Avoid conflicts if possible
   - Converge to consistent state
   - May require application logic

4. **Quorums** (`w + r > n`) provide eventual consistency in leaderless systems

5. **Version vectors** track causality and detect concurrent writes

6. **Choose replication method** based on your requirements:
   - Need strong consistency → Single-leader (or synchronous replication)
   - Need high availability → Leaderless
   - Need multi-datacenter → Multi-leader

7. **There's no perfect solution** - all involve trade-offs between consistency, availability, and performance
