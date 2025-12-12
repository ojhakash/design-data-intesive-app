# Chapter 7: Transactions - Detailed Explanation

## What Are Transactions?

A **transaction** is a way for an application to group several reads and writes together into a logical unit. The entire transaction either succeeds (commit) or fails (abort/rollback). If it fails, the application can safely retry. This simplifies the programming model because you don't need to worry about partial failures.

### Why Transactions Matter

Without transactions, handling errors becomes extremely complex:
- What if the system crashes halfway through a series of writes?
- What if two users try to modify the same data simultaneously?
- What if a read sees partially updated data?

Transactions provide **safety guarantees** that make these problems manageable.

---

## ACID Properties

ACID is an acronym describing the properties that transactions should have:

### Atomicity
**Meaning**: All-or-nothing execution. If a transaction involves multiple writes, either all succeed or all fail.

**Example**: Transferring money between bank accounts
```
BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE account_id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE account_id = 2;
COMMIT
```
If the system crashes after the first UPDATE, atomicity ensures the first update is rolled back. You can't end up with money disappearing!

**Implementation**: Typically uses a write-ahead log (WAL). If a crash occurs, the database can replay or undo operations.

### Consistency
**Meaning**: The database moves from one valid state to another valid state, maintaining application-specific invariants.

**Reality**: This is somewhat vague and actually depends more on the application than the database. If you write bad data, the database won't stop you unless you've defined constraints.

**Example**: If you have a constraint that "account balance must be non-negative," the database enforces this, but more complex business rules (like "total credits must equal total debits") must be enforced by the application.

### Isolation
**Meaning**: Concurrently executing transactions are isolated from each other. They appear to execute serially, even if they actually run in parallel.

**This is the most complex property** and has multiple levels (explained in detail below).

### Durability
**Meaning**: Once a transaction is committed, its data won't be lost, even if the system crashes.

**Implementation**: 
- Write-ahead logs written to disk
- Replication to multiple machines
- Backups

**Reality**: Perfect durability is impossible (what if all datacenters burn down?), but systems aim for very high durability.

---

## Weak Isolation Levels

True serializability (perfect isolation) has a performance cost. Many databases offer weaker isolation levels that provide some protection while performing better.

### 1. Read Committed

**Two guarantees:**
1. **No dirty reads**: You only see committed data
2. **No dirty writes**: You only overwrite committed data

#### No Dirty Reads

**Problem**: Imagine a transaction that updates multiple related values. If another transaction can read some updated values before the first transaction commits, it sees inconsistent state.

**Example**:
```
Transaction 1:
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;
  -- (crash before commit)

Transaction 2 (reads between the updates):
  SELECT * FROM accounts WHERE id IN (1, 2);
  -- Sees account 1 with $100 less, but account 2 unchanged
  -- Money appears to have vanished!
```

**Solution**: Keep both old and new values. When another transaction reads, it sees the old value until the writing transaction commits.

#### No Dirty Writes

**Problem**: Two transactions both overwriting the same objects can cause corruption.

**Example**: Listing a used car for sale
```
Transaction 1 (Alice):
  UPDATE listings SET seller = 'Alice', price = 5000 WHERE car_id = 123;
  UPDATE inventory SET owner = 'Alice' WHERE car_id = 123;

Transaction 2 (Bob):
  UPDATE listings SET seller = 'Bob', price = 4500 WHERE car_id = 123;
  UPDATE inventory SET owner = 'Bob' WHERE car_id = 123;
```

If these interleave improperly, you could end up with `seller = 'Alice'` but `owner = 'Bob'`!

**Solution**: Use row-level locks. When a transaction wants to modify a row, it must first acquire a lock. The lock is held until commit or abort.

#### Limitations of Read Committed

Read committed prevents dirty reads and writes, but **it doesn't prevent all race conditions**:

**Read Skew Example**: 
```
Account 1: $500
Account 2: $500
Total: $1000

Transaction (checking total balance):
  balance1 = SELECT balance FROM accounts WHERE id = 1;  -- reads $500
  -- Meanwhile, another transaction transfers $100 from account 1 to account 2 and commits
  balance2 = SELECT balance FROM accounts WHERE id = 2;  -- reads $600
  total = balance1 + balance2;  -- $1100 (wrong!)
```

This is called **nonrepeatable read** or **read skew**. The transaction saw different states at different times.

---

### 2. Snapshot Isolation (Repeatable Read)

**Core idea**: Each transaction sees a **consistent snapshot** of the database from the start of the transaction.

This prevents read skew: even if data changes during the transaction, you see the values as they were when the transaction began.

#### Multi-Version Concurrency Control (MVCC)

**How it works**: The database keeps multiple versions of each object.

**Implementation**:
- Each transaction gets a unique, increasing transaction ID
- Each write includes the transaction ID that created it
- When reading, use visibility rules to determine which version to see

**Example structure**:
```
accounts table:
row_id | account_id | balance | created_by_txn | deleted_by_txn
1      | 1          | 500     | 10             | 15
2      | 1          | 400     | 15             | NULL
3      | 2          | 500     | 10             | 20
4      | 2          | 600     | 20             | NULL
```

**Visibility rules**: A transaction with ID `T` can see a version if:
1. Created by a transaction that committed before `T` started
2. Not deleted, or deleted by a transaction that hadn't committed when `T` started

**Benefits**:
- Readers never block writers
- Writers never block readers
- Only writers block other writers (when modifying the same object)

#### Preventing Lost Updates

Even with snapshot isolation, you can lose updates:

**Problem**:
```
-- Current value: counter = 10

Transaction 1:
  x = SELECT counter FROM page WHERE id = 5;  -- reads 10
  UPDATE page SET counter = x + 1 WHERE id = 5;  -- sets to 11

Transaction 2 (concurrent):
  x = SELECT counter FROM page WHERE id = 5;  -- reads 10
  UPDATE page SET counter = x + 1 WHERE id = 5;  -- sets to 11

-- Final value: 11 (should be 12!)
```

**Solutions**:

1. **Atomic operations**: `UPDATE page SET counter = counter + 1`
2. **Explicit locking**: `SELECT ... FOR UPDATE` (locks the row)
3. **Automatic detection**: Database detects lost update and aborts one transaction
4. **Compare-and-set**: `UPDATE ... WHERE counter = 10` (fails if value changed)

#### Write Skew

A more subtle problem that snapshot isolation **doesn't prevent**:

**Example**: Hospital on-call doctors
```
-- Rule: At least one doctor must be on call
-- Currently: Alice and Bob are both on call

Transaction 1 (Alice going off-call):
  oncall = SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- 2
  IF oncall >= 2:
    UPDATE doctors SET on_call = false WHERE name = 'Alice';

Transaction 2 (Bob going off-call, concurrent):
  oncall = SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- 2
  IF oncall >= 2:
    UPDATE doctors SET on_call = false WHERE name = 'Bob';

-- Result: Both go off-call! Constraint violated.
```

**Why it happens**: Both transactions read the same consistent snapshot, make decisions based on that snapshot, then write non-overlapping data (different rows). Snapshot isolation doesn't detect this conflict.

**Characteristics of write skew**:
1. Transaction reads some data
2. Makes a decision based on that read
3. Writes something (different from what was read)
4. By the time the write is committed, the premise of the decision is no longer true

**Solutions**:
- Serializable isolation (discussed next)
- Explicit locking of affected rows
- Materialized conflicts (create rows explicitly to lock)

---

## Serializability

**Definition**: The strongest isolation level. Guarantees that the result is the same as if transactions executed one at a time, in **some** serial order, even if they actually executed concurrently.

### Three Main Implementations:

---

### 1. Actual Serial Execution

**Idea**: Literally execute transactions one at a time, on a single thread.

**Why it works now** (didn't in the past):
- RAM is cheaper → entire dataset can fit in memory
- OLTP transactions are typically short and quick

**Used by**: VoltDB, Redis, Datomic

**Requirements for good performance**:
- Each transaction must be **very fast** (no slow network calls, disk I/O)
- Dataset must fit in memory
- Write throughput must be low enough for single thread
- Cross-partition transactions should be rare

**Example**:
```python
# Instead of interactive transactions:
balance = db.get_account_balance(account_id)  # Query 1
new_balance = balance - amount
db.set_account_balance(account_id, new_balance)  # Query 2

# Use stored procedures:
def transfer_money(from_account, to_account, amount):
    # Executed atomically on server
    from_balance = get_balance(from_account)
    to_balance = get_balance(to_account)
    set_balance(from_account, from_balance - amount)
    set_balance(to_account, to_balance + amount)
```

**Pros**:
- Simple to understand and implement
- No concurrency control overhead
- Predictable latency

**Cons**:
- Limited throughput (single thread)
- All transactions must be fast
- Doesn't scale to very large datasets

---

### 2. Two-Phase Locking (2PL)

**Core idea**: If one transaction has read an object and another wants to write it (or vice versa), they must wait.

**Not to be confused with** two-phase commit (2PC) - completely different!

#### How 2PL Works

**Lock types**:
- **Shared lock (read lock)**: Multiple transactions can hold simultaneously
- **Exclusive lock (write lock)**: Only one transaction can hold it

**Rules**:
1. To read an object, must acquire shared lock
2. To write an object, must acquire exclusive lock
3. If transaction A holds any lock and transaction B requests conflicting lock, B must wait
4. Locks held until transaction commits or aborts

**Example**:
```
Transaction 1:
  SELECT * FROM accounts WHERE id = 1;  -- acquires shared lock
  -- Holds lock...

Transaction 2 (concurrent):
  UPDATE accounts SET balance = 500 WHERE id = 1;  -- wants exclusive lock
  -- BLOCKED! Must wait for Transaction 1 to finish
```

#### Predicate Locks

**Problem**: Preventing phantoms (new rows appearing that match a query)

**Example**:
```
-- Room booking system, prevent double booking

Transaction 1 (check room 101 at 2pm):
  SELECT * FROM bookings 
  WHERE room = 101 AND time = '2pm' AND date = '2024-01-15';
  -- No results
  INSERT INTO bookings VALUES (101, '2pm', '2024-01-15', 'Alice');

Transaction 2 (concurrent):
  SELECT * FROM bookings 
  WHERE room = 101 AND time = '2pm' AND date = '2024-01-15';
  -- No results
  INSERT INTO bookings VALUES (101, '2pm', '2024-01-15', 'Bob');

-- Result: Double booking!
```

**Solution**: **Predicate lock** locks all objects matching some search condition, including objects that don't yet exist.

```
Transaction 1:
  -- Acquires lock on: room=101 AND time='2pm' AND date='2024-01-15'
  SELECT * FROM bookings WHERE ...;
  INSERT INTO bookings VALUES (...);

Transaction 2:
  -- Blocked! The predicate conflicts with Transaction 1's lock
```

#### Index-Range Locks

Predicate locks have poor performance (checking all locks against all predicates is expensive).

**Practical solution**: Index-range locking (or next-key locking)
- Lock a range of values in an index
- More coarse-grained than predicate locks
- Much better performance

**Example**: Instead of locking exactly "room=101 AND time='2pm'", lock all bookings for room 101 between 1pm and 3pm.

#### Performance of 2PL

**Major disadvantage**: **Significantly reduced concurrency**

- Readers block writers
- Writers block readers
- If transaction needs to wait for lock, unpredictable latencies (queue behind slow transaction)
- Deadlocks are possible (two transactions waiting for each other)

**Deadlock detection**: Database automatically detects cycles in wait-for graph and aborts one transaction.

---

### 3. Serializable Snapshot Isolation (SSI)

**The newest approach** - combines benefits of snapshot isolation with serializability.

**Key insight**: Use an **optimistic** approach
- Let transactions proceed without blocking
- At commit time, check if anything bad happened
- If so, abort and retry

**Used by**: PostgreSQL (since 9.1), FoundationDB, CockroachDB

#### How SSI Works

SSI builds on snapshot isolation but adds checks to detect when a transaction might have acted on **outdated premises**.

**Two types of conflicts to detect**:

1. **Detecting stale reads** (reading outdated values)
2. **Detecting writes that affect prior reads**

#### Detecting Stale Reads

**Track**: When a transaction reads data, and that data is modified by another concurrent transaction

**Example**:
```
Transaction 1 (snapshot isolation):
  oncall = SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- reads 2
  IF oncall >= 2:
    UPDATE doctors SET on_call = false WHERE name = 'Alice';

Transaction 2 (concurrent):
  oncall = SELECT COUNT(*) FROM doctors WHERE on_call = true;  -- reads 2
  IF oncall >= 2:
    UPDATE doctors SET on_call = false WHERE name = 'Bob';
```

**SSI tracking**:
- Transaction 1 reads the "on-call" status
- Transaction 2 modifies the "on-call" status (writes Bob off-call)
- When Transaction 1 tries to commit, SSI detects that its read is now stale
- **Abort Transaction 1**

#### Detecting Writes Affecting Prior Reads

**Track**: When a transaction's write might affect what another transaction read

Uses **index-range locks** (similar to 2PL) but doesn't block - just records that transactions conflict.

**At commit time**:
- Check if any transaction that read data you wrote has committed
- If yes → possible conflict → abort

#### Performance Characteristics

**Advantages over 2PL**:
- No blocking → better latency at percentiles
- Reads don't block writes, writes don't block reads
- Only conflicting transactions are affected

**Advantages over serial execution**:
- Can use multiple CPU cores
- Can partition data across machines
- Not limited by single-threaded throughput

**Disadvantage**:
- Abort rate can be significant if many conflicts
- Requires transaction retry logic in application
- Performance degrades under high contention

#### When Transactions Are Aborted

With SSI, transactions can be aborted even after they've done significant work.

**Application must**:
- Detect aborted transactions
- Retry automatically
- Ensure retry is safe (idempotent operations, or track which operations completed)

---

## Summary: Choosing an Isolation Level

| Isolation Level | Prevents | Performance | Complexity |
|----------------|----------|-------------|------------|
| **Read Uncommitted** | Almost nothing | Highest | Lowest |
| **Read Committed** | Dirty reads/writes | High | Low |
| **Snapshot Isolation** | Read skew, lost updates | Medium-High | Medium |
| **Serializable** | All race conditions | Varies | High |

### Serializability Implementation Comparison

| Approach | Best For | Limitations |
|----------|---------|-------------|
| **Serial Execution** | Small, fast transactions; in-memory datasets | Single-threaded throughput |
| **2PL** | Complex transactions with strong guarantees | Significant performance penalty, deadlocks |
| **SSI** | Most workloads | Abort rate under high contention |

---

## Key Takeaways

1. **Transactions simplify error handling** by providing all-or-nothing semantics

2. **Weak isolation levels have subtle bugs** that can cause rare data corruption (lost updates, write skew)

3. **Serializable isolation is the safest** but comes with performance trade-offs

4. **Three ways to achieve serializability**:
   - Actually execute serially (works if transactions are fast)
   - Use pessimistic locking (2PL - works but slow)
   - Use optimistic concurrency control (SSI - best balance for many workloads)

5. **Not all applications need serializability** - understand your consistency requirements and choose appropriately

6. **Race conditions are subtle** - even experienced developers miss them. Use strong isolation when in doubt.

---

## Practical Advice

**For most applications**: Start with snapshot isolation (or serializable if your database supports SSI with good performance).

**Use serial execution when**:
- Dataset fits in memory
- Transactions are very fast
- Write throughput requirements are modest

**Use 2PL when**:
- You need serializability
- Your database doesn't support SSI
- You can tolerate reduced performance

**Use SSI when**:
- Available in your database (PostgreSQL, CockroachDB)
- You need serializability
- You want better performance than 2PL
- You can handle occasional transaction retries

**Consider weaker isolation when**:
- Performance is critical
- Your application logic can handle the anomalies
- You're willing to carefully reason about race conditions
