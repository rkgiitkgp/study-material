## VII. Advanced Topics (System Design & Scaling)

---

### A. Distributed Databases

**1. Leader-follower replication vs multi-leader.**

**Leader-Follower Replication (Primary-Replica):**

A single leader (primary) accepts writes, and multiple followers (replicas) replicate the data asynchronously or synchronously.

**How it works:**
- Leader accepts all write operations
- Changes are propagated to followers via replication log
- Followers can serve read queries
- Automatic or manual failover promotes follower to leader if leader fails

**Advantages:**
- Simple to understand and implement
- Strong consistency on leader
- Read scaling (distribute reads across followers)
- No write conflicts

**Disadvantages:**
- Single point of failure for writes
- Write scalability limited to one node
- Replication lag can cause stale reads
- Failover requires coordination

**Example Architecture:**
```
                     ┌─────────────┐
                     │   Leader    │
                     │  (Writes)   │
                     └──────┬──────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
              ▼             ▼             ▼
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │Follower1│   │Follower2│   │Follower3│
        │ (Reads) │   │ (Reads) │   │ (Reads) │
        └─────────┘   └─────────┘   └─────────┘
```

**PostgreSQL Leader-Follower Setup:**
```sql
-- Leader configuration (postgresql.conf)
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

-- Follower configuration
hot_standby = on

-- Check replication status on leader
SELECT 
    client_addr,
    state,
    sync_state,
    replay_lag
FROM pg_stat_replication;
```

---

**Multi-Leader Replication (Multi-Master):**

Multiple leaders accept writes concurrently, and changes are replicated between all leaders.

**How it works:**
- Each leader accepts writes independently
- Changes propagate asynchronously to other leaders
- Conflict resolution required when same data modified
- Each leader also acts as follower for other leaders

**Advantages:**
- Write scalability across regions
- Better performance (write to nearest leader)
- Higher availability (multiple write nodes)
- Continues working during network partition

**Disadvantages:**
- Complex conflict resolution
- Eventual consistency issues
- Data conflicts are inevitable
- More complex to maintain

**Example Architecture:**
```
    US Region               EU Region               Asia Region
┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  Leader US     │◄──►│  Leader EU     │◄──►│  Leader Asia   │
│  (Read/Write)  │    │  (Read/Write)  │    │  (Read/Write)  │
└────────┬───────┘    └────────┬───────┘    └────────┬───────┘
         │                     │                     │
    ┌────┴────┐           ┌────┴────┐           ┌────┴────┐
    ▼         ▼           ▼         ▼           ▼         ▼
Follower  Follower    Follower  Follower    Follower  Follower
```

**Conflict Resolution Strategies:**

**1. Last Write Wins (LWW)**
```sql
-- Each write has timestamp
UPDATE users 
SET name = 'John', updated_at = '2024-11-01 10:30:00' 
WHERE id = 1;

-- Conflict resolution: Keep most recent timestamp
-- Simple but can lose data
```

**2. Version Vectors**
```sql
-- Track version per node
{
    "user_id": 1,
    "name": "John",
    "version": {
        "node_us": 5,
        "node_eu": 3,
        "node_asia": 2
    }
}
```

**3. Application-Level Resolution**
```python
def resolve_conflict(version_a, version_b):
    # Custom business logic
    if version_a['priority'] > version_b['priority']:
        return version_a
    return version_b
```

**4. CRDT (Conflict-Free Replicated Data Types)**
```sql
-- Counter that can be merged without conflicts
-- Node 1: counter = 10 + 5 = 15
-- Node 2: counter = 10 + 3 = 13
-- Merged: 10 + 5 + 3 = 18 (no conflict!)
```

---

**Comparison Table:**

| Aspect | Leader-Follower | Multi-Leader |
|--------|----------------|--------------|
| Write Operations | Single leader only | Multiple leaders |
| Write Conflicts | No conflicts | Frequent conflicts |
| Write Performance | Limited by one node | Scales across leaders |
| Read Performance | Scales with followers | Scales with all nodes |
| Complexity | Low | High |
| Consistency | Strong on leader | Eventual consistency |
| Failover | Manual/automatic | Built-in redundancy |
| Network Partition | Writes unavailable | Continues working |
| Use Case | Single region, strong consistency | Multi-region, high availability |
| Examples | PostgreSQL, MySQL | Cassandra, CouchDB |

---

**When to Use Each:**

**Use Leader-Follower When:**
- Single region deployment
- Need strong consistency
- Simple architecture preferred
- Read-heavy workload
- Write conflicts unacceptable

**Example:** Banking system (consistency critical)

**Use Multi-Leader When:**
- Multi-region deployment
- High write throughput needed
- Can tolerate eventual consistency
- Need geo-distributed writes
- Application can handle conflicts

**Example:** Collaborative editing (Google Docs), Social media feeds

---

**Real-World Example: E-commerce Platform**

**Leader-Follower Approach:**
```sql
-- US Leader (writes)
INSERT INTO orders (user_id, product_id, amount)
VALUES (123, 456, 99.99);

-- EU Follower (reads only, might have replication lag)
SELECT * FROM orders WHERE user_id = 123;
-- Might not see order immediately (eventual consistency)
```

**Multi-Leader Approach:**
```sql
-- US Leader
INSERT INTO orders (id, user_id, product_id, amount, region)
VALUES (uuid_v4(), 123, 456, 99.99, 'US');

-- EU Leader (concurrent write, same user)
INSERT INTO orders (id, user_id, product_id, amount, region)
VALUES (uuid_v4(), 123, 789, 49.99, 'EU');

-- Both succeed, no blocking
-- Conflict resolution not needed (different order IDs)
```

---

**Hybrid Approach: Leader-Follower per Region + Multi-Leader Across Regions**

```
    US Region                          EU Region
┌──────────────────┐            ┌──────────────────┐
│   Leader US      │◄──────────►│   Leader EU      │
│   (Writes)       │            │   (Writes)       │
└────────┬─────────┘            └────────┬─────────┘
         │                               │
    ┌────┴────┐                     ┌────┴────┐
    ▼         ▼                     ▼         ▼
Follower  Follower              Follower  Follower
(Reads)   (Reads)               (Reads)   (Reads)

- Within region: Leader-Follower (strong consistency)
- Across regions: Multi-Leader (high availability)
```

**Best Practices:**

**Leader-Follower:**
1. ✅ Monitor replication lag
2. ✅ Use synchronous replication for critical data
3. ✅ Test failover procedures regularly
4. ✅ Route reads to followers
5. ✅ Handle replication lag in application

**Multi-Leader:**
1. ✅ Implement robust conflict resolution
2. ✅ Use unique identifiers (UUIDs, not auto-increment)
3. ✅ Design schema to minimize conflicts
4. ✅ Monitor conflict frequency
5. ✅ Test cross-region failover

**Monitoring:**
```sql
-- PostgreSQL: Check replication lag
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
    replay_lag
FROM pg_stat_replication;

-- Alert if lag > 10 seconds
```

Answer:


**2. Consensus algorithms (Raft, Paxos).**

**Consensus Algorithms:**

Consensus algorithms allow multiple nodes in a distributed system to agree on a single value/state, even in the presence of failures. They ensure that all nodes see the same data and maintain consistency.

**Why Consensus is Needed:**
- Leader election in distributed databases
- Ensuring all replicas have the same data
- Coordinating distributed transactions
- Managing cluster membership
- Maintaining consistent state across nodes

---

**RAFT Consensus Algorithm:**

Raft is a consensus algorithm designed to be easier to understand than Paxos while providing the same guarantees.

**Key Concepts:**

**1. Node States:**
- **Leader:** Handles all client requests, replicates log entries
- **Follower:** Passive, receives log entries from leader
- **Candidate:** Temporary state during leader election

**2. Terms:**
- Logical time divided into terms
- Each term has at most one leader
- Term number increases monotonically

**3. Log Replication:**
- Leader receives client request
- Leader appends entry to its log
- Leader replicates entry to followers
- Once majority acknowledges, entry is committed
- Leader applies entry and responds to client

**Raft Leader Election Process:**

```
Initial State: All nodes are Followers
      ↓
Follower times out (no heartbeat from leader)
      ↓
Follower becomes Candidate
      ↓
Candidate votes for itself, increments term
      ↓
Candidate requests votes from other nodes
      ↓
┌─────────────────────────────────────────┐
│                                         │
│  Receives majority votes?               │
│                                         │
│  YES → Becomes Leader                   │
│        Sends heartbeats to followers    │
│                                         │
│  NO  → Receives higher term?            │
│        → Becomes Follower               │
│                                         │
│  Timeout → Start new election           │
│           (increment term, try again)   │
│                                         │
└─────────────────────────────────────────┘
```

**Raft Example Timeline:**

```
Time  Node 1       Node 2       Node 3       Node 4       Node 5
----  ----------   ----------   ----------   ----------   ----------
T1    Follower     Follower     Follower     Follower     Follower
T2    Leader       Follower     Follower     Follower     Follower
      (elected)
T3    Leader ────► Follower     Follower     Follower     Follower
      (heartbeat)
T4    Leader       Follower     Follower     Follower     Follower
      (write req)  (replicate)  (replicate)  (replicate)  (DOWN)
T5    Leader       Follower     Follower     Follower     
      (committed)  (ACK)        (ACK)        (ACK)
      3/5 quorum achieved ✓
```

**Raft Write Operation:**

```sql
-- Client sends write request to Leader
-- Leader: Term 5, Log Index 10

Step 1: Leader appends entry to own log
Log[10] = {term: 5, command: "SET x = 100"}

Step 2: Leader replicates to followers
Leader → Follower1: AppendEntries(term=5, index=10, entry="SET x = 100")
Leader → Follower2: AppendEntries(term=5, index=10, entry="SET x = 100")
Leader → Follower3: AppendEntries(term=5, index=10, entry="SET x = 100")
Leader → Follower4: AppendEntries(term=5, index=10, entry="SET x = 100")

Step 3: Followers append and acknowledge
Follower1 → Leader: ACK
Follower2 → Leader: ACK
Follower3 → Leader: ACK
Follower4 → Leader: (network failure, no ACK)

Step 4: Leader commits (3/5 quorum = majority)
commitIndex = 10

Step 5: Leader applies to state machine and responds
x = 100
Response to client: "Success"

Step 6: Leader notifies followers to commit
(Followers apply to their state machines)
```

**Raft Configuration Example (etcd - uses Raft):**

```bash
# etcd cluster using Raft
etcd --name node1 \
  --initial-advertise-peer-urls http://node1:2380 \
  --listen-peer-urls http://node1:2380 \
  --listen-client-urls http://node1:2379 \
  --advertise-client-urls http://node1:2379 \
  --initial-cluster node1=http://node1:2380,node2=http://node2:2380,node3=http://node3:2380 \
  --initial-cluster-state new

# Check cluster status
etcdctl member list
etcdctl endpoint status --write-out=table

# Example output:
# +--------+---------+---------+--------+----------+-------------+
# | NODE   | STATUS  | LEADER  | TERM   | RAFTINDEX| RAFTAPPLIED |
# +--------+---------+---------+--------+----------+-------------+
# | node1  | Leader  | true    | 5      | 1247     | 1247        |
# | node2  | Follower| false   | 5      | 1247     | 1247        |
# | node3  | Follower| false   | 5      | 1247     | 1247        |
# +--------+---------+---------+--------+----------+-------------+
```

---

**PAXOS Consensus Algorithm:**

Paxos is the original consensus algorithm, known for being difficult to understand and implement.

**Key Roles:**
- **Proposer:** Proposes values
- **Acceptor:** Votes on proposals
- **Learner:** Learns the chosen value

(Note: A node can play multiple roles)

**Paxos Phases:**

**Phase 1: Prepare (Leader Election)**
```
Proposer → Acceptors: PREPARE(n)
  where n = proposal number

Acceptors respond:
  - PROMISE(n) if n > any previous proposal
  - Include any already accepted value
  - Reject if n ≤ previous proposal
```

**Phase 2: Accept (Value Agreement)**
```
Proposer → Acceptors: ACCEPT(n, value)
  - Uses value from Phase 1 if any
  - Otherwise uses own value

Acceptors respond:
  - ACCEPTED(n, value) if n ≥ promised
  - Reject otherwise

If majority accepts → value is chosen
```

**Paxos Example:**

```
Initial State: 3 Acceptors, 2 Proposers

Time  Proposer A            Proposer B            Acceptor 1      Acceptor 2      Acceptor 3
----  ----------            ----------            ----------      ----------      ----------
T1    PREPARE(1) ────────► (prepare msg)         PROMISE(1)      PROMISE(1)      PROMISE(1)
                                                  
T2    Receives majority promises ✓
      ACCEPT(1, "X") ──────►                     ACCEPTED(1,X)   ACCEPTED(1,X)   ACCEPTED(1,X)

T3    Value "X" is chosen! ✓
      
T4                          PREPARE(2) ───────►                                  PROMISE(2)
      (Too late, "X" already chosen)                                            (but X accepted)

T5                          Receives promise with accepted value "X"
                            Must propose "X"
                            ACCEPT(2, "X") ────►  ACCEPTED(2,X)   ACCEPTED(2,X)   ACCEPTED(2,X)
```

**Paxos Write Operation Example:**

```
Scenario: Write value "balance=500" to distributed system

Phase 1: Prepare
-----------------
Proposer sends: PREPARE(proposal_number=10)

Acceptor 1: "I promise not to accept proposals < 10"
            Returns: PROMISE(10, no_previous_value)

Acceptor 2: "I promise not to accept proposals < 10"
            Returns: PROMISE(10, no_previous_value)

Acceptor 3: (network partition, no response)

Quorum (2/3) achieved ✓

Phase 2: Accept
-----------------
Proposer sends: ACCEPT(10, "balance=500")

Acceptor 1: "Proposal 10 ≥ my promise, I accept"
            Returns: ACCEPTED(10, "balance=500")

Acceptor 2: "Proposal 10 ≥ my promise, I accept"
            Returns: ACCEPTED(10, "balance=500")

Acceptor 3: (still partitioned)

Quorum (2/3) achieved ✓

Result: "balance=500" is committed!
```

---

**Raft vs Paxos Comparison:**

| Aspect | Raft | Paxos |
|--------|------|-------|
| **Understandability** | Easier to understand | Very complex |
| **Implementation** | Simpler to implement | Complex to implement |
| **Leader Election** | Explicit, clear process | Implicit in protocol |
| **Log Structure** | Strong leader, sequential log | More flexible |
| **Membership Changes** | Built-in joint consensus | Requires extension |
| **Performance** | Comparable | Comparable |
| **Real-World Usage** | etcd, Consul, CockroachDB | Google Chubby, Spanner |
| **Documentation** | Well-documented | Academic papers |
| **Industry Adoption** | High (modern systems) | Lower (legacy systems) |

**Visual Comparison:**

```
RAFT: Clear Leadership Model
┌──────────┐
│  Leader  │ ◄── All writes go here
└────┬─────┘
     │ replicates to
     ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│Follower │  │Follower │  │Follower │
└─────────┘  └─────────┘  └─────────┘

PAXOS: More Democratic
┌─────────┐  ┌─────────┐  ┌─────────┐
│Acceptor │  │Acceptor │  │Acceptor │
└────▲────┘  └────▲────┘  └────▲────┘
     │            │            │
     └────────────┼────────────┘
           ┌──────┴──────┐
           │  Proposers  │ ◄── Multiple can propose
           └─────────────┘
```

---

**Real-World Applications:**

**Systems Using Raft:**
- **etcd:** Kubernetes' configuration store
- **Consul:** Service discovery and configuration
- **CockroachDB:** Distributed SQL database
- **TiDB:** MySQL-compatible distributed database

**Systems Using Paxos:**
- **Google Spanner:** Global distributed database
- **Google Chubby:** Distributed lock service
- **Apache Cassandra:** Lightweight transactions (Paxos variant)
- **Amazon DynamoDB:** (Modified Paxos)

---

**Practical Example: Database Failover with Raft**

```python
# Simplified Raft implementation concept

class RaftNode:
    def __init__(self, node_id, peers):
        self.node_id = node_id
        self.state = "FOLLOWER"  # FOLLOWER, CANDIDATE, LEADER
        self.current_term = 0
        self.voted_for = None
        self.log = []
        self.commit_index = 0
        self.peers = peers
        
    def start_election(self):
        """Node becomes candidate and requests votes"""
        self.state = "CANDIDATE"
        self.current_term += 1
        self.voted_for = self.node_id
        
        votes = 1  # Vote for self
        
        for peer in self.peers:
            response = peer.request_vote(self.current_term, self.node_id)
            if response.vote_granted:
                votes += 1
        
        if votes > len(self.peers) / 2:
            self.become_leader()
    
    def become_leader(self):
        """Node becomes leader after winning election"""
        self.state = "LEADER"
        print(f"Node {self.node_id} became leader for term {self.current_term}")
        self.send_heartbeats()
    
    def replicate_log(self, data):
        """Leader replicates data to followers"""
        entry = LogEntry(self.current_term, len(self.log), data)
        self.log.append(entry)
        
        acks = 1  # Leader acknowledges itself
        
        for peer in self.peers:
            response = peer.append_entries(entry)
            if response.success:
                acks += 1
        
        if acks > len(self.peers) / 2:
            # Quorum reached, commit entry
            self.commit_index = len(self.log) - 1
            self.apply_to_state_machine(entry)
            return True
        
        return False
```

---

**Handling Split-Brain Scenario:**

```
Scenario: Network partition splits cluster

Initial: 5-node cluster
┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐
│ L  │ │ F  │ │ F  │ │ F  │ │ F  │
└────┘ └────┘ └────┘ └────┘ └────┘
Leader  Follower  (Network Partition)

After partition:
Partition A (2 nodes)     Partition B (3 nodes)
┌────┐ ┌────┐            ┌────┐ ┌────┐ ┌────┐
│ L  │ │ F  │            │ F  │ │ F  │ │ F  │
└────┘ └────┘            └────┘ └────┘ └────┘

Partition A:
- Leader can't reach quorum (needs 3/5)
- Becomes read-only or steps down

Partition B:
- Followers elect new leader (3/5 quorum ✓)
- New leader serves writes

When partition heals:
- Old leader sees higher term number
- Old leader becomes follower
- Cluster reunites under new leader
```

---

**Key Guarantees:**

**Both Raft and Paxos Guarantee:**
1. **Safety:** Never return incorrect results
2. **Availability:** Cluster available if majority nodes alive
3. **Consistency:** All nodes agree on committed values
4. **Ordering:** Operations applied in same order on all nodes

**Failure Tolerance:**
- 3-node cluster: Tolerates 1 failure
- 5-node cluster: Tolerates 2 failures
- 7-node cluster: Tolerates 3 failures
- Formula: Tolerates (N-1)/2 failures

---

**Best Practices:**

1. ✅ Use odd number of nodes (3, 5, 7)
2. ✅ Don't go above 7 nodes (diminishing returns)
3. ✅ Monitor cluster health and term changes
4. ✅ Ensure reliable network between nodes
5. ✅ Set appropriate timeouts (election timeout > heartbeat interval)
6. ✅ Test failover scenarios regularly
7. ❌ Don't use even numbers (risk of split votes)
8. ❌ Don't spread nodes across high-latency networks

**Monitoring:**

```bash
# etcd (Raft) monitoring
etcdctl endpoint health
etcdctl endpoint status

# Check leader
etcdctl endpoint status --write-out=table

# Watch for leader changes
etcdctl watch "" --prefix --print-leader

# Metrics to monitor:
# - Leader election frequency (should be rare)
# - Proposal commit latency
# - Number of failed proposals
# - Network latency between nodes
```

Answer:


**3. Eventual consistency vs strong consistency.**

**Consistency Models:**

Consistency models define the guarantees about when and how changes become visible to readers in a distributed system.

---

**Strong Consistency (Immediate Consistency):**

Every read receives the most recent write or an error. All nodes see the same data at the same time.

**Guarantees:**
- Read always returns latest written value
- All replicas synchronized before acknowledging write
- Linearizability - operations appear to occur instantaneously
- No stale data

**How it Works:**
```
Client Write Request
        ↓
    Primary Node
        ↓
Replicate to ALL replicas SYNCHRONOUSLY
        ↓
Wait for ALL confirmations
        ↓
Acknowledge to client
        ↓
All subsequent reads see new value
```

**Example:**
```sql
-- Time T1: User writes
UPDATE accounts SET balance = 5000 WHERE user_id = 123;
-- Response: OK (only after ALL replicas updated)

-- Time T2: User reads (from any replica)
SELECT balance FROM accounts WHERE user_id = 123;
-- Result: 5000 (guaranteed latest value)
```

**Real-World Analogy:**
Like a classroom where the teacher writes on the board. Everyone sees the same content immediately, or nobody can read until the teacher finishes writing.

**Implementation - Synchronous Replication:**
```sql
-- PostgreSQL: Synchronous replication
-- postgresql.conf
synchronous_standby_names = 'standby1,standby2'
synchronous_commit = on

-- Write is confirmed only after standby ACKs
BEGIN;
UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;  -- Waits for standby acknowledgment before returning
```

**Advantages:**
- ✅ Data always consistent
- ✅ No stale reads
- ✅ Simple to reason about
- ✅ Perfect for financial transactions

**Disadvantages:**
- ❌ Higher latency (wait for all replicas)
- ❌ Lower availability (fails if replica down)
- ❌ Poor performance across regions
- ❌ Not partition-tolerant

**Use Cases:**
- Banking and financial systems
- Inventory management
- Booking systems (seats, hotels)
- Any system where stale data is unacceptable

---

**Eventual Consistency:**

Updates propagate asynchronously. Eventually all replicas will have the same data, but reads may return stale data temporarily.

**Guarantees:**
- If no new updates, all replicas eventually converge
- Reads may return old values (bounded staleness)
- Higher availability and performance
- Tolerates network partitions

**How it Works:**
```
Client Write Request
        ↓
    Primary Node
        ↓
Acknowledge to client IMMEDIATELY
        ↓
Replicate to replicas ASYNCHRONOUSLY (background)
        ↓
Eventually all replicas updated
        ↓
Reads may see old value during propagation
```

**Example:**
```sql
-- Time T1: User writes (US region)
UPDATE posts SET likes = likes + 1 WHERE post_id = 456;
-- Response: OK (immediate, doesn't wait for replicas)

-- Time T2 (1ms later): User reads (EU region)
SELECT likes FROM posts WHERE post_id = 456;
-- Result: Old value (replication in progress)

-- Time T3 (100ms later): User reads again
SELECT likes FROM posts WHERE post_id = 456;
-- Result: New value (replication completed)
```

**Real-World Analogy:**
Like social media likes/comments. You post a comment, it appears immediately for you, but takes a moment to appear for users in other regions. Eventually everyone sees it.

**Implementation - Asynchronous Replication:**
```sql
-- PostgreSQL: Asynchronous replication
-- postgresql.conf
synchronous_commit = off  -- Don't wait for standby

-- Write returns immediately
BEGIN;
UPDATE posts SET content = 'Updated' WHERE id = 1;
COMMIT;  -- Returns immediately, replication happens in background
```

**Advantages:**
- ✅ Low latency (fast writes)
- ✅ High availability (works during partition)
- ✅ Good for geo-distributed systems
- ✅ Better scalability

**Disadvantages:**
- ❌ Stale reads possible
- ❌ Complex to reason about
- ❌ Conflict resolution needed
- ❌ May lose recent writes on failure

**Use Cases:**
- Social media (likes, comments, followers)
- Content delivery (blogs, videos)
- Analytics and reporting
- Shopping cart
- DNS systems

---

**Comparison Table:**

| Aspect | Strong Consistency | Eventual Consistency |
|--------|-------------------|---------------------|
| **Read Guarantee** | Always latest value | May return stale data |
| **Write Latency** | High (wait for replicas) | Low (immediate ACK) |
| **Read Latency** | Medium | Low |
| **Availability** | Lower (replica must be up) | Higher (works during partition) |
| **CAP Choice** | CP (Consistency + Partition tolerance) | AP (Availability + Partition tolerance) |
| **Complexity** | Simple | Complex (conflict resolution) |
| **Performance** | Slower | Faster |
| **Data Loss Risk** | Lower | Higher (recent writes) |
| **Scalability** | Limited | Better |
| **Use Case** | Banking, inventory | Social media, caching |
| **Examples** | Google Spanner, CockroachDB | DynamoDB, Cassandra, Riak |

---

**CAP Theorem Context:**

```
CAP Theorem: You can have at most 2 of 3

         Consistency
              ▲
             / \
            /   \
           /     \
          /   🤔  \
         /         \
        /___________\
  Availability    Partition
                Tolerance

Strong Consistency = CP
- Sacrifice Availability during partition
- Examples: MongoDB, HBase

Eventual Consistency = AP  
- Sacrifice Consistency during partition
- Examples: Cassandra, DynamoDB, Riak
```

---

**Practical Example: E-commerce Inventory**

**Strong Consistency Approach (Prevent Overselling):**
```sql
-- Transaction must wait for all replicas
BEGIN;

-- Lock the row
SELECT stock FROM products WHERE id = 100 FOR UPDATE;
-- stock = 5

-- Check if available
IF stock > 0 THEN
    UPDATE products SET stock = stock - 1 WHERE id = 100;
    INSERT INTO orders (product_id, user_id) VALUES (100, 123);
    COMMIT;  -- Waits for all replicas to confirm
ELSE
    ROLLBACK;
END IF;

-- Guarantee: No overselling, but slower checkout
```

**Eventual Consistency Approach (Fast but Risk Overselling):**
```sql
-- Write returns immediately
BEGIN;

SELECT stock FROM products WHERE id = 100;
-- stock = 5 (might be stale!)

UPDATE products SET stock = stock - 1 WHERE id = 100;
INSERT INTO orders (product_id, user_id) VALUES (100, 123);
COMMIT;  -- Returns immediately

-- Risk: Multiple regions might oversell
-- Solution: Allow backorders, compensate later
```

---

**Hybrid Approaches:**

**1. Read-Your-Own-Writes Consistency:**
```python
# User sees their own updates immediately
def update_profile(user_id, data):
    # Write to primary
    primary_db.update(user_id, data)
    
    # Cache user's own write
    cache.set(f"user:{user_id}", data, ttl=60)
    
def get_profile(user_id, requesting_user_id):
    # If reading own profile, use cache/primary
    if user_id == requesting_user_id:
        return cache.get(f"user:{user_id}") or primary_db.get(user_id)
    
    # Others can read from eventually consistent replica
    return replica_db.get(user_id)
```

**2. Causal Consistency:**
```sql
-- Related operations maintain order
-- Example: Comment must appear after post

-- Post (causally before)
INSERT INTO posts (id, content, timestamp) VALUES (1, 'Hello', 100);

-- Comment (causally after)
INSERT INTO comments (post_id, content, timestamp) VALUES (1, 'Nice!', 101);

-- Guarantee: Comments never appear before their posts
```

**3. Session Consistency:**
```python
# Within same session, reads see writes
session_token = "abc123"

# Write
db.write(key="user:1", value="John", session=session_token)

# Read in same session sees write
db.read(key="user:1", session=session_token)  # Returns "John"

# Read in different session might see old value
db.read(key="user:1", session="xyz789")  # Might return old value
```

**4. Tunable Consistency (Cassandra, DynamoDB):**
```sql
-- Cassandra: Choose consistency per query

-- Strong consistency (W + R > N)
-- Write to 2 replicas, Read from 2 replicas (out of 3 total)
INSERT INTO users (id, name) VALUES (1, 'John') 
USING CONSISTENCY QUORUM;  -- Write to majority

SELECT * FROM users WHERE id = 1 
USING CONSISTENCY QUORUM;  -- Read from majority
-- Guaranteed to see latest write


-- Eventual consistency
INSERT INTO users (id, name) VALUES (1, 'John') 
USING CONSISTENCY ONE;  -- Write to 1 replica

SELECT * FROM users WHERE id = 1 
USING CONSISTENCY ONE;  -- Read from 1 replica
-- Might see stale data, but faster
```

---

**Timeline Visualization:**

**Strong Consistency:**
```
Time  Node 1 (Primary)    Node 2 (Replica)    Client
----  -----------------   ----------------    ------
T1    Write: x = 10       
T2    Replicate ─────────► Write: x = 10
T3    Wait for ACK ◄────── ACK
T4    ─────────────────────────────────────► Success
T5                        Read: x = 10 ✓     Read: x = 10 ✓

All reads return 10 immediately
```

**Eventual Consistency:**
```
Time  Node 1 (Primary)    Node 2 (Replica)    Client
----  -----------------   ----------------    ------
T1    Write: x = 10
T2    ─────────────────────────────────────► Success (immediate!)
T3    Replicate ────────► (in progress)
T4                                           Read from Node2: x = 5 (stale!)
T5                        Write: x = 10 ✓
T6                                           Read from Node2: x = 10 ✓

Reads may see old value until replication completes
```

---

**Eventual Consistency - Conflict Resolution:**

**Problem: Concurrent Writes to Same Data**

```
US Region                          EU Region
---------                          ---------
T1: UPDATE users                   T1: UPDATE users
    SET name = 'John'                  SET name = 'Johnny'
    WHERE id = 1                       WHERE id = 1

Conflict! Which value wins?
```

**Resolution Strategies:**

**1. Last Write Wins (LWW):**
```sql
-- Include timestamp with each write
UPDATE users 
SET name = 'John', updated_at = '2024-11-01 10:00:00' 
WHERE id = 1;

-- Conflict resolution: Keep most recent timestamp
-- Simple but can lose data
```

**2. Version Vectors:**
```json
{
  "user_id": 1,
  "name": "John",
  "version": {
    "us_region": 5,
    "eu_region": 3
  }
}
```

**3. Application-Level Merge:**
```python
def merge_profiles(version_a, version_b):
    return {
        "name": version_b["name"],  # Use latest name
        "email": version_a["email"], # Keep original email
        "followers": version_a["followers"] + version_b["followers"]  # Merge lists
    }
```

---

**Monitoring Replication Lag:**

```sql
-- PostgreSQL: Check replication lag
SELECT 
    application_name,
    client_addr,
    state,
    replay_lag,
    write_lag,
    flush_lag
FROM pg_stat_replication;

-- Alert if lag > threshold
-- Strong consistency: Lag should be ~0ms
-- Eventual consistency: Lag acceptable (e.g., <1 second)
```

```python
# Application monitoring
import time

def check_consistency():
    # Write with timestamp
    primary_db.write("test_key", {"value": "test", "timestamp": time.time()})
    
    # Read from replica
    time.sleep(0.1)  # Small delay
    replica_data = replica_db.read("test_key")
    
    lag = time.time() - replica_data["timestamp"]
    print(f"Replication lag: {lag*1000}ms")
    
    if lag > 1.0:  # Alert if lag > 1 second
        alert("High replication lag detected")
```

---

**When to Choose Each:**

**Choose Strong Consistency When:**
- ✅ Financial transactions
- ✅ Inventory management
- ✅ Booking systems
- ✅ User authentication
- ✅ Regulatory compliance required
- ❌ Can tolerate higher latency
- ❌ Single region deployment OK

**Choose Eventual Consistency When:**
- ✅ Social media interactions
- ✅ Content delivery
- ✅ Analytics/metrics
- ✅ Caching layers
- ✅ Global distribution needed
- ✅ High throughput required
- ❌ Stale reads acceptable
- ❌ Can handle conflicts

---

**Best Practices:**

**For Strong Consistency:**
1. ✅ Use synchronous replication
2. ✅ Set appropriate timeouts
3. ✅ Monitor replica health
4. ✅ Have failover procedures
5. ✅ Use transactions properly

**For Eventual Consistency:**
1. ✅ Design for idempotency
2. ✅ Implement conflict resolution
3. ✅ Show "pending" states to users
4. ✅ Use vector clocks or timestamps
5. ✅ Monitor replication lag
6. ✅ Set user expectations (UI feedback)
7. ✅ Design compensating transactions

**Mixed Approach:**
```python
class DataStore:
    def write(self, key, value, consistency_level="eventual"):
        if consistency_level == "strong":
            # Critical data: Wait for all replicas
            return self.sync_write(key, value)
        else:
            # Non-critical data: Async replication
            return self.async_write(key, value)
    
# Use strong consistency for critical operations
store.write("account_balance", 5000, consistency_level="strong")

# Use eventual consistency for non-critical operations  
store.write("profile_views_count", 123, consistency_level="eventual")
```

Answer:


**4. How does database handle network partition?**

**Network Partition (Split-Brain Problem):**

A network partition occurs when network failures split a cluster into isolated groups that cannot communicate with each other. Each partition may continue operating independently, potentially leading to data inconsistencies.

**The Core Challenge:**
```
Normal Operation:
┌─────────────────────────────────────┐
│   Node1 ←→ Node2 ←→ Node3 ←→ Node4  │
│         All nodes connected          │
└─────────────────────────────────────┘

After Network Partition:
┌──────────────────┐        ┌──────────────────┐
│  Node1 ←→ Node2  │   ✗    │  Node3 ←→ Node4  │
│   Partition A    │        │   Partition B    │
└──────────────────┘        └──────────────────┘

Problem: Both partitions think they're the cluster!
Risk: Conflicting writes, divergent state (split-brain)
```

**CAP Theorem and Partition Handling:**

According to CAP theorem, during a network partition, you must choose between:
- **Consistency (C):** Refuse writes, maintain data consistency
- **Availability (A):** Accept writes, risk inconsistency

**Quorum-Based Approach:**
- Requires majority (quorum) of nodes: Quorum = ⌊N/2⌋ + 1
- Partition without quorum becomes read-only or unavailable
- Example: 5-node cluster needs 3 nodes for quorum

**Example: MongoDB Handling Partition (CP System)**
```javascript
// 5-node replica set
// Partition: 2 nodes | 3 nodes

// Minority partition (2 nodes):
db.orders.insert({ item: "laptop" })
// Error: NotWritablePrimary - no quorum

// Majority partition (3 nodes):
db.orders.insert({ item: "laptop" })
// Success - has quorum
```

**Example: Cassandra Handling Partition (AP System)**
```sql
-- Both partitions continue accepting writes
-- Uses eventual consistency and conflict resolution
-- Last Write Wins based on timestamp

-- When partition heals:
nodetool repair  -- Syncs data across nodes
```

**Best Practices:**
1. ✅ Use odd number of nodes (3, 5, 7)
2. ✅ Monitor heartbeats and detect partitions quickly
3. ✅ Implement proper timeout and retry logic
4. ✅ Test partition scenarios regularly
5. ✅ Design for idempotent operations
6. ✅ Geographic distribution across availability zones

Answer:


---

### B. Architecture Trade-offs

**1. When to denormalize data intentionally.**

**Denormalization** is the intentional introduction of redundancy into a database design to improve read performance by reducing the need for expensive JOIN operations.

**When to Denormalize:**

**1. Read-Heavy Workloads**
- Reads significantly outnumber writes (e.g., 100:1 ratio)
- JOIN operations become bottleneck
- Query performance is critical

**Example:**
```sql
-- Normalized (requires JOIN)
SELECT 
    o.order_id,
    c.customer_name,
    c.customer_email,
    c.customer_phone
FROM orders o
JOIN customers c ON o.customer_id = c.id
WHERE o.order_date > '2024-01-01';
-- Slow: Must JOIN for every query

-- Denormalized (no JOIN needed)
CREATE TABLE orders_denormalized (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),      -- ← Redundant
    customer_email VARCHAR(100),     -- ← Redundant
    customer_phone VARCHAR(20),      -- ← Redundant
    order_date DATE,
    total_amount DECIMAL(10,2)
);

SELECT * FROM orders_denormalized 
WHERE order_date > '2024-01-01';
-- Fast: Single table scan, no JOINs
```

**2. Reporting and Analytics**
- Complex aggregations across multiple tables
- Historical data that rarely changes
- Dashboard queries that run frequently

**Example:**
```sql
-- Create denormalized summary table for dashboards
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    DATE_TRUNC('day', o.order_date) as sales_date,
    c.country,
    c.region,
    p.category,
    p.brand,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(o.total_amount) as total_revenue,
    AVG(o.total_amount) as avg_order_value,
    COUNT(DISTINCT c.customer_id) as unique_customers
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id
GROUP BY sales_date, c.country, c.region, p.category, p.brand;

-- Refresh periodically (e.g., daily)
REFRESH MATERIALIZED VIEW daily_sales_summary;

-- Lightning-fast dashboard queries
SELECT * FROM daily_sales_summary
WHERE sales_date BETWEEN '2024-01-01' AND '2024-12-31';
-- Instant results instead of scanning millions of rows
```

**3. Microservices Architecture**
- Service independence is priority
- Avoid cross-service database calls
- Each service has own data model

**Example:**
```python
# Order Service Database
class Order:
    order_id: int
    customer_id: int
    customer_name: str        # ← Denormalized from Customer Service
    customer_email: str       # ← Denormalized from Customer Service
    items: list
    total: decimal
    
# Benefit: Order service doesn't need to call Customer service
# Trade-off: Must sync when customer updates their info
```

**4. NoSQL Databases**
- Document databases (MongoDB) naturally denormalized
- Embedding related data in single document
- Optimized for document retrieval

**Example:**
```javascript
// MongoDB: Denormalized user document with embedded posts
{
  "_id": 123,
  "username": "john_doe",
  "email": "john@example.com",
  "profile": {
    "bio": "Software Engineer",
    "location": "San Francisco",
    "followers_count": 1250
  },
  "recent_posts": [  // ← Denormalized: Embed recent posts
    {
      "post_id": 456,
      "content": "Hello world!",
      "likes": 42,
      "created_at": "2024-11-01"
    },
    {
      "post_id": 457,
      "content": "Learning databases",
      "likes": 67,
      "created_at": "2024-11-02"
    }
  ],
  "stats": {  // ← Denormalized: Precomputed stats
    "total_posts": 342,
    "total_likes_received": 5420
  }
}

// Single query gets everything - no joins!
db.users.findOne({ _id: 123 })
```

**5. Caching Layers**
- Pre-compute expensive queries
- Store aggregated results
- Reduce database load

**Example:**
```sql
-- Cache table for expensive calculations
CREATE TABLE user_statistics_cache (
    user_id INT PRIMARY KEY,
    total_orders INT,
    total_spent DECIMAL(10,2),
    avg_order_value DECIMAL(10,2),
    last_order_date DATE,
    favorite_category VARCHAR(50),
    updated_at TIMESTAMP
);

-- Update cache periodically or on-demand
INSERT INTO user_statistics_cache
SELECT 
    customer_id as user_id,
    COUNT(*) as total_orders,
    SUM(total_amount) as total_spent,
    AVG(total_amount) as avg_order_value,
    MAX(order_date) as last_order_date,
    MODE() WITHIN GROUP (ORDER BY category) as favorite_category,
    NOW() as updated_at
FROM orders
GROUP BY customer_id
ON CONFLICT (user_id) DO UPDATE
SET total_orders = EXCLUDED.total_orders,
    total_spent = EXCLUDED.total_spent,
    -- ... update all fields
    updated_at = NOW();
```

**6. Time-Series Data**
- Append-only data
- Historical data never changes
- Queries span time ranges

**Example:**
```sql
-- Denormalized sensor readings with metadata
CREATE TABLE sensor_readings_denormalized (
    reading_id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    sensor_id INT,
    sensor_name VARCHAR(100),        -- ← Denormalized
    sensor_location VARCHAR(200),    -- ← Denormalized
    sensor_type VARCHAR(50),         -- ← Denormalized
    building_name VARCHAR(100),      -- ← Denormalized
    floor_number INT,                -- ← Denormalized
    temperature DECIMAL(5,2),
    humidity DECIMAL(5,2),
    INDEX idx_timestamp (timestamp)
);

-- Fast queries without joins
SELECT * FROM sensor_readings_denormalized
WHERE building_name = 'Building A'
  AND timestamp > NOW() - INTERVAL '24 hours';
```

**7. Performance-Critical Paths**
- Checkout process
- Search results
- Real-time feeds

**Example:**
```sql
-- E-commerce product search (denormalized for speed)
CREATE TABLE products_search (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(255),
    description TEXT,
    price DECIMAL(10,2),
    category_name VARCHAR(100),      -- ← From categories table
    brand_name VARCHAR(100),         -- ← From brands table
    avg_rating DECIMAL(3,2),         -- ← Computed from reviews
    review_count INT,                -- ← Computed from reviews
    in_stock BOOLEAN,                -- ← From inventory table
    image_url VARCHAR(500),
    search_vector TSVECTOR           -- ← Full-text search
);

-- Ultra-fast product search
SELECT * FROM products_search
WHERE search_vector @@ to_tsquery('laptop')
  AND price BETWEEN 500 AND 1500
  AND in_stock = true
ORDER BY avg_rating DESC
LIMIT 20;
-- No joins, all data in one table
```

---

**Trade-offs of Denormalization:**

| Pros | Cons |
|------|------|
| ✅ Faster reads (no JOINs) | ❌ Slower writes (update multiple places) |
| ✅ Simpler queries | ❌ More storage space |
| ✅ Better for OLAP | ❌ Data inconsistency risk |
| ✅ Reduced CPU usage | ❌ Complex update logic |
| ✅ Lower query latency | ❌ Higher maintenance |

---

**Strategies to Manage Denormalization:**

**1. Hybrid Approach:**
```sql
-- Keep normalized tables as source of truth
-- Create denormalized views/tables for reads

-- Normalized (write to these)
CREATE TABLE customers (...);
CREATE TABLE orders (...);

-- Denormalized (read from this)
CREATE MATERIALIZED VIEW orders_with_customer AS
SELECT o.*, c.name, c.email FROM orders o JOIN customers c ...;
```

**2. Async Updates:**
```python
# Update denormalized data asynchronously
def update_customer_name(customer_id, new_name):
    # Update normalized table
    db.execute(
        "UPDATE customers SET name = %s WHERE id = %s",
        (new_name, customer_id)
    )
    
    # Queue async job to update denormalized tables
    queue.enqueue(
        update_denormalized_orders,
        customer_id=customer_id,
        new_name=new_name
    )

def update_denormalized_orders(customer_id, new_name):
    # Update denormalized data in background
    db.execute(
        "UPDATE orders_denormalized SET customer_name = %s WHERE customer_id = %s",
        (new_name, customer_id)
    )
```

**3. Event-Driven Updates:**
```python
# Use events to propagate changes
class CustomerService:
    def update_customer(self, customer_id, data):
        # Update customer
        self.db.update(customer_id, data)
        
        # Publish event
        event_bus.publish('customer.updated', {
            'customer_id': customer_id,
            'name': data['name'],
            'email': data['email']
        })

# Other services listen and update their denormalized data
class OrderService:
    @event_bus.subscribe('customer.updated')
    def on_customer_updated(self, event):
        self.db.execute("""
            UPDATE orders_denormalized 
            SET customer_name = %s, customer_email = %s
            WHERE customer_id = %s
        """, (event['name'], event['email'], event['customer_id']))
```

**4. Cache Invalidation:**
```python
# Invalidate caches when source data changes
def update_product(product_id, data):
    # Update product
    db.execute("UPDATE products SET ... WHERE id = %s", product_id)
    
    # Invalidate denormalized caches
    cache.delete(f"product:{product_id}")
    cache.delete(f"product_search:{product_id}")
    
    # Or: Update cache immediately
    cache.set(f"product:{product_id}", get_product_with_details(product_id))
```

---

**Real-World Examples:**

**1. Social Media Feed (Twitter/Facebook):**
```javascript
// Denormalized feed with embedded user and post data
{
  "feed_item_id": "abc123",
  "user_id": 456,
  "user_name": "John Doe",           // ← Denormalized
  "user_avatar": "https://...",      // ← Denormalized
  "user_verified": true,             // ← Denormalized
  "post_id": 789,
  "post_content": "Hello world!",
  "post_image": "https://...",
  "like_count": 142,                 // ← Denormalized count
  "comment_count": 23,               // ← Denormalized count
  "timestamp": "2024-11-01T10:30:00Z"
}
// Fetching feed doesn't require joins with users/posts tables
```

**2. E-commerce Search (Amazon):**
```sql
-- Highly denormalized for search performance
CREATE TABLE product_search_index (
    product_id INT,
    title VARCHAR(500),
    brand VARCHAR(100),
    category VARCHAR(100),
    price DECIMAL(10,2),
    rating DECIMAL(3,2),            -- ← Aggregated from reviews
    num_reviews INT,                -- ← Counted from reviews
    seller_name VARCHAR(100),       -- ← From sellers table
    seller_rating DECIMAL(3,2),     -- ← From sellers table
    is_prime BOOLEAN,
    in_stock BOOLEAN,
    keywords TEXT[],
    images TEXT[]
);
```

---

**When NOT to Denormalize:**

❌ **Frequently changing data**
- User profile that updates constantly
- Inventory counts that change every second
- Real-time stock prices

❌ **Write-heavy workloads**
- High transaction rate
- Many concurrent updates
- Complex business logic

❌ **Strong consistency requirements**
- Financial transactions
- Legal records
- Audit logs

❌ **Small datasets**
- JOINs are fast enough
- No performance issues
- Additional complexity not worth it

---

**Best Practices:**

1. ✅ Start normalized, denormalize when needed
2. ✅ Measure performance before denormalizing
3. ✅ Keep normalized tables as source of truth
4. ✅ Use materialized views when possible
5. ✅ Implement proper sync mechanisms
6. ✅ Document denormalized fields clearly
7. ✅ Monitor data consistency
8. ✅ Consider eventual consistency acceptable?
9. ✅ Use database triggers or application events
10. ✅ Plan for data migration and backfilling

Answer:


**2. Data lake vs database vs warehouse.**

These are three different data storage architectures, each optimized for specific use cases.

---

**Database (OLTP):**

**Purpose:** Day-to-day transactional operations  
**Data:** Structured, current operational data  
**Schema:** Schema-on-write (defined upfront)  
**Processing:** Real-time transactions (INSERT, UPDATE, DELETE)

**Characteristics:**
- Highly normalized
- ACID transactions
- Low latency (milliseconds)
- Small queries, high frequency
- Optimized for writes and point reads

**Example Technologies:**
- PostgreSQL, MySQL, Oracle
- MongoDB, Cassandra

**Use Case:**
```sql
-- E-commerce transactional database
-- Real-time operations

-- Place order
INSERT INTO orders (customer_id, total, status) 
VALUES (123, 99.99, 'pending');

-- Update inventory
UPDATE products SET stock = stock - 1 WHERE id = 456;

-- Get user profile
SELECT * FROM users WHERE id = 123;
```

---

**Data Warehouse (OLAP):**

**Purpose:** Business intelligence, analytics, and reporting  
**Data:** Structured, historical data from multiple sources  
**Schema:** Schema-on-write (Star/Snowflake schema)  
**Processing:** Complex analytical queries, aggregations

**Characteristics:**
- Denormalized (Star/Snowflake schema)
- Optimized for read-heavy workloads
- Historical data (append-mostly)
- Complex queries, low frequency
- Columnar storage
- ETL/ELT processes

**Architecture:**
```
         ETL/ELT Pipeline
              ↓
┌─────────────────────────────────┐
│      Data Warehouse             │
│                                 │
│   ┌─────────────────────┐      │
│   │   Fact Tables       │      │
│   │  - Sales Facts      │      │
│   │  - Order Facts      │      │
│   └──────────┬──────────┘      │
│              │                  │
│   ┌──────────┴──────────┐      │
│   │  Dimension Tables   │      │
│   │  - Time Dimension   │      │
│   │  - Product Dimension│      │
│   │  - Customer Dim     │      │
│   └─────────────────────┘      │
└─────────────────────────────────┘
```

**Example Technologies:**
- Amazon Redshift, Snowflake, Google BigQuery
- Azure Synapse, Teradata

**Use Case:**
```sql
-- Analytical queries on historical data

-- Sales analysis by region and product category
SELECT 
    d_date.year,
    d_date.quarter,
    d_location.region,
    d_product.category,
    SUM(f_sales.revenue) as total_revenue,
    AVG(f_sales.revenue) as avg_revenue,
    COUNT(DISTINCT f_sales.customer_id) as unique_customers
FROM fact_sales f_sales
JOIN dim_date d_date ON f_sales.date_key = d_date.date_key
JOIN dim_location d_location ON f_sales.location_key = d_location.location_key
JOIN dim_product d_product ON f_sales.product_key = d_product.product_key
WHERE d_date.year >= 2023
GROUP BY d_date.year, d_date.quarter, d_location.region, d_product.category
ORDER BY total_revenue DESC;
```

---

**Data Lake:**

**Purpose:** Store raw data in native format for future analysis  
**Data:** Structured, semi-structured, unstructured (all types)  
**Schema:** Schema-on-read (applied when queried)  
**Processing:** Batch processing, machine learning, exploration

**Characteristics:**
- Stores raw, unprocessed data
- Any data format (JSON, CSV, Parquet, images, logs)
- Cheap storage (object storage like S3)
- Flexible - don't need to know use case upfront
- ELT approach (Extract-Load-Transform)
- Schema applied at read time

**Architecture:**
```
Data Sources → Data Lake (Raw Storage) → Processing Layer → Outputs
                      ↓
          ┌───────────┴───────────┐
          │                       │
    Raw Zone              Curated Zone
    (Bronze)              (Silver/Gold)
    
    - Logs                - Cleaned data
    - JSON files          - Partitioned data
    - Images              - Optimized formats
    - Videos              - Aggregated tables
    - IoT data
```

**Example Technologies:**
- Storage: Amazon S3, Azure Data Lake Storage, Google Cloud Storage
- Processing: Apache Spark, Apache Flink, Databricks
- Query: AWS Athena, Presto, Apache Hive

**Use Case:**
```python
# Store any type of data in raw format

# 1. Ingest raw data (no schema required)
s3.upload_file('user_events.json', 'data-lake/raw/events/2024/11/01/')
s3.upload_file('application.log', 'data-lake/raw/logs/2024/11/01/')
s3.upload_file('video.mp4', 'data-lake/raw/media/2024/11/01/')

# 2. Query when needed (schema on read)
df = spark.read.json('s3://data-lake/raw/events/2024/11/01/')
df.filter(df.event_type == 'purchase').show()

# 3. Process and save to curated zone
df.write.parquet('s3://data-lake/curated/purchases/')
```

---

**Comparison Table:**

| Aspect | Database (OLTP) | Data Warehouse (OLAP) | Data Lake |
|--------|----------------|----------------------|-----------|
| **Data Type** | Structured | Structured | All types (structured, semi, unstructured) |
| **Schema** | Schema-on-write | Schema-on-write | Schema-on-read |
| **Purpose** | Transactions | Analytics | Storage + Future analysis |
| **Data State** | Current | Historical | Raw + Processed |
| **Query Type** | Simple, fast | Complex, aggregations | Ad-hoc, exploratory |
| **Users** | Applications | Business analysts | Data scientists, engineers |
| **Latency** | Milliseconds | Seconds to minutes | Minutes to hours |
| **Data Quality** | High (enforced) | High (cleaned) | Variable (raw data) |
| **Cost** | High (per GB) | Medium | Low (object storage) |
| **Normalization** | Normalized | Denormalized | N/A (raw files) |
| **Updates** | Frequent | Batch (periodic) | Immutable (append-only) |
| **Size** | Gigabytes | Terabytes | Petabytes |
| **Examples** | PostgreSQL, MySQL | Snowflake, Redshift | S3 + Spark, ADLS |

---

**Evolution: Database → Warehouse → Lake**

```
1990s: Databases
- Just operational databases
- Basic reporting from production DB
- Limited analytics

2000s: Data Warehouses  
- Separate from operational systems
- ETL pipelines to move data
- Structured analytics
- Star schemas, OLAP cubes

2010s: Data Lakes
- Big Data era (Hadoop)
- Store everything cheaply
- Flexibility for data scientists
- Machine learning, AI

2020s: Data Lakehouse (Hybrid)
- Best of both worlds
- Delta Lake, Apache Iceberg
- ACID transactions + Flexibility
- Example: Databricks Lakehouse
```

---

**Modern Architecture (Lambda Architecture):**

```
┌──────────────┐
│Data Sources  │
└──────┬───────┘
       │
       ├──────────────────┐
       │                  │
       ▼                  ▼
┌──────────────┐   ┌──────────────┐
│ Speed Layer  │   │ Batch Layer  │
│ (Real-time)  │   │ (Historical) │
│              │   │              │
│  Database    │   │  Data Lake   │
│  Streaming   │   │  Spark Jobs  │
└──────┬───────┘   └──────┬───────┘
       │                  │
       └────────┬─────────┘
                ▼
         ┌──────────────┐
         │Serving Layer │
         │              │
         │ Data Warehouse│
         │  (Analytics)  │
         └──────────────┘
```

---

**Real-World Example: E-commerce Company**

**Database (Transactional):**
```sql
-- PostgreSQL: Handle real-time operations
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    product_id INT,
    quantity INT,
    price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Fast inserts/updates
INSERT INTO orders VALUES (...);
UPDATE orders SET status = 'shipped' WHERE order_id = 123;
```

**Data Lake (Raw Storage):**
```
s3://company-data-lake/
├── raw/
│   ├── orders/
│   │   ├── 2024/11/01/orders.json
│   │   └── 2024/11/02/orders.json
│   ├── clickstream/
│   │   └── 2024/11/01/events.parquet
│   ├── images/
│   │   └── products/*.jpg
│   └── logs/
│       └── application.log
└── processed/
    └── aggregated_sales/
        └── daily_summary.parquet
```

**Data Warehouse (Analytics):**
```sql
-- Snowflake: Complex analytical queries
CREATE TABLE fact_sales (
    sale_id INT,
    date_key INT,
    customer_key INT,
    product_key INT,
    revenue DECIMAL(10,2),
    quantity INT
);

-- Analytical query
SELECT 
    d.month_name,
    p.category,
    SUM(f.revenue) as total_revenue
FROM fact_sales f
JOIN dim_date d ON f.date_key = d.date_key
JOIN dim_product p ON f.product_key = p.product_key
GROUP BY d.month_name, p.category;
```

---

**Data Flow Example:**

```
1. User places order
   → Write to Database (PostgreSQL)
   
2. CDC (Change Data Capture)
   → Stream to Kafka
   
3. Land in Data Lake
   → Store raw JSON in S3
   
4. ETL Process (nightly)
   → Spark reads from Data Lake
   → Transforms and aggregates
   → Loads into Data Warehouse
   
5. Business analysts
   → Query Data Warehouse (Snowflake)
   → Create dashboards (Tableau)
   
6. Data scientists
   → Read from Data Lake
   → Train ML models
   → Deploy predictions back to Database
```

---

**When to Use Each:**

**Use Database When:**
- ✅ Real-time transactions
- ✅ Strong ACID requirements
- ✅ Application backend
- ✅ Current operational data
- ✅ Low latency critical

**Use Data Warehouse When:**
- ✅ Business intelligence
- ✅ Historical analysis
- ✅ Reporting and dashboards
- ✅ Structured analytical queries
- ✅ Aggregated metrics

**Use Data Lake When:**
- ✅ Storing diverse data types
- ✅ Uncertain future use cases
- ✅ Machine learning pipelines
- ✅ Long-term archival
- ✅ Cost-effective storage
- ✅ Big data processing

---

**Modern Trend: Data Lakehouse**

Combines best of data lake and warehouse:

```sql
-- Delta Lake (on S3/ADLS)
-- ACID transactions on data lake
-- Schema enforcement + evolution
-- Time travel

CREATE TABLE sales
USING DELTA
LOCATION 's3://data-lake/sales/'
AS SELECT * FROM raw_sales;

-- Query like a database
SELECT * FROM sales WHERE date > '2024-01-01';

-- Time travel
SELECT * FROM sales VERSION AS OF 10;

-- ACID updates on data lake!
UPDATE sales SET status = 'completed' WHERE order_id = 123;
```

**Technologies:**
- Databricks Delta Lake
- Apache Iceberg
- Apache Hudi

Answer:


**3. Batch vs streaming ingestion.**

**Batch Ingestion** and **Streaming Ingestion** are two approaches for moving data from sources to destination systems.

**Batch Ingestion:** Collecting and processing data in large chunks at scheduled intervals (hourly, daily). High latency but cost-effective for large volumes.

**Streaming Ingestion:** Continuously processing data as it arrives in real-time (milliseconds to seconds). Low latency but higher infrastructure cost.

**Comparison:**

| Aspect | Batch | Streaming |
|--------|-------|-----------|
| **Latency** | Hours to days | Milliseconds to seconds |
| **Data Volume** | Large chunks | Continuous small events |
| **Complexity** | Lower | Higher |
| **Cost** | Lower | Higher |
| **Use Case** | Historical analysis | Real-time decisions |
| **Tools** | Airflow, Spark | Kafka, Flink, Kinesis |

**Best Practice:** Use streaming for time-sensitive data (fraud detection, real-time dashboards) and batch for cost-effective bulk processing (daily reports, data warehouse loading). Combine both in Lambda architecture for comprehensive solution.

Answer:


**4. Designing schema for analytics-heavy systems.**

**Goal:** Design databases optimized for complex analytical queries and reporting.

**Key Pattern: Star Schema**

```sql
-- Fact table (center): Measurements/metrics
CREATE TABLE fact_sales (
    sale_id BIGINT PRIMARY KEY,
    date_key INT,           -- FK to dim_date
    product_key INT,        -- FK to dim_product  
    customer_key INT,       -- FK to dim_customer
    revenue DECIMAL(10,2),
    quantity INT,
    profit DECIMAL(10,2)
);

-- Dimension tables (points): Descriptive attributes
CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    date DATE,
    month_name VARCHAR(10),
    quarter INT,
    year INT
);

CREATE TABLE dim_product (
    product_key INT PRIMARY KEY,
    product_name VARCHAR(200),
    category VARCHAR(100),
    brand VARCHAR(100)
);
```

**Why Star Schema:**
- Simple JOINs (fact → dimension)
- Fast query performance
- Optimized for BI tools
- Denormalized dimensions (fewer JOINs)

**Other Key Techniques:**

1. **Columnar Storage** (Redshift, BigQuery) - Only read columns needed
2. **Partitioning** - Partition fact tables by date for faster queries
3. **Aggregation Tables** - Pre-compute common metrics
4. **Slowly Changing Dimensions (SCD Type 2)** - Track historical changes
5. **Surrogate Keys** - Use auto-increment IDs instead of natural keys

**Best Practices:**
- ✅ Denormalize dimensions
- ✅ Partition large fact tables
- ✅ Use columnar storage
- ✅ Pre-aggregate frequently queried metrics
- ✅ Index foreign keys

Answer:


---

### C. Real System Design Scenarios

**1. Design a scalable logging system (PostgreSQL + Elasticsearch).**

**Requirements:**
- Ingest millions of logs per second
- Full-text search across logs
- Retention: 90 days hot, 1 year cold
- Query logs by timestamp, level, service, message

**Architecture:**

```
Application Servers
     │
     ├─► Kafka (log buffer)
     │        │
     │        ├─► Stream Processor (Logstash/Flink)
     │        │         │
     │        │         ├─► Elasticsearch (search + hot data)
     │        │         └─► PostgreSQL (structured queries)
     │        │
     │        └─► S3 (long-term archive)
     │
     └─► Query Layer
              ├─► Elasticsearch (text search, recent logs)
              ├─► PostgreSQL (structured queries, aggregations)
              └─► S3 (old data via Athena)
```

**Component Responsibilities:**

**1. Kafka (Ingestion Layer):**
```python
# High-throughput log ingestion
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['kafka:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Application logs to Kafka
log_entry = {
    'timestamp': '2024-11-01T10:30:00Z',
    'level': 'ERROR',
    'service': 'payment-service',
    'message': 'Payment failed for order 12345',
    'metadata': {'user_id': 789, 'order_id': 12345}
}

producer.send('application-logs', log_entry)
```

**2. Elasticsearch (Search + Hot Data - Last 7 days):**
```json
// Index template for logs
{
  "mappings": {
    "properties": {
      "timestamp": {"type": "date"},
      "level": {"type": "keyword"},
      "service": {"type": "keyword"},
      "message": {"type": "text"},
      "metadata": {"type": "object"}
    }
  },
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 1
  }
}

// Full-text search
GET /logs-2024-11-01/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"message": "payment failed"}},
        {"term": {"level": "ERROR"}},
        {"range": {"timestamp": {"gte": "now-1h"}}}
      ]
    }
  }
}
```

**3. PostgreSQL (Structured Queries - 90 days):**
```sql
-- Aggregated metrics and structured queries
CREATE TABLE log_summary (
    id BIGSERIAL PRIMARY KEY,
    timestamp TIMESTAMP NOT NULL,
    service VARCHAR(100),
    level VARCHAR(20),
    count INT,
    INDEX idx_service_time (service, timestamp),
    INDEX idx_level_time (level, timestamp)
) PARTITION BY RANGE (timestamp);

-- Query: Error rate by service
SELECT 
    service,
    DATE_TRUNC('hour', timestamp) as hour,
    SUM(CASE WHEN level = 'ERROR' THEN count ELSE 0 END) as errors,
    SUM(count) as total,
    (SUM(CASE WHEN level = 'ERROR' THEN count ELSE 0 END)::float / SUM(count) * 100) as error_rate
FROM log_summary
WHERE timestamp > NOW() - INTERVAL '24 hours'
GROUP BY service, hour
ORDER BY error_rate DESC;
```

**4. Data Flow:**
```python
# Stream processor (Flink/Logstash)
def process_log(log):
    # 1. Send to Elasticsearch for search
    es_client.index(
        index=f"logs-{date.today()}",
        document=log
    )
    
    # 2. Aggregate and store in PostgreSQL
    db.execute("""
        INSERT INTO log_summary (timestamp, service, level, count)
        VALUES (%s, %s, %s, 1)
        ON CONFLICT (timestamp, service, level)
        DO UPDATE SET count = log_summary.count + 1
    """, (log['timestamp'], log['service'], log['level']))
    
    # 3. Archive to S3 for long-term storage
    if should_archive(log):
        s3.put_object(
            Bucket='logs-archive',
            Key=f"year={year}/month={month}/day={day}/{uuid}.json",
            Body=json.dumps(log)
        )
```

**5. Retention Policy:**
```python
# Elasticsearch: Delete old indices (7 days)
DELETE /logs-2024-10-24

# PostgreSQL: Drop old partitions (90 days)
DROP TABLE log_summary_2024_08;

# S3: Lifecycle policy (1 year → Glacier, 3 years → Delete)
```

**Query Layer:**
```python
def search_logs(query, time_range):
    if time_range <= '7 days':
        # Recent data: Elasticsearch (fast full-text search)
        return elasticsearch.search(query)
    elif time_range <= '90 days':
        # Medium-term: PostgreSQL (structured queries)
        return postgres.query(query)
    else:
        # Old data: S3 + Athena (cold storage)
        return athena.query(f"SELECT * FROM logs_archive WHERE {query}")
```

**Scaling:**
- Kafka: Add partitions for throughput
- Elasticsearch: Add nodes, use hot-warm architecture
- PostgreSQL: Read replicas, partitioning
- S3: Unlimited scale

**Best Practices:**
- ✅ Use time-based indices in Elasticsearch
- ✅ Partition PostgreSQL tables by date
- ✅ Implement log sampling for high-volume services
- ✅ Use async processing (don't block application)
- ✅ Monitor pipeline lag

Answer:


**2. Design a recommendation engine database layer.**

**Requirements:**
- Recommend products based on user behavior and preferences
- Real-time personalization
- Collaborative filtering + content-based filtering
- Handle millions of users and products

**Architecture:**

```
┌──────────────────────────────────────────┐
│         User Activity Tracking           │
│    (Clicks, views, purchases, ratings)   │
└────────────┬─────────────────────────────┘
             │
             ▼
      ┌─────────────┐
      │    Kafka    │
      └──────┬──────┘
             │
      ┌──────┴──────────────────────────────┐
      │                                      │
      ▼                                      ▼
┌─────────────┐                      ┌──────────────┐
│  PostgreSQL │                      │    Redis     │
│ (User/Item  │                      │  (Real-time  │
│  metadata)  │                      │   cache)     │
└─────────────┘                      └──────────────┘
      │                                      │
      │                                      │
      ▼                                      ▼
┌─────────────┐                      ┌──────────────┐
│  Cassandra  │                      │ Elasticsearch│
│ (User-Item  │                      │  (Content-   │
│ interactions)│                      │   based)     │
└─────────────┘                      └──────────────┘
      │                                      │
      └──────────────┬───────────────────────┘
                     ▼
             ┌───────────────┐
             │  ML Pipeline  │
             │ (Spark/Flink) │
             └───────┬───────┘
                     ▼
             ┌───────────────┐
             │  Vector DB    │
             │  (Pinecone/   │
             │   Milvus)     │
             └───────────────┘
```

**Database Design:**

**1. PostgreSQL (User & Product Metadata):**
```sql
-- Users table
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    username VARCHAR(100),
    age INT,
    location VARCHAR(100),
    preferences JSONB,
    created_at TIMESTAMP
);

-- Products table
CREATE TABLE products (
    product_id BIGSERIAL PRIMARY KEY,
    name VARCHAR(200),
    category VARCHAR(100),
    brand VARCHAR(100),
    price DECIMAL(10,2),
    features JSONB,  -- For content-based filtering
    avg_rating DECIMAL(3,2),
    INDEX idx_category (category)
);

-- Product features for content-based filtering
CREATE TABLE product_features (
    product_id BIGINT,
    feature_name VARCHAR(100),
    feature_value TEXT,
    INDEX idx_product (product_id),
    INDEX idx_feature (feature_name, feature_value)
);
```

**2. Cassandra (User-Item Interactions - High write throughput):**
```sql
-- User interaction events (time-series)
CREATE TABLE user_interactions (
    user_id BIGINT,
    timestamp TIMESTAMP,
    product_id BIGINT,
    interaction_type VARCHAR(20),  -- view, click, purchase, rating
    interaction_value FLOAT,       -- rating value if applicable
    session_id UUID,
    PRIMARY KEY ((user_id), timestamp, product_id)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- Materialized view for recent interactions
CREATE MATERIALIZED VIEW recent_user_interactions AS
    SELECT user_id, product_id, interaction_type, interaction_value
    FROM user_interactions
    WHERE user_id IS NOT NULL 
      AND timestamp IS NOT NULL
      AND timestamp > now() - 30 days
    PRIMARY KEY (user_id, timestamp);

-- Product popularity
CREATE TABLE product_stats (
    product_id BIGINT,
    date DATE,
    view_count COUNTER,
    purchase_count COUNTER,
    rating_sum COUNTER,
    rating_count COUNTER,
    PRIMARY KEY (product_id, date)
);
```

**3. Redis (Real-time Cache):**
```python
# Cache pre-computed recommendations
redis.setex(
    f"recommendations:user:{user_id}",
    3600,  # 1 hour TTL
    json.dumps({
        'products': [123, 456, 789],
        'scores': [0.95, 0.89, 0.85],
        'computed_at': time.time()
    })
)

# User session data (recent views)
redis.zadd(
    f"session:user:{user_id}",
    {product_id: timestamp for product_id, timestamp in recent_views}
)

# Trending products (sorted set by popularity)
redis.zincrby("trending:products", 1, product_id)
```

**4. Elasticsearch (Content-Based Search):**
```json
// Product index for similarity search
{
  "mappings": {
    "properties": {
      "product_id": {"type": "long"},
      "name": {"type": "text"},
      "description": {"type": "text"},
      "category": {"type": "keyword"},
      "brand": {"type": "keyword"},
      "tags": {"type": "keyword"},
      "price": {"type": "float"}
    }
  }
}

// More Like This query (content-based recommendations)
GET /products/_search
{
  "query": {
    "more_like_this": {
      "fields": ["name", "description", "tags"],
      "like": [{"_id": "123"}],
      "min_term_freq": 1,
      "max_query_terms": 12
    }
  }
}
```

**5. Vector Database (Embedding-Based Recommendations):**
```python
# Store product embeddings
from pinecone import Pinecone

pc = Pinecone(api_key="...")
index = pc.Index("product-embeddings")

# Insert product vectors
index.upsert([
    (str(product_id), embedding_vector, {"category": "electronics", "price": 299.99})
    for product_id, embedding_vector in product_embeddings
])

# Find similar products
results = index.query(
    vector=user_preference_vector,
    top_k=10,
    include_metadata=True
)
```

**Recommendation Generation:**

**1. Collaborative Filtering (User-User / Item-Item):**
```python
def collaborative_filtering(user_id):
    # Find similar users based on interaction history
    similar_users = cassandra.execute("""
        SELECT similar_user_id, similarity_score
        FROM user_similarity
        WHERE user_id = %s
        LIMIT 100
    """, [user_id])
    
    # Get products liked by similar users
    recommended_products = []
    for similar_user in similar_users:
        products = cassandra.execute("""
            SELECT product_id, interaction_value
            FROM user_interactions
            WHERE user_id = %s
              AND interaction_type = 'purchase'
        """, [similar_user.id])
        
        recommended_products.extend(products)
    
    return rank_products(recommended_products)
```

**2. Content-Based Filtering:**
```python
def content_based_recommendations(user_id):
    # Get user's past purchases
    user_products = postgres.query("""
        SELECT product_id FROM purchases WHERE user_id = %s
    """, [user_id])
    
    # Find similar products using Elasticsearch
    recommendations = []
    for product_id in user_products:
        similar = elasticsearch.more_like_this(product_id)
        recommendations.extend(similar)
    
    return deduplicate_and_rank(recommendations)
```

**3. Hybrid Approach:**
```python
def get_recommendations(user_id):
    # Check cache first
    cached = redis.get(f"recommendations:user:{user_id}")
    if cached:
        return json.loads(cached)
    
    # Generate recommendations
    collab_recs = collaborative_filtering(user_id)  # Weight: 0.6
    content_recs = content_based_recommendations(user_id)  # Weight: 0.3
    trending_recs = get_trending_products()  # Weight: 0.1
    
    # Merge and rank
    final_recs = merge_recommendations(
        collab_recs, content_recs, trending_recs,
        weights=[0.6, 0.3, 0.1]
    )
    
    # Cache results
    redis.setex(
        f"recommendations:user:{user_id}",
        3600,
        json.dumps(final_recs)
    )
    
    return final_recs
```

**Batch Processing (Offline):**
```python
# Spark job: Compute user-user similarity (nightly)
from pyspark.ml.recommendation import ALS

# Load interaction data from Cassandra
interactions = spark.read.format("org.apache.spark.sql.cassandra") \
    .options(table="user_interactions", keyspace="recommendations").load()

# Train collaborative filtering model
als = ALS(userCol="user_id", itemCol="product_id", ratingCol="interaction_value")
model = als.fit(interactions)

# Generate recommendations for all users
user_recs = model.recommendForAllUsers(20)

# Store in Cassandra for serving
user_recs.write.format("org.apache.spark.sql.cassandra") \
    .options(table="user_recommendations", keyspace="recommendations").save()
```

**Real-Time Updates:**
```python
# Kafka consumer: Update user profile in real-time
def process_user_event(event):
    if event['type'] == 'purchase':
        # Update interaction history
        cassandra.execute("""
            INSERT INTO user_interactions (user_id, timestamp, product_id, interaction_type)
            VALUES (%s, %s, %s, 'purchase')
        """, [event['user_id'], event['timestamp'], event['product_id']])
        
        # Invalidate recommendation cache
        redis.delete(f"recommendations:user:{event['user_id']}")
        
        # Trigger real-time recommendation update
        update_recommendations_async(event['user_id'])
```

**Scalability:**
- Cassandra: Distributed, handles billions of interactions
- Redis: In-memory cache for low-latency reads
- Elasticsearch: Scales horizontally for content search
- Vector DB: Optimized for similarity search at scale

**Best Practices:**
- ✅ Cache recommendations (Redis)
- ✅ Pre-compute offline (Spark batch jobs)
- ✅ Real-time updates for recent activity
- ✅ A/B test recommendation algorithms
- ✅ Monitor click-through rate (CTR) and conversion

Answer:


**3. Handle billions of time-series events per day (TimescaleDB vs Cassandra).**

**Scenario:** IoT system ingesting sensor data from millions of devices

**Requirements:**
- 10 billion events/day (~115K events/second)
- Metrics: temperature, humidity, pressure per device
- Query: Recent data, time-range aggregations, per-device analysis
- Retention: 7 days hot, 90 days cold, 2 years archived

---

**Solution 1: TimescaleDB (Time-Series Extension for PostgreSQL)**

**Schema:**
```sql
-- Create hypertable (automatic partitioning by time)
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL,
    device_id INT NOT NULL,
    temperature DECIMAL(5,2),
    humidity DECIMAL(5,2),
    pressure DECIMAL(6,2),
    location VARCHAR(100),
    metadata JSONB
);

-- Convert to hypertable (partitioned by time)
SELECT create_hypertable('sensor_readings', 'time');

-- Add indexes
CREATE INDEX idx_device_time ON sensor_readings (device_id, time DESC);
CREATE INDEX idx_location_time ON sensor_readings (location, time DESC);
```

**Continuous Aggregates (Pre-computed rollups):**
```sql
-- Materialized view for hourly aggregates
CREATE MATERIALIZED VIEW sensor_readings_hourly
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) AS bucket,
    device_id,
    AVG(temperature) as avg_temp,
    MAX(temperature) as max_temp,
    MIN(temperature) as min_temp,
    AVG(humidity) as avg_humidity,
    COUNT(*) as reading_count
FROM sensor_readings
GROUP BY bucket, device_id;

-- Auto-refresh policy
SELECT add_continuous_aggregate_policy('sensor_readings_hourly',
    start_offset => INTERVAL '2 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');
```

**Compression:**
```sql
-- Enable compression for old data (saves 90%+ storage)
ALTER TABLE sensor_readings SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Auto-compress data older than 2 days
SELECT add_compression_policy('sensor_readings', INTERVAL '2 days');
```

**Retention Policy:**
```sql
-- Drop data older than 90 days
SELECT add_retention_policy('sensor_readings', INTERVAL '90 days');
```

**Queries:**
```sql
-- Recent data for specific device
SELECT * FROM sensor_readings
WHERE device_id = 12345 
  AND time > NOW() - INTERVAL '1 hour'
ORDER BY time DESC;

-- Aggregated query (uses continuous aggregate)
SELECT bucket, avg_temp, max_temp
FROM sensor_readings_hourly
WHERE device_id = 12345
  AND bucket >= NOW() - INTERVAL '7 days';

-- Multi-device aggregation
SELECT 
    time_bucket('5 minutes', time) AS bucket,
    location,
    AVG(temperature) as avg_temp
FROM sensor_readings
WHERE time > NOW() - INTERVAL '1 day'
GROUP BY bucket, location
ORDER BY bucket DESC;
```

**Pros:**
- ✅ SQL interface (familiar)
- ✅ ACID transactions
- ✅ Automatic compression
- ✅ Continuous aggregates
- ✅ Good for complex queries

**Cons:**
- ❌ Single-node write bottleneck (unless using distributed TimescaleDB)
- ❌ Limited horizontal scalability
- ❌ More expensive than Cassandra for massive scale

---

**Solution 2: Cassandra (Distributed NoSQL)**

**Schema:**
```sql
-- Partition by device and time bucket for distribution
CREATE TABLE sensor_readings (
    device_id INT,
    date DATE,  -- Partition key component
    timestamp TIMESTAMP,
    temperature DECIMAL,
    humidity DECIMAL,
    pressure DECIMAL,
    location TEXT,
    metadata MAP<TEXT, TEXT>,
    PRIMARY KEY ((device_id, date), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC)
  AND compaction = {'class': 'TimeWindowCompactionStrategy'}
  AND default_time_to_live = 7776000;  -- 90 days TTL

-- Materialized view for location-based queries
CREATE MATERIALIZED VIEW readings_by_location AS
    SELECT device_id, location, timestamp, temperature
    FROM sensor_readings
    WHERE location IS NOT NULL AND date IS NOT NULL AND timestamp IS NOT NULL
    PRIMARY KEY ((location, date), timestamp);
```

**Write Path:**
```python
from cassandra.cluster import Cluster
from cassandra.query import BatchStatement

cluster = Cluster(['cassandra1', 'cassandra2', 'cassandra3'])
session = cluster.connect('iot_data')

# High-throughput writes
def insert_reading(device_id, timestamp, temperature, humidity, pressure):
    session.execute("""
        INSERT INTO sensor_readings 
        (device_id, date, timestamp, temperature, humidity, pressure)
        VALUES (%s, %s, %s, %s, %s, %s)
        USING TTL 7776000
    """, (device_id, date.today(), timestamp, temperature, humidity, pressure))

# Batch inserts for efficiency
batch = BatchStatement()
for reading in batch_readings:
    batch.add(insert_statement, reading)
session.execute(batch)
```

**Queries:**
```sql
-- Recent data for device (efficient - partition key)
SELECT * FROM sensor_readings
WHERE device_id = 12345 
  AND date = '2024-11-01'
  AND timestamp > '2024-11-01 10:00:00'
ORDER BY timestamp DESC;

-- Location-based query (uses materialized view)
SELECT * FROM readings_by_location
WHERE location = 'Building-A'
  AND date = '2024-11-01';

-- Aggregation (application-level or Spark)
-- Cassandra doesn't support complex aggregations natively
```

**Aggregations with Spark:**
```python
# Read from Cassandra, aggregate with Spark
from pyspark.sql import SparkSession

spark = SparkSession.builder \
    .config("spark.cassandra.connection.host", "cassandra1") \
    .getOrCreate()

readings = spark.read \
    .format("org.apache.spark.sql.cassandra") \
    .options(table="sensor_readings", keyspace="iot_data") \
    .load()

# Compute hourly averages
hourly_avg = readings \
    .groupBy("device_id", window("timestamp", "1 hour")) \
    .agg(avg("temperature").alias("avg_temp"))

# Write back to Cassandra
hourly_avg.write \
    .format("org.apache.spark.sql.cassandra") \
    .options(table="sensor_readings_hourly", keyspace="iot_data") \
    .save()
```

**Pros:**
- ✅ Massive horizontal scalability
- ✅ High write throughput (linear scaling)
- ✅ Multi-datacenter replication
- ✅ No single point of failure
- ✅ Lower cost at extreme scale

**Cons:**
- ❌ No JOINs or complex queries
- ❌ Limited aggregation support
- ❌ Eventual consistency
- ❌ Query flexibility limited by schema design

---

**Comparison:**

| Aspect | TimescaleDB | Cassandra |
|--------|-------------|-----------|
| **Write Throughput** | 100K-500K/sec (single node) | 1M+/sec (distributed) |
| **Scalability** | Vertical (or distributed version) | Horizontal (unlimited) |
| **Query Flexibility** | High (full SQL) | Limited (by partition key) |
| **Aggregations** | Native (continuous aggregates) | External (Spark) |
| **Consistency** | Strong (ACID) | Tunable (eventual by default) |
| **Compression** | Excellent (built-in) | Good (via compaction) |
| **Ops Complexity** | Lower | Higher |
| **Cost** | Higher per node | Lower (commodity hardware) |
| **Best For** | <1B events/day, complex queries | >10B events/day, simple queries |

---

**Recommendation:**

**Use TimescaleDB when:**
- Data volume < 10 billion events/day
- Need complex SQL queries
- Want automatic compression and aggregation
- Prefer operational simplicity
- Budget for powerful servers

**Use Cassandra when:**
- Data volume > 10 billion events/day
- Simple query patterns (by device, time range)
- Need multi-datacenter replication
- Want linear scalability
- Cost-sensitive (use commodity hardware)

**Hybrid Approach:**
```
Ingestion: Kafka → Cassandra (raw data storage)
           ↓
Processing: Spark (aggregations)
           ↓
Serving: TimescaleDB (aggregated data for dashboards)
```

This combines Cassandra's write scalability with TimescaleDB's query flexibility.

Answer:


**4. Design fault-tolerant architecture for global read replicas.**

**Goal:** Design a globally distributed database architecture that provides:
- Low-latency reads from any region
- High availability (survives regional failures)
- Data consistency across replicas
- Automatic failover

**Architecture:**

```
Global Distribution:

┌─────────────────────────────────────────────────────────┐
│                       US-EAST-1 (Primary)                │
│                                                          │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐     │
│  │ Primary  │◄────►│ Replica  │◄────►│ Replica  │     │
│  │  (RW)    │      │  (R)     │      │  (R)     │     │
│  └────┬─────┘      └──────────┘      └──────────┘     │
└───────┼──────────────────────────────────────────────────┘
        │
        │ (Async Replication)
        │
        ├────────────────────────┬────────────────────────┐
        │                        │                        │
        ▼                        ▼                        ▼
┌────────────────┐      ┌────────────────┐      ┌────────────────┐
│   EU-WEST-1    │      │   AP-SOUTH-1   │      │  SA-EAST-1     │
│    (Replica)   │      │    (Replica)   │      │   (Replica)    │
│                │      │                │      │                │
│  ┌──────────┐  │      │  ┌──────────┐  │      │  ┌──────────┐  │
│  │ Replica  │  │      │  │ Replica  │  │      │  │ Replica  │  │
│  │  (R)     │  │      │  │  (R)     │  │      │  │  (R)     │  │
│  └──────────┘  │      │  └──────────┘  │      │  └──────────┘  │
│  ┌──────────┐  │      │  ┌──────────┐  │      │  ┌──────────┐  │
│  │ Replica  │  │      │  │ Replica  │  │      │  │ Replica  │  │
│  │  (R)     │  │      │  │  (R)     │  │      │  │  (R)     │  │
│  └──────────┘  │      │  └──────────┘  │      │  └──────────┘  │
└────────────────┘      └────────────────┘      └────────────────┘
```

---

**Implementation Options:**

**Option 1: PostgreSQL with Multi-Region Replication**

**Setup:**
```sql
-- Primary (US-EAST-1): postgresql.conf
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on
hot_standby_feedback = on

-- Create replication user
CREATE USER replicator REPLICATION LOGIN ENCRYPTED PASSWORD 'secure_password';

-- Create publication (for logical replication)
CREATE PUBLICATION global_publication FOR ALL TABLES;
```

**Replicas (EU, AP, SA):**
```bash
# Create subscription on each read replica
psql -h replica-eu.example.com -U postgres

CREATE SUBSCRIPTION eu_subscription
    CONNECTION 'host=primary-us.example.com port=5432 user=replicator password=secure_password dbname=production'
    PUBLICATION global_publication
    WITH (copy_data = true, create_slot = true);

# Check replication lag
SELECT 
    slot_name,
    pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag_size,
    confirmed_flush_lsn
FROM pg_replication_slots;
```

**Connection Routing (HAProxy/PgBouncer):**
```bash
# haproxy.cfg - Route writes to primary, reads to nearest replica

frontend postgres_frontend
    bind *:5432
    default_backend postgres_primary

backend postgres_primary
    mode tcp
    option pgsql-check user postgres
    server primary primary-us.example.com:5432 check

backend postgres_replicas_eu
    mode tcp
    balance roundrobin
    option pgsql-check user postgres
    server replica1 replica1-eu.example.com:5432 check
    server replica2 replica2-eu.example.com:5432 check

# Application routing
def get_db_connection(operation_type, user_location):
    if operation_type == 'write':
        return connect_to_primary()
    else:
        # Route reads to nearest replica
        replica = get_nearest_replica(user_location)
        return connect_to_replica(replica)
```

**Automatic Failover with Patroni:**
```yaml
# Patroni configuration for HA
scope: postgres-cluster
name: node1

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      parameters:
        max_connections: 100
        max_worker_processes: 8

postgresql:
  listen: 0.0.0.0:5432
  connect_address: node1.example.com:5432
  data_dir: /var/lib/postgresql/data
  authentication:
    replication:
      username: replicator
      password: secure_password

# Patroni handles:
# - Leader election
# - Automatic failover
# - Health checks
# - Replica promotion
```

---

**Option 2: Aurora Global Database (AWS-specific)**

```sql
-- Create Aurora Global Database
CREATE GLOBAL DATABASE my-global-db
    ENGINE aurora-postgresql
    ENGINE_VERSION 14.6;

-- Primary cluster (US-EAST-1)
CREATE DB CLUSTER primary-cluster
    GLOBAL CLUSTER my-global-db
    REGION us-east-1
    INSTANCE_CLASS db.r6g.xlarge;

-- Secondary clusters (read replicas in other regions)
ADD REGION TO GLOBAL DATABASE my-global-db
    REGION eu-west-1
    INSTANCE_CLASS db.r6g.xlarge;

ADD REGION TO GLOBAL DATABASE my-global-db
    REGION ap-south-1
    INSTANCE_CLASS db.r6g.xlarge;
```

**Benefits:**
- Replication lag < 1 second
- Automatic failover (promote secondary to primary)
- Up to 5 secondary regions
- Cross-region disaster recovery

---

**Option 3: Multi-Master with Cassandra**

```sql
-- Cassandra multi-datacenter setup
-- All regions can accept writes

CREATE KEYSPACE global_data WITH replication = {
    'class': 'NetworkTopologyStrategy',
    'us-east': 3,
    'eu-west': 3,
    'ap-south': 3,
    'sa-east': 3
};

-- Application: Write LOCAL_QUORUM, Read LOCAL_QUORUM
INSERT INTO users (id, name, email) VALUES (1, 'John', 'john@example.com')
USING CONSISTENCY LOCAL_QUORUM;

SELECT * FROM users WHERE id = 1
USING CONSISTENCY LOCAL_QUORUM;

-- Cross-datacenter replication happens asynchronously
```

**Pros:**
- No single primary (all regions writable)
- Automatic failover (no manual promotion)
- Linear scalability

**Cons:**
- Eventual consistency
- Conflict resolution needed

---

**Failover Strategies:**

**1. Automated Failover (Patroni + etcd):**
```python
# Patroni monitors primary health
# If primary fails:
# 1. Elect new leader from replicas
# 2. Promote replica to primary
# 3. Update DNS/routing
# 4. Reconfigure other replicas

# Health check
def check_primary_health():
    try:
        conn = psycopg2.connect(primary_connection_string, connect_timeout=5)
        conn.cursor().execute("SELECT 1")
        return True
    except:
        trigger_failover()
        return False

def trigger_failover():
    # Patroni handles this automatically
    # 1. Determine best replica (least lag)
    # 2. Promote to primary
    # 3. Update etcd/consul
    # 4. Route traffic to new primary
    logger.info("Failover initiated")
```

**2. Monitoring Replication Lag:**
```sql
-- Monitor lag on each replica
CREATE VIEW replication_status AS
SELECT 
    application_name,
    client_addr,
    state,
    sync_state,
    pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn) AS sending_lag,
    pg_wal_lsn_diff(sent_lsn, flush_lsn) AS flushing_lag,
    pg_wal_lsn_diff(flush_lsn, replay_lsn) AS replaying_lag,
    pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS total_lag
FROM pg_stat_replication;

-- Alert if lag > 10MB or 60 seconds
SELECT * FROM replication_status 
WHERE total_lag > 10485760  -- 10MB
   OR (clock_timestamp() - pg_last_xact_replay_timestamp()) > INTERVAL '60 seconds';
```

**3. Read Replica Routing:**
```python
# GeoDNS / Route53 latency-based routing
class DatabaseRouter:
    def __init__(self):
        self.primary = "primary-us.example.com"
        self.replicas = {
            'us': 'replica-us.example.com',
            'eu': 'replica-eu.example.com',
            'ap': 'replica-ap.example.com',
            'sa': 'replica-sa.example.com'
        }
    
    def get_connection(self, operation_type, user_region):
        if operation_type == 'write':
            return self.connect_to_primary()
        else:
            # Read from nearest replica
            replica_host = self.replicas.get(user_region, self.replicas['us'])
            return self.connect_to_replica(replica_host)
    
    def connect_with_retry(self, host, max_retries=3):
        for attempt in range(max_retries):
            try:
                return psycopg2.connect(
                    host=host,
                    database='production',
                    user='app_user',
                    password='password',
                    connect_timeout=5
                )
            except psycopg2.OperationalError:
                if attempt == max_retries - 1:
                    # Fallback to primary
                    return self.connect_to_primary()
                time.sleep(2 ** attempt)  # Exponential backoff
```

---

**Disaster Recovery:**

```bash
# Regular backups to S3
pg_basebackup -h primary-us.example.com -D /backup -Ft -z -P

# Point-in-Time Recovery (PITR)
# Restore from backup + apply WAL logs

# Cross-region backup replication
aws s3 sync s3://backups-us-east-1/postgres/ s3://backups-eu-west-1/postgres/ --region eu-west-1

# Test failover regularly (chaos engineering)
```

---

**Best Practices:**

1. ✅ Use synchronous replication within region (consistency)
2. ✅ Use asynchronous replication across regions (performance)
3. ✅ Monitor replication lag continuously
4. ✅ Implement automatic failover (Patroni, Aurora)
5. ✅ Route reads to nearest replica (geo-routing)
6. ✅ Route writes to primary only
7. ✅ Test failover scenarios regularly
8. ✅ Implement circuit breakers for failed replicas
9. ✅ Use connection pooling (PgBouncer)
10. ✅ Set up cross-region backups

Answer:

