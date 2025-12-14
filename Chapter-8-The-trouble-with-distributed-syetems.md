# Chapter 8: The Trouble with Distributed Systems - Detailed Explanation

## Introduction

**Working with distributed systems is fundamentally different** from programming on a single computer.

**Single computer**: Things mostly work reliably or fail completely.

**Distributed system**: Partial failures are the norm. Some parts work, others don't.

**This chapter**: Understanding what can go wrong, so we can build systems that work despite problems.

---

## The Fundamental Problems

Distributed systems suffer from three main categories of problems:

1. **Networks are unreliable**
2. **Clocks are unreliable**
3. **Processes can pause unexpectedly**

Each creates challenges that don't exist in single-machine systems.

---

# PART 1: UNRELIABLE NETWORKS

## The Network Problem

**Distributed systems** use networks for communication.

**Key insight**: When you send a request over the network, **many things can go wrong**:

1. Request lost (never arrives)
2. Request waiting in a queue (delayed)
3. Remote node has failed
4. Remote node temporarily stopped responding
5. Response lost on network
6. Response delayed

**The sender cannot distinguish** between these scenarios!

---

## Network Faults in Practice

**Example scenario**:
```
Node A sends request to Node B

Case 1: Node B crashed
Case 2: Network cable unplugged
Case 3: Network congested, packet delayed
Case 4: Node B is running but very slow

From Node A's perspective: ALL LOOK THE SAME!
No response.
```

**Only option**: Wait for timeout.

---

### Real-World Network Reliability

**Networks are unreliable**, even in data centers:

**Study findings**:
- In a medium-sized datacenter: ~12 network faults per month
- Could cause 10-50 machines to become unreachable
- EC2 instances: noticeable network issues multiple times per year

**Public internet**: Even worse
- Routers fail
- Packets get lost
- ISP problems

**Conclusion**: Must assume network is unreliable and plan accordingly.

---

## Network Partitions

**Network partition** (netsplit): Network broken into separate parts that can't communicate.

**Example**:
```
Before partition:
Node A ←→ Node B ←→ Node C
(all can communicate)

After partition:
Node A ←→ Node B     Node C
(A and B can talk, but C is isolated)
```

**Even if nodes are working**, they can't communicate.

**Important**: Partitions are a type of network fault.

---

## Detecting Faults

**Question**: How to detect if a node has failed?

**Answer**: You can't be certain!

**Techniques (all imperfect)**:

### 1. Timeouts

**Most common approach**: If no response within timeout, assume node is down.

**Problem**: Can't distinguish:
- Dead node
- Slow node
- Network delay

**Example**:
```
Node A sends request to Node B
Waits 5 seconds
No response → Assumes B is dead

Reality: B is alive but response delayed 6 seconds
```

---

### 2. Choosing Timeout Values

**Timeout too short**:
- False positives (declare node dead when it's just slow)
- Unnecessary failovers
- Cascade failures

**Timeout too long**:
- Slow to detect real failures
- Poor user experience

**No perfect value!**

**Experimentally measure**:
- Network round-trip time distribution
- Application response time
- Set timeout = 2 × 99th percentile

---

### 3. TCP vs Application-Level Timeouts

**TCP**: Retransmits lost packets, but takes a long time to give up (minutes!)

**Application-level timeout**: Typically shorter (seconds)

**Best practice**: Set application-level timeout, don't rely on TCP alone.

---

## Synchronous vs Asynchronous Networks

### Synchronous Networks (Traditional Telephone)

**Circuit switching**: Fixed guaranteed bandwidth.

**Example**: Phone call
- When you dial, circuit is established
- 16 kbps bandwidth reserved for entire call
- Guaranteed latency (bounded delay)

**No queueing delay**: Bandwidth pre-allocated.

---

### Asynchronous Networks (Internet, Data Centers)

**Packet switching**: Best-effort delivery, no guarantees.

**Characteristics**:
- Packets queued if network busy
- Variable latency (unbounded)
- No bandwidth guarantee
- Can drop packets if queue full

**Why?**: Optimized for bursty traffic
- Web browsing: idle → burst → idle
- Email: occasional large attachments
- File transfers: temporary spikes

**Trade-off**: Better utilization but unpredictable latency.

---

### Why Not Use Circuit Switching for Data Centers?

**Seems attractive**: Bounded delays!

**Reality**: Doesn't match data center workloads
- Most traffic is bursty
- Pre-allocating bandwidth wastes resources
- TCP dynamically adapts to available bandwidth

**Current research**: "Quality of Service" (QoS) tries to emulate circuit switching, but limited adoption.

**Conclusion**: Must assume network delays are unbounded.

---

# PART 2: UNRELIABLE CLOCKS

**Why do we need clocks in distributed systems?**

1. **Timeouts**: Deciding when to give up waiting
2. **Timestamps**: Recording when something happened
3. **Measuring duration**: How long did something take?
4. **Ordering events**: Did A happen before B?

**Problem**: Clocks in distributed systems are surprisingly problematic.

---

## Types of Clocks

### 1. Time-of-Day Clocks

**Purpose**: Tells you what time it is (wall-clock time).

**Implementation**: 
- Unix timestamp: seconds/milliseconds since epoch (1970-01-01)
- Example: `System.currentTimeMillis()` (Java), `clock_gettime(CLOCK_REALTIME)` (Linux)

**Synchronized with NTP** (Network Time Protocol)
- Contacts servers periodically
- Adjusts local clock to match

---

#### Problems with Time-of-Day Clocks

**1. Clock Jumps**

**NTP can move clock backward or forward** suddenly!

**Example**:
```
10:00:00 - Current time
10:00:05 - NTP sync discovers local clock is 10 seconds fast
09:59:55 - Clock jumps BACKWARD

Event at 10:00:03 appears to happen AFTER event at 09:59:57!
```

**2. Leap Seconds**

Occasionally a second is added or removed to account for Earth's rotation.

**Some systems handle this poorly**:
- Linux smears it over a day (clock runs slightly slow/fast)
- Some systems crash
- Some ignore it

**3. Time Zone Changes**

DST, political decisions can change time zones.

---

#### NTP Synchronization Accuracy

**Best case** (GPS-based, local network): 1 millisecond

**Typical case** (public NTP servers): 10-100 milliseconds

**Poor case** (congested network, misconfigured): Seconds or worse

**Error can be large!**

---

### 2. Monotonic Clocks

**Purpose**: Measuring elapsed time / duration.

**Guarantees**: Always moves forward (never jumps backward).

**Implementation**:
- `System.nanoTime()` (Java)
- `clock_gettime(CLOCK_MONOTONIC)` (Linux)

**Use cases**:
- Timeouts
- Response time measurement
- Rate limiting

**NOT synchronized across machines!**
- Each machine has its own monotonic clock
- Comparing monotonic times from different machines is meaningless

---

### Which Clock to Use?

| Use Case | Clock Type |
|----------|------------|
| Event timestamp | Time-of-day (but be careful!) |
| Log entry timestamp | Time-of-day |
| Timeout | Monotonic |
| Measure duration | Monotonic |
| Order events on different machines | Neither! (use logical clocks) |

---

## Clock Synchronization and Accuracy

### Relying on Synchronized Clocks is Dangerous

**Example: Distributed database with leader lease**

**Scenario**:
```
Node 1 is leader (lease expires at 10:00:05 according to its clock)
Node 1's clock is fast by 1 minute

Real time 10:00:04 - Node 1 thinks it's 10:01:04
Node 1: "My lease expired, I'll step down"
Node 2: Gets elected as new leader

Real time 10:00:05 - TWO LEADERS! (split brain)
```

**Result**: Data corruption possible.

---

### Last Write Wins (LWW) with Timestamps

**Common pattern**: Resolve conflicts using timestamps.

**Example**:
```
Client A writes x=1 at timestamp 100
Client B writes x=2 at timestamp 101

Database keeps x=2 (higher timestamp wins)
```

**Problem**: If clocks are not synchronized:

```
Client A's clock is fast (timestamp 200)
Client B's clock is slow (timestamp 100)

Client B writes x=2 at t=100
Client A writes x=1 at t=200

Result: x=1 (wrong! B's write should win)
```

**LWW discards data** based on potentially arbitrary timestamp ordering.

**Alternative**: Use version vectors (Chapter 5).

---

### Timestamps for Ordering Events

**Intuition**: If timestamp(A) < timestamp(B), then A happened before B.

**Reality**: NOT ALWAYS TRUE!

**Example**:
```
Real time:
10:00:00 - Event A happens on Node 1
10:00:01 - Event B happens on Node 2

But Node 1's clock is 5 seconds slow:
Event A timestamp: 09:59:55
Event B timestamp: 10:00:01

Appears that B happened 6 seconds after A (wrong!)
```

**Logical clocks** (Lamport timestamps, vector clocks) are better for ordering.

---

## Clock Readings Have a Confidence Interval

**Key insight**: Clock reading has **uncertainty**.

**Example**: Google Spanner

**Most systems**:
- Return single timestamp: 10:00:00.123
- Pretends clock is precise

**Google Spanner** (TrueTime API):
- Returns interval: [10:00:00.123, 10:00:00.130]
- Explicitly acknowledges uncertainty

**Based on**: GPS + atomic clocks (very accurate, but still uncertainty).

**Uncertainty depends on**:
- Time since last NTP sync
- Network latency to NTP server
- Local clock drift rate

---

# PART 3: PROCESS PAUSES

**Even with perfect clocks and networks**, another problem exists: **process pauses**.

**Question**: Can a thread assume nothing else happens while it's running?

**Answer**: NO!

---

## Types of Process Pauses

**Many reasons a running thread can pause**:

1. **Garbage collection** (GC)
   - "Stop the world" GC pauses all threads
   - Can be hundreds of milliseconds
   - Unpredictable timing

2. **Virtual machine suspend**
   - VM hypervisor can suspend VM (e.g., live migration)
   - Pause can be tens of seconds

3. **Context switching**
   - OS switches to another thread
   - If many threads, individual thread gets little CPU time

4. **Paging** (swapping to disk)
   - If memory under pressure, pages swapped to disk
   - Page fault = massive delay

5. **I/O operations**
   - Disk I/O, network I/O can block

6. **SIGSTOP** signal
   - Debugger pausing process
   - Ctrl-Z in terminal

---

## The Danger of Pauses: Lease Example

**Scenario**: Node holds a lease (lock with expiration).

**Assumption**: If lease hasn't expired, we hold exclusive access.

**Code**:
```java
// Get lease from lock service
Lease lease = lockService.getLease();

// Lease valid for 10 seconds
while (lease.isValid()) {
    // We have exclusive access
    writeToDatabase();
}
```

**What can go wrong?**

```
Thread Timeline:

t=0    Get lease (valid until t=10)
t=1    Check lease.isValid() → true
t=2    Long GC pause starts (15 seconds!)
t=17   GC pause ends
t=17   Execute writeToDatabase()

But lease expired at t=10!
Another node may have acquired lease!
```

**Result**: Two nodes writing simultaneously, data corruption.

---

## Fencing Tokens

**Solution**: Use **fencing token** (monotonically increasing).

**How it works**:

1. Lock service issues a token with each lease
2. Client includes token with every write
3. Storage system rejects writes with older tokens

**Example**:
```
t=0   Client 1 gets lease (token=33)
t=5   Client 1 pauses (GC)
t=8   Lease expires
t=9   Client 2 gets lease (token=34)
t=10  Client 2 writes with token=34 ✓ (accepted)
t=15  Client 1 resumes, writes with token=33 ✗ (rejected)
```

**Storage system** remembers highest token seen, rejects lower tokens.

**Used by**: ZooKeeper (`zxid`), etcd

---

## Limiting the Impact of Garbage Collection

**GC pauses are problematic**. How to reduce impact?

### 1. Use GC-Friendly Data Structures

- Immutable data structures (less GC work)
- Off-heap memory (native memory, not managed by GC)

### 2. Short-Lived GC Pauses

- Use generational GC
- Only young generation pauses are short
- But old generation pauses can still be long

### 3. Treat GC as Brief Planned Outage

**Process**:
1. Node warns cluster it's about to GC
2. Other nodes stop sending requests to it
3. Node performs GC
4. Node resumes handling requests

**Used by**: Some HBase configurations

### 4. Use Runtime Without GC

**Examples**:
- C/C++ (manual memory management)
- Rust (ownership system, no GC)
- Go (low-latency GC, but still has pauses)

**Trade-off**: More complex programming, risk of memory leaks/corruption.

---

# PART 4: KNOWLEDGE, TRUTH, AND LIES

## The Truth Is Defined by the Majority

**In distributed systems**: There is no absolute truth.

**When nodes disagree**: Majority vote decides.

**Example**: Is node 5 dead?
```
Node 1: Yes, node 5 is dead (no response to pings)
Node 2: Yes, node 5 is dead
Node 3: No, node 5 is alive (still getting messages)
Node 4: Yes, node 5 is dead

Majority (3 out of 4): Node 5 is dead → Declare it dead
```

Even if node 5 is actually alive but network-partitioned!

---

## The Leader and the Lock

**Many distributed algorithms** need exactly one node to be special:
- Database leader (for writes)
- Lock holder (for exclusive access)
- Partition primary (for consistency)

**How to choose?** Use a **lock** with lease (expires automatically).

---

### Incorrect Implementation of Lock

**Naive approach**:
```
1. Client requests lock from lock service
2. Lock service grants lock (with expiration time)
3. Client assumes it has lock until expiration
```

**Problem**: Client can pause (GC, network delay, etc.)

**Scenario**:
```
Client 1 gets lock (expires in 10 seconds)
Client 1 pauses for 15 seconds
Lock expires at t=10
Client 2 gets lock at t=11
Client 1 resumes at t=15, thinks it still has lock!
Both clients think they have exclusive access!
```

---

### Fencing Tokens: The Correct Solution

**Add monotonically increasing token** to every lock grant.

**Client must include token** with every write.

**Storage checks token**:
- If token > current highest → Accept write, update highest
- If token ≤ current highest → Reject write

**Example**:
```
Client 1 gets lock with token=33
Client 1 pauses
Lock expires
Client 2 gets lock with token=34
Client 2 writes with token=34 → Storage accepts, highest=34
Client 1 resumes, writes with token=33 → Storage rejects (33 ≤ 34)
```

**Requires**:
- Storage system must check tokens
- Cannot rely on lock service alone

---

## Byzantine Faults

**Byzantine fault**: Node behaves in arbitrary, potentially malicious ways.

**Examples**:
- Sending corrupted/contradictory messages
- Pretending to crash, then recovering
- Colluding with other nodes to deceive

**Contrast with**: Crash-stop faults (node stops, doesn't send bad data)

---

### When Byzantine Faults Matter

**Typically NOT a concern** in data centers:
- All nodes controlled by same organization
- No incentive to be malicious
- Software bugs cause crashes, not malicious behavior (usually)

**Byzantine fault tolerance needed when**:
1. **Untrusted environments** (blockchain, peer-to-peer networks)
2. **Aerospace**: Radiation can flip bits in memory
3. **Critical systems**: Where bugs could look like malicious behavior

**Byzantine fault-tolerant algorithms**: Expensive and complex
- Requires more than 3f + 1 nodes to tolerate f Byzantine failures
- Much overhead compared to crash-fault-tolerant algorithms

---

### Weak Forms of Byzantine Fault Protection

**Even if not full Byzantine fault tolerance**, use:

1. **Checksums** (detect corrupted data in network/disk)
2. **Input validation** (sanitize user input)
3. **NTP sanity checks** (reject wildly incorrect time)

**Example**: Jump in time-of-day clock > 100ms → likely error, ignore.

---

## System Model and Reality

**System model**: Abstraction describing what can go wrong.

**Helps reason about** what guarantees are possible.

---

### Timing Assumptions

Three main models:

#### 1. Synchronous Model

**Assumption**: Bounded network delay, bounded process pauses, bounded clock drift.

**Reality**: NOT realistic for most systems.

**Use case**: Real-time systems with hard deadlines.

---

#### 2. Partially Synchronous Model

**Assumption**: System behaves synchronously **most of the time**, but occasionally exceeds bounds.

**Realistic model** for most data centers!

**Example**: 
- Usually network latency < 100ms
- Occasionally spikes to 1 second

---

#### 3. Asynchronous Model

**Assumption**: No timing assumptions at all. No clocks.

**Very restrictive**, but some algorithms work even with this model.

---

### Node Failure Models

Three main models:

#### 1. Crash-Stop Faults

**Node can fail only by crashing**. Once crashed, it never comes back.

#### 2. Crash-Recovery Faults

**Node can crash and come back**. 

**Data in memory** is lost.

**Data on disk** is preserved (assumed durable).

**Most realistic** for typical systems.

#### 3. Byzantine (Arbitrary) Faults

**Nodes can do anything**, including malicious behavior.

---

### Correctness of Algorithms

**Algorithms can provide guarantees** despite faults.

**Example properties**:

**Uniqueness**: Only one node can hold a lock.

**Monotonic sequence**: Fencing tokens must increase.

**Availability**: Algorithm eventually responds (if majority of nodes working).

---

## Safety and Liveness

**Two types of properties**:

### Safety Properties

**Definition**: "Nothing bad happens."

**Characteristics**:
- If violated, can point to specific point in time where it went wrong
- Violation cannot be undone

**Examples**:
- Uniqueness (only one lock holder)
- No data loss
- Consistent reads

**Analogy**: "The door is locked" - can verify at any moment.

---

### Liveness Properties

**Definition**: "Something good eventually happens."

**Characteristics**:
- May not hold at specific point in time, but will hold eventually
- Always hope it might be satisfied in the future

**Examples**:
- Availability (request eventually gets response)
- A message eventually delivered

**Analogy**: "The package will arrive" - can't definitively say it failed until infinite time passes.

---

### Why This Distinction Matters

**In distributed systems**:
- **Safety**: Cannot be violated even if everything goes wrong
- **Liveness**: Allowed to fail temporarily, but must recover

**Example**: Consensus algorithm
- **Safety**: All nodes agree on same value (even during network partition)
- **Liveness**: Algorithm eventually terminates (if majority of nodes working)

**Network partition**:
- Partition can violate liveness (algorithm blocks)
- But must not violate safety (nodes must still agree)

---

# Mapping System Models to Reality

## What Can We Assume?

**Realistic assumptions** for most systems:

1. **Partial synchrony** (timing usually bounded, occasionally not)
2. **Crash-recovery** (nodes crash and recover, disk survives)
3. **Network** can drop, delay, duplicate, or reorder packets

**Cannot assume**:
- Clocks are perfectly synchronized
- Network is reliable
- Nodes never pause

---

## Implications for System Design

### 1. Don't Trust Clocks for Ordering

**Use logical clocks** (Lamport timestamps, vector clocks) to order events.

**If must use physical clocks**:
- Be aware of uncertainty
- Use confidence intervals (like Google Spanner)
- Don't use for strict ordering

---

### 2. Use Timeouts, But Carefully

**Timeouts are necessary** (only way to detect failures).

**But**:
- Choose values based on measurements
- Too short → false positives
- Too long → poor availability

---

### 3. Expect and Handle Process Pauses

**Don't assume** thread runs continuously.

**Use techniques** like:
- Fencing tokens
- Lease expiration checks
- Idempotent operations

---

### 4. Test for Rare Events

**Problems often hidden** under normal conditions.

**Deliberately inject faults**:
- Kill processes randomly (chaos monkey)
- Disconnect network
- Slow down nodes
- Desynchronize clocks

**Jepsen**: Tool that tests distributed systems by injecting faults.

---

## Real-World Examples of Problems

### 1. GitHub Outage (2012)

**Cause**: Clock on MySQL server jumped forward.

**Effect**: 
- Replication broke
- Service offline for hours

---

### 2. Cloudflare Leap Second (2017)

**Cause**: Leap second caused time to go backward.

**Effect**:
- Edge servers crashed
- DNS resolution failed
- Global outage

---

### 3. AWS Clock Drift

**Problem**: EC2 instances' clocks can drift significantly.

**Recommendation**: Use Amazon Time Sync Service or NTP.

---

## Summary: The Fundamental Problems

### 1. Networks Are Unreliable

**Reality**:
- Packets can be lost, delayed, duplicated
- Network partitions occur
- Cannot distinguish crashed node from slow node

**Implication**:
- Use timeouts (but choose carefully)
- Design for partition tolerance
- Test with network faults

---

### 2. Clocks Are Unreliable

**Reality**:
- Clocks can jump backward/forward
- Clocks drift at different rates
- Synchronization is imperfect

**Implication**:
- Don't rely on clocks for ordering
- Use logical clocks when ordering matters
- Be aware of clock uncertainty

---

### 3. Processes Pause Unexpectedly

**Reality**:
- GC pauses
- VM suspensions
- Operating system scheduling
- Disk I/O

**Implication**:
- Don't assume continuous execution
- Use fencing tokens
- Design for pauses

---

## Key Takeaways

1. **Distributed systems are fundamentally different** from single-machine systems
   - Partial failures are the norm
   - Cannot make strong timing assumptions

2. **Networks are unreliable**
   - Packets lost, delayed, duplicated
   - Partitions happen
   - Timeouts only imperfect way to detect failures

3. **Clocks are problematic**
   - Not synchronized across machines
   - Can jump backward
   - Don't use for ordering (use logical clocks)

4. **Process pauses are real**
   - GC, VM suspend, context switching
   - Use fencing tokens to protect critical sections

5. **Byzantine faults** usually not a concern in data centers
   - But use checksums, input validation

6. **System models help reason** about what's possible
   - Partial synchrony + crash-recovery is realistic
   - Safety properties must hold always
   - Liveness properties can fail temporarily

7. **Test with faults** injected
   - Only way to build confidence
   - Rare events will eventually happen

8. **No perfect solution**
   - Every design involves trade-offs
   - Understanding failure modes enables informed decisions

9. **The next chapter (9) builds on this**
   - Shows how to build reliable systems despite these problems
   - Consensus algorithms, coordination services

10. **Humility is essential**
    - Distributed systems are hard
    - Expect things to fail
    - Design for failure recovery
