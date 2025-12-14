# Chapter 9: Consistency and Consensus - Detailed Explanation

## Introduction

**Chapter 8 showed** all the things that can go wrong in distributed systems.

**Chapter 9 shows** how to build reliable systems despite these problems.

**Core challenge**: Getting all nodes to agree on something, even when failures occur.

**This chapter covers**:
- Consistency guarantees
- Distributed transactions
- Consensus algorithms
- How they're used in practice

---

# PART 1: CONSISTENCY GUARANTEES

## Why Consistency Matters

**In a distributed system with replication**, different nodes may have different views of the data.

**Questions**:
- What can the application assume about the data?
- What guarantees does the database provide?

**Consistency models** define what behaviors are possible.

---

## Eventual Consistency

**Weakest consistency model** commonly used.

**Guarantee**: If you stop writing, eventually all replicas will converge to the same value.

**"Eventually"** is deliberately vague:
- Could be milliseconds
- Could be minutes
- No upper bound!

**Problem**: During the convergence period, different replicas return different values.

**Example**:
```
Write x=1 to replica A
Replica A: x=1
Replica B: x=0 (not yet replicated)
Replica C: x=0 (not yet replicated)

Client reads from B: gets x=0
Client reads from A: gets x=1

Inconsistent reads!
```

---

## Stronger Consistency Models

**Applications often need stronger guarantees** than eventual consistency.

**This chapter explores**:
1. **Linearizability** (strongest single-object consistency)
2. **Causality** (partial ordering of events)
3. **Consensus** (getting everyone to agree)

---

# PART 2: LINEARIZABILITY

## What Is Linearizability?

**Also called**: Atomic consistency, strong consistency, immediate consistency, external consistency.

**Intuitive definition**: Make a system appear as if there is only one copy of the data.

**Formal definition**: Once a write completes, all subsequent reads must return the new value.

---

## Linearizability Example

**Scenario**: Two clients writing and reading a register.

**Timeline**:
```
Client A: write(x, 1) ─────────────────────> completes at t1
Client B:                   read(x) ───────> at t2

If t2 > t1: Client B MUST see x=1
```

**Key property**: There is a **single, global, real-time order** of operations.

---

### Non-Linearizable Example

**Scenario**: Alice and Bob watching a football game.

```
Real-time events:
t=0: Goal scored
t=1: Alice's phone shows: "Goal! Score: 1-0"
t=2: Bob checks website, shows: "Score: 0-0" (stale replica)
t=3: Bob's phone shows: "Goal! Score: 1-0"

Alice: "Did you see the goal?"
Bob: "What goal?" (looking at website showing 0-0)
```

**Not linearizable**: Bob saw the future (Alice's notification) before seeing the past (website score).

---

### Linearizable Example

**Same scenario, but linearizable**:

```
t=0: Goal scored, written to database
t=1: Database write completes
t=2: Alice's phone reads, sees goal
t=3: Bob's website read MUST see goal (no stale reads)

Both Alice and Bob see consistent state
```

**Once write completes**: All subsequent reads see it.

---

## What Makes a System Linearizable?

### Requirements

1. **Read-your-writes**: After a write completes, subsequent reads see that value
2. **Monotonic reads**: Once you've seen a value, you never see an older value
3. **Writes appear to happen instantaneously**: At some point between start and completion

---

### Linearizability vs Serializability

**Completely different concepts!**

| Property | Linearizability | Serializability |
|----------|-----------------|-----------------|
| **Scope** | Single operation on single object | Multiple operations on multiple objects |
| **About** | Recency guarantee | Isolation guarantee (transactions) |
| **Guarantee** | Reads return latest value | Transactions appear serial |

**Can have both**: Strict serializability = Serializability + Linearizability

---

## Implementing Linearizability

### Which Systems Are Linearizable?

**Single-leader replication** (with synchronous replication):
- ✓ Can be linearizable (if reads from leader or synchronous follower)
- ✗ Not linearizable (if reads from asynchronous followers)

**Consensus algorithms** (Paxos, Raft, ZAB):
- ✓ Linearizable (designed to be)

**Multi-leader replication**:
- ✗ NOT linearizable (concurrent writes, conflicts)

**Leaderless replication** (Dynamo-style):
- ✗ Usually not linearizable (even with quorums!)
- ✓ Can be with strict quorum reads + sync writes (but poor performance)

---

### Why Quorums Don't Guarantee Linearizability

**Surprising result**: Even w + r > n doesn't guarantee linearizability!

**Example**:
```
n=3, w=2, r=2

Client A: write(x, 1) to replicas 1, 2 (completes)
Client B: read(x) from replicas 2, 3
  Replica 2: x=1 (new)
  Replica 3: x=0 (stale)
  Client B might see x=0! (if picks replica 3's value)

Client C: read(x) from replicas 1, 2
  Both have x=1
  Client C sees x=1

Time order: B's read happened after A's write completed,
but B saw old value!
Not linearizable!
```

**Why?**: Timing issues, network delays, no coordination between reads/writes.

---

## The Cost of Linearizability

### CAP Theorem

**CAP theorem states**: In presence of network partition, must choose between:
- **Consistency** (linearizability)
- **Availability** (every request gets a response)

**Cannot have both** during a partition!

---

### CAP Theorem Example

**Setup**: Multi-datacenter database with network partition.

**Scenario**:
```
Datacenter 1 ←──X──→ Datacenter 2
(partition: cannot communicate)

Client in DC1: write(x, 1)
Client in DC2: read(x)
```

**Option 1: Prioritize Consistency (CP)**
- Block read in DC2 until partition heals
- **Not available** during partition

**Option 2: Prioritize Availability (AP)**
- Allow read in DC2 (returns stale value)
- **Not linearizable**

**Cannot have both!**

---

### Network Partitions Are Inevitable

**Even in a single datacenter**, partitions can occur:
- Network switch failure
- Misconfigured firewall
- Overloaded network

**On the internet**: Even more common.

**CAP is not a binary choice**: It's about what happens **during** partition.

---

### Linearizability and Performance

**Even without network failures**, linearizability has a cost.

**Reason**: Requires coordination/consensus.

**Performance impact**:
- Read/write latency increases
- Throughput decreases

**Many systems sacrifice linearizability** for better performance:
- Eventually consistent databases
- Multi-leader replication

---

# PART 3: ORDERING GUARANTEES

## Why Ordering Matters

**Many consistency problems** come down to **ordering**.

**Examples**:
1. **Leader writes**: Must process in specific order
2. **Serializable transactions**: Appear in some serial order
3. **Timestamps**: Events ordered by time

**Ordering is fundamental** to distributed systems.

---

## Causality

**Causality**: Relationship between cause and effect.

**Example**:
```
Event A: Question "How far can you see into the future?"
Event B: Answer "About 10 seconds"

A causes B (A → B)
B cannot happen before A!
```

**Causal order**: Partial order where some events are causally related, others are concurrent.

---

### Causality Examples in Databases

**Example 1: Foreign Key Constraint**
```
Cause: INSERT INTO users (id=1, name='Alice')
Effect: INSERT INTO posts (user_id=1, text='Hello')

Post insert depends on user existing
User insert must happen before post insert
```

**Example 2: Read-Your-Writes**
```
Cause: User writes comment
Effect: User reads page to see their comment

Read should see the write (causal dependency)
```

**Example 3: Consistent Prefix Reads** (from Chapter 5)
```
Cause: Question asked
Effect: Answer given

Must see question before answer (preserve causality)
```

---

### Causal Consistency

**Causal consistency**: Strongest consistency model that doesn't slow down due to network delays.

**Stronger than**: Eventual consistency

**Weaker than**: Linearizability

**Key difference from linearizability**:
- Linearizability: Total order (all operations in single timeline)
- Causality: Partial order (only causally-related operations ordered)

**Concurrent operations** (not causally related): No defined order.

---

### Linearizability Implies Causality

**Linearizability** → Total order → Includes causal order

**But**: Don't need linearizability just for causality!

**Causal consistency**:
- Good performance (no coordination needed for concurrent operations)
- Makes sense to applications
- Doesn't slow down during network delays

**Many systems** that appear linearizable actually provide causal consistency.

---

## Capturing Causal Dependencies

**How to determine causal order?**

**Need to track**: Which operation happened before another.

---

### Approach 1: Sequence Numbers

**Idea**: Assign a sequence number to each operation.

**Properties**:
- Monotonically increasing
- If A causes B, then seq(A) < seq(B)

**Single-leader**: Simple! Leader assigns sequence numbers.

**Multi-leader or leaderless**: More complex.

---

### Approach 2: Lamport Timestamps

**Lamport timestamps**: Create a total order consistent with causality.

**How it works**:
- Each node has a counter
- Each event gets (counter, node ID)
- Counter increments for each operation

**Ordering rule**:
```
(counter1, nodeID1) < (counter2, nodeID2) if:
  counter1 < counter2, OR
  counter1 == counter2 AND nodeID1 < nodeID2
```

**When receiving message with timestamp (t_msg, node_msg)**:
```
local_counter = max(local_counter, t_msg) + 1
```

---

### Lamport Timestamp Example

```
Node A          Node B
────────        ────────
(1, A) ──msg──→ 
                (1, B)
                (2, B) (received msg, updated counter)
(2, A)
(3, A) ←──msg──
(4, A) (received msg, updated counter)
                (3, B)

Total order: (1,A) < (1,B) < (2,A) < (2,B) < (3,A) < (3,B) < (4,A)
```

**Advantage**: Creates total order without coordination.

**Disadvantage**: Can't tell if two operations are concurrent (just from timestamps).

---

### Version Vectors

**Better than Lamport timestamps** for detecting concurrency.

**Version vector**: One counter per node.

**Example** (3 nodes):
```
Node A: [3, 1, 2]  (3 ops from A, 1 from B seen, 2 from C seen)
Node B: [2, 2, 1]  (2 ops from A seen, 2 from B, 1 from C seen)
```

**Comparison**:
- V1 < V2 if all counters in V1 ≤ corresponding counters in V2, and at least one <
- V1 and V2 concurrent if neither V1 < V2 nor V2 < V1

**Used by**: Riak, Voldemort

---

## Timestamp Ordering Is Not Sufficient

**Problem with sequence numbers/timestamps**: Can't make certain decisions **immediately**.

**Example: Username uniqueness**

```
Node A: User registers username "alice" → Timestamp (1, A)
Node B: User registers username "alice" → Timestamp (1, B)

Which one wins?
(1, A) < (1, B) → Node A wins

But Node A doesn't know about Node B's operation yet!
Can't tell user immediately whether registration succeeded!
```

**Total order alone** doesn't solve the problem.

**Need**: Way to know when decision is **final**.

---

# PART 4: TOTAL ORDER BROADCAST

## What Is Total Order Broadcast?

**Also called**: Atomic broadcast.

**Guarantees**:
1. **Reliable delivery**: If message delivered to one node, delivered to all
2. **Totally ordered delivery**: Messages delivered to all nodes in same order

**Use case**: State machine replication
- All nodes process same commands in same order
- All nodes arrive at same state

---

## Total Order Broadcast Example

**Scenario**: Banking system

```
Node A receives: "Transfer $100 from Alice to Bob"
Node B receives: "Transfer $50 from Bob to Charlie"

Total order broadcast ensures:
ALL nodes process these in SAME order
Maybe: Transfer1 then Transfer2
Or: Transfer2 then Transfer1
But all nodes agree on order!
```

---

## Implementing Total Order Broadcast

**Not trivial!** Requires consensus.

**Used by**:
- ZooKeeper (ZAB protocol)
- etcd, Consul (Raft)

**Once you have total order broadcast**, can build many things:
- Linearizable storage
- Locks
- Uniqueness constraints

---

## Using Total Order Broadcast

### 1. Linearizable Storage

**Can implement linearizable storage** using total order broadcast!

**For compare-and-set** (linearizable operation):
```
1. Append proposed write to log (total order broadcast)
2. Read log, check if anyone else wrote first
3. If you were first → commit, else abort
```

**For reads**:
- Read latest position in log
- Query replica at that position
- Or append read to log (slower but guaranteed linearizable)

---

### 2. Locks

**Distributed lock** using total order broadcast:
```
1. Append "acquire lock X" to log
2. All nodes see messages in same order
3. First message for lock X gets it
4. Others wait
```

**Fencing token**: Use log sequence number!

---

### 3. Uniqueness Constraints

**Username uniqueness**:
```
1. Append "register username alice" to log
2. All nodes process in same order
3. First one to claim "alice" gets it
4. Others get rejection
```

---

## Linearizability vs Total Order Broadcast

**Can implement one from the other**:

**Total order broadcast → Linearizable storage**: (As shown above)

**Linearizable storage → Total order broadcast**:
- Use linearizable compare-and-set register
- Increment counter atomically
- Counter value = sequence number for messages

**They're equivalent!** Both require consensus.

---

# PART 5: DISTRIBUTED TRANSACTIONS AND CONSENSUS

## The Consensus Problem

**Consensus**: Getting all nodes to agree on something.

**Examples**:
- **Leader election**: Nodes must agree on which is leader
- **Atomic commit**: All nodes must agree to commit or abort transaction

**Formal properties**:
1. **Uniform agreement**: All nodes decide the same value
2. **Integrity**: No node decides twice
3. **Validity**: If decided v, then v was proposed by some node
4. **Termination**: Every non-crashed node eventually decides

---

## Why Consensus Is Hard

**FLP Result** (Fischer, Lynch, Paterson, 1985):

**Proven impossible**: No deterministic consensus algorithm that satisfies all properties in asynchronous system with crash-stop failures.

**Sounds bad!** But there's a loophole...

---

### The FLP Loophole

**Asynchronous system**: No timing assumptions.

**In practice**: Systems are **partially synchronous**
- Most of the time, timing assumptions hold
- Occasionally they're violated

**Consensus algorithms work** in partially synchronous systems!

**They use timeouts** to detect failures, but don't rely on timing for correctness.

---

## Atomic Commit and Two-Phase Commit (2PC)

**Scenario**: Transaction spans multiple databases/partitions.

**Requirement**: Either all commit or all abort (atomicity).

**Cannot just commit independently**:
```
Node A: Commits transaction
Node B: Crashes before committing

Result: Inconsistent! Only partial transaction committed.
```

**Need**: Atomic commit protocol.

---

### Two-Phase Commit (2PC)

**Most common atomic commit protocol**.

**Roles**:
- **Coordinator** (transaction manager)
- **Participants** (databases/partitions)

---

### 2PC Protocol

**Phase 1: Prepare**
```
Coordinator:
  1. Sends PREPARE message to all participants
  
Participants:
  2. Check if can commit (no conflicts, constraints satisfied)
  3. Write to transaction log
  4. Reply YES or NO
  
  If YES: Promise to commit if coordinator says so
```

**Phase 2: Commit**
```
Coordinator:
  5. If ALL votes YES → write COMMIT to transaction log
     If ANY vote NO → write ABORT to transaction log
  6. Send COMMIT or ABORT to all participants
  
Participants:
  7. Execute commit or abort
  8. Reply with ACK
```

---

### 2PC Commit Point

**Critical moment**: When coordinator writes COMMIT to its log.

**Before this point**: Can still abort.

**After this point**: Must commit (even if participants crash).

**Participants** are stuck waiting if coordinator crashes at this point!

---

### Problems with 2PC

**1. Blocking Protocol**

**If coordinator crashes**:
```
Participants sent YES
Waiting for COMMIT or ABORT
Coordinator crashes!

Participants BLOCKED! Cannot commit or abort without coordinator.
```

**Must wait** for coordinator to recover (could be long time).

**2. Performance**

- Additional network round-trips (2 phases)
- Synchronous (waits for all participants)
- Disk writes (transaction log)

**Much slower** than single-node transactions.

**3. Coordinator Is Single Point of Failure**

If coordinator's disk is corrupted, participants remain forever uncertain.

---

## Distributed Transactions in Practice

### XA Transactions

**X/Open XA**: Standard API for 2PC.

**Supported by**: Many databases, message queues.

**Example**: Transaction spans PostgreSQL + RabbitMQ.

**Problem**: Locks held until commit → reduced availability.

---

### Database-Internal Distributed Transactions

**Within a single distributed database** (e.g., VoltDB, MySQL Cluster).

**Better than XA**:
- Can optimize (all nodes trust each other)
- Use alternative protocols

---

### Heterogeneous Distributed Transactions

**Across different systems** (different databases, message queues).

**XA standard** attempts to make this work.

**Challenges**:
- Coordinator may be part of application (can crash)
- Orphaned in-doubt transactions (if coordinator forgets)
- Poor performance

**Many modern systems avoid** distributed transactions entirely.

---

## Fault-Tolerant Consensus

**Better than 2PC**: Fault-tolerant consensus algorithms.

**Key difference**: 
- 2PC: Coordinator is SPOF
- Consensus: No single point of failure

**Examples**: Paxos, Raft, ZAB (ZooKeeper), Viewstamped Replication

---

### Consensus Algorithm Properties

**Decided value is one proposed** (validity)

**All nodes decide same value** (agreement)

**Node decides only once** (integrity)

**If majority of nodes working, will reach decision** (termination)

---

### How Consensus Algorithms Work

**High-level overview**:

1. **Leader election**: Choose one node as leader
2. **Proposal**: Leader proposes value
3. **Voting**: Nodes vote on proposal
4. **Decision**: If majority votes yes, value is decided

**Key insight**: Use **epochs** (ballot numbers, view numbers, terms)
- Each leader has unique epoch number
- Higher epoch takes precedence
- Prevents conflicts between multiple leaders

---

### Epoch Numbers and Quorums

**Every time leader elected**: Epoch number incremented.

**Quorum**: Majority of nodes.

**Two rounds of voting**:
1. **Leader election**: Must get votes from quorum
2. **Proposal acceptance**: Must get votes from quorum

**Crucially**: Quorums must overlap
- If quorum elected leader in epoch 1
- And different quorum elects leader in epoch 2
- At least one node in both quorums
- That node carries information forward

**Ensures**: Previous decisions not lost.

---

### Consensus Algorithm: Raft (Simplified)

**Example**: Raft (easier to understand than Paxos)

**Three roles**:
- **Leader**: Accepts client requests, replicates log
- **Follower**: Passive, responds to leader
- **Candidate**: Trying to become leader

**Normal operation**:
```
1. Client sends request to leader
2. Leader appends to its log
3. Leader replicates to followers
4. Once majority have entry, leader commits
5. Leader notifies followers to commit
```

**Leader election**:
```
1. Follower times out (no heartbeat from leader)
2. Becomes candidate, increments term
3. Requests votes from other nodes
4. If gets majority → becomes leader
5. Starts sending heartbeats
```

---

### Consensus Performance

**Limitations**:
- Synchronous replication (waits for quorum)
- Requires strict majority (can't tolerate more than n/2 failures)
- Network issues can cause frequent leader elections

**But**: Much better than 2PC!
- No single point of failure
- Always makes progress (if majority alive)

**Used by**: Most modern distributed databases that need strong consistency.

---

## Membership and Coordination Services

**Consensus algorithms** are typically not used directly by applications.

**Instead**: Wrapped in coordination services.

**Examples**: ZooKeeper, etcd, Consul

---

### ZooKeeper

**Provides**:
- **Linearizable atomic operations**: CAS, increment
- **Total order broadcast**: All operations ordered
- **Failure detection**: Heartbeats, sessions
- **Change notifications**: Subscribers notified of changes

**Built on**: ZAB consensus algorithm.

**Used by**: HBase, Kafka, Hadoop, many others.

---

### ZooKeeper Use Cases

**1. Distributed Locks**
```
Create ephemeral node /locks/my-lock
If created → You have lock
If already exists → Wait and retry
When session ends → Node auto-deleted (lock released)
```

**2. Leader Election**
```
All nodes create ephemeral sequential nodes /election/node-0000000001, etc.
Node with lowest sequence number is leader
Watch next-lowest node
If next-lowest disappears, you might be new leader
```

**3. Service Discovery**
```
Services register by creating nodes
Clients watch parent node for changes
Get notified when services join/leave
```

**4. Configuration Management**
```
Store configuration in ZooKeeper nodes
Applications watch nodes
Get notified when config changes
```

---

### Allocating Work to Nodes

**ZooKeeper** doesn't directly assign work.

**Application uses ZooKeeper** to coordinate:

**Example: Partitioned database**
```
1. ZooKeeper stores partition assignments
2. Nodes subscribe to changes
3. When node joins/fails, rebalancing logic runs
4. New assignment written to ZooKeeper
5. Nodes see change, pick up new partitions
```

---

## Limitations of Consensus

**Consensus is not a silver bullet!**

**Limitations**:

**1. Performance Cost**
- Synchronous replication
- Multiple network round-trips
- Disk writes

**2. Requires Majority**
- Can't tolerate n/2 or more failures
- Geographic distribution tricky (latency across continents)

**3. Fixed Membership**
- Most algorithms assume fixed set of nodes
- Adding/removing nodes requires reconfiguration (complex)

**4. Reliance on Timeouts**
- Failure detection uses timeouts
- Network delays can cause unnecessary elections
- Tuning timeout values is difficult

---

## Alternatives to Consensus

**Many systems** avoid consensus entirely.

**Approaches**:

**1. Leaderless replication** (Dynamo-style)
- No consensus
- Eventual consistency
- Lower latency

**2. Multi-leader replication**
- No consensus
- Conflict resolution
- Better availability

**3. Single-leader replication**
- Simpler (no distributed consensus)
- Manual failover or accept downtime

**Trade-off**: Weaker consistency for better availability/performance.

---

# SUMMARY AND KEY TAKEAWAYS

## Consistency Models Hierarchy

**Strongest to weakest**:

1. **Linearizability**
   - Operations appear instantaneous
   - Total order
   - Expensive (coordination required)

2. **Causal Consistency**
   - Causally-related operations ordered
   - Concurrent operations unordered
   - Good performance

3. **Eventual Consistency**
   - Eventually converges
   - No ordering guarantees during convergence
   - Best performance

---

## Key Concepts

### 1. Linearizability

**What**: System appears as if only one copy of data exists.

**When needed**:
- Locks, leader election
- Uniqueness constraints
- Cross-channel timing dependencies

**Cost**: Performance penalty, not available during network partition.

---

### 2. Causality

**What**: Partial ordering based on cause-and-effect.

**Stronger than**: Eventual consistency

**Weaker than**: Linearizability

**Captured by**: Sequence numbers, Lamport timestamps, version vectors

---

### 3. Consensus

**What**: Getting all nodes to agree.

**Required for**:
- Leader election
- Atomic commit
- Total order broadcast

**Algorithms**: Paxos, Raft, ZAB

**Use via**: ZooKeeper, etcd, Consul

---

### 4. Total Order Broadcast

**What**: Delivering messages to all nodes in same order.

**Equivalent to**: Consensus (can implement one from other)

**Used for**: State machine replication

---

### 5. Two-Phase Commit (2PC)

**What**: Atomic commit protocol for distributed transactions.

**Problem**: Coordinator is single point of failure, can block.

**Better alternative**: Fault-tolerant consensus

---

## Decision Guide

**Need linearizability?**
- Yes → Use consensus (Raft/Paxos via ZooKeeper/etcd)
- No → Can use weaker consistency (better performance)

**Need distributed transactions?**
- Same datacenter → Maybe use 2PC or consensus
- Across datacenters → Avoid if possible (use eventual consistency, sagas)

**Need high availability?**
- Yes → Avoid consensus (use leaderless/multi-leader)
- No → Consensus is fine

**Need strong consistency?**
- Yes → Single-leader or consensus
- No → Multi-leader or leaderless

---

## Key Takeaways

1. **Consistency models are a spectrum**
   - Linearizability (strongest) to eventual consistency (weakest)
   - Stronger consistency = higher cost

2. **Linearizability is expensive**
   - Requires coordination
   - Not available during network partition (CAP theorem)
   - Many systems sacrifice it for availability

3. **Causality is a good middle ground**
   - Captures many application needs
   - Doesn't require coordination for concurrent operations
   - Good performance

4. **Consensus solves many problems**
   - Leader election
   - Atomic commit
   - Linearizable storage
   - But has performance cost

5. **Use consensus via coordination services**
   - Don't implement yourself (extremely hard!)
   - Use ZooKeeper, etcd, Consul
   - Let them handle Raft/Paxos complexity

6. **2PC is not fault-tolerant**
   - Blocking protocol
   - Coordinator is SPOF
   - Use consensus-based atomic commit instead

7. **Most applications can tolerate weaker consistency**
   - Eventual consistency is often sufficient
   - Causal consistency covers many use cases
   - Only use linearizability when truly needed

8. **Consensus is not free**
   - Performance cost
   - Requires majority of nodes
   - Fixed membership challenges
   - Consider if you really need it

9. **Understanding trade-offs is crucial**
   - No perfect solution
   - Choose based on application requirements
   - Consistency vs. availability vs. performance

10. **The next part (Part 3) explores alternatives**
    - Batch processing
    - Stream processing
    - Building systems without distributed transactions
    - Derived data and eventual consistency
