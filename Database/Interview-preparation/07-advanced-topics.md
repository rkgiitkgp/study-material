## VII. Advanced Topics (System Design & Scaling)

---

### A. Distributed Databases

**1. Leader-follower replication vs multi-leader.**

**Leader-Follower (Master-Slave) Replication:**
A single leader (master) handles all writes, and followers (replicas) replicate data from the leader. Followers handle read requests only.

**How it works:**
- All writes go to the leader
- Leader writes changes to its transaction log
- Followers asynchronously replicate changes from leader
- Reads can be distributed across followers

**Architecture:**
```
Write Request → Leader (Primary) → Transaction Log
                              ↓
                        Replicates to
                              ↓
        Follower 1 ←  Follower 2 ←  Follower 3
        (Read Replica) (Read Replica) (Read Replica)
```

**Example: PostgreSQL Streaming Replication**
```sql
-- On Leader (Primary)
-- Configure for replication
ALTER SYSTEM SET wal_level = 'replica';
ALTER SYSTEM SET max_wal_senders = 3;
SELECT pg_reload_conf();

-- On Follower (Standby)
-- Create recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=leader.example.com port=5432 user=replicator'
```

**Multi-Leader (Master-Master) Replication:**
Multiple nodes can accept writes. Each leader processes writes independently and propagates changes to other leaders.

**How it works:**
- Multiple nodes accept writes
- Changes are replicated bidirectionally
- Conflict resolution needed when same data modified on different leaders
- Can have geographic distribution

**Architecture:**
```
Leader A (US)          Leader B (EU)          Leader C (Asia)
   ↕                       ↕                       ↕
   └─────────────── Bidirectional Replication ─────┘
```

**Example: Multi-Leader Setup**
```sql
-- Leader A (US Region)
INSERT INTO users (id, name, region) VALUES (1, 'John', 'US');
-- Replicates to Leader B and C

-- Leader B (EU Region) - can also write
INSERT INTO users (id, name, region) VALUES (2, 'Jane', 'EU');
-- Replicates to Leader A and C

-- Both can write simultaneously!
```

**Comparison Table:**

| Aspect | Leader-Follower | Multi-Leader |
|--------|----------------|--------------|
| Write Nodes | Single leader | Multiple leaders |
| Write Latency | Low (single node) | Varies (conflict resolution) |
| Read Scalability | High (many followers) | High (all nodes readable) |
| Conflict Resolution | Not needed | Required |
| Consistency | Strong (eventual) | Eventual (conflicts possible) |
| Failover | Manual/automatic promotion | More complex |
| Use Cases | Read-heavy workloads | Multi-region, offline support |
| Examples | PostgreSQL, MySQL | MySQL Cluster, CouchDB |

**When to Use Leader-Follower:**
- Read-heavy workloads (90% reads, 10% writes)
- Need read scaling without write complexity
- Can tolerate read lag
- Single region deployment

**Example Use Case: E-commerce Product Catalog**
```sql
-- Leader handles all product updates
UPDATE products SET price = 99.99 WHERE id = 123;  -- On leader

-- Multiple followers handle product reads
SELECT * FROM products WHERE category = 'electronics';  -- On follower 1
SELECT * FROM products WHERE category = 'books';        -- On follower 2
SELECT * FROM products WHERE category = 'clothing';     -- On follower 3
```

**When to Use Multi-Leader:**
- Multi-region deployment
- Need write availability in each region
- Can handle conflict resolution
- Offline-first applications

**Example Use Case: Global SaaS Application**
```python
# User in US writes to US leader
us_leader.execute("INSERT INTO notes (user_id, content) VALUES (1, 'Meeting notes')")

# User in EU writes to EU leader (simultaneously)
eu_leader.execute("INSERT INTO notes (user_id, content) VALUES (1, 'To-do list')")

# Both replicate to each other
# Conflict resolution needed if same note_id used
```

**Multi-Leader Conflict Resolution Strategies:**

**1. Last Write Wins (LWW)**
```sql
-- Uses timestamp
UPDATE users SET name = 'John', updated_at = NOW() WHERE id = 1;
-- Latest timestamp wins
```

**2. Custom Conflict Resolution**
```python
# Application-level conflict resolution
def resolve_conflict(conflict):
    if conflict.region == 'US':
        return conflict.us_value  # US takes precedence
    elif conflict.timestamp > conflict.us_timestamp:
        return conflict.eu_value
    else:
        return conflict.us_value
```

**3. Automatic Merge (for certain data types)**
```sql
-- Increment operations can merge
UPDATE counters SET value = value + 1 WHERE id = 1;
-- Both leaders increment: 10 + 1 + 1 = 12 (no conflict)
```

**Trade-offs:**

**Leader-Follower:**
- ✅ Simple to understand and operate
- ✅ Strong consistency (on leader)
- ✅ No conflict resolution needed
- ❌ Single point of failure for writes
- ❌ Write scaling limited to single node

**Multi-Leader:**
- ✅ Write availability (multiple regions)
- ✅ Lower write latency (local writes)
- ✅ Better fault tolerance
- ❌ Complex conflict resolution
- ❌ Eventual consistency challenges
- ❌ More complex operations

**Best Practices:**

**Leader-Follower:**
1. Monitor replication lag
2. Use connection pooling to route reads to followers
3. Implement automatic failover (e.g., Patroni, MHA)
4. Monitor follower health

**Multi-Leader:**
1. Design conflict-free data structures when possible
2. Implement robust conflict resolution
3. Use vector clocks or timestamps for ordering
4. Test conflict scenarios thoroughly
5. Consider partition tolerance

**2. Consensus algorithms (Raft, Paxos).**

Consensus algorithms ensure that multiple nodes in a distributed system agree on a single value or state, even in the presence of failures. They're fundamental for implementing distributed databases, leader election, and replicated state machines.

**Why Consensus is Needed:**
- Ensure all nodes agree on the same data
- Handle node failures gracefully
- Maintain consistency across replicas
- Elect a single leader when needed

**The Consensus Problem:**
```
Given N nodes, at least (N/2 + 1) nodes must agree for:
- Leader election
- Data replication
- Transaction commits
- Configuration changes
```

---

**Raft Algorithm:**

Raft is designed to be easier to understand than Paxos while providing equivalent fault tolerance. It decomposes consensus into three sub-problems:
1. **Leader Election**
2. **Log Replication**
3. **Safety**

**How Raft Works:**

**1. Leader Election:**
- Each node has a random timeout (150-300ms)
- When timeout expires, node becomes candidate
- Candidate requests votes from other nodes
- Node with majority votes becomes leader
- Leader sends periodic heartbeats to maintain leadership

**Example: Leader Election**
```
Initial State: All followers
Node 1 timeout expires → Becomes Candidate
Node 1: "RequestVote from Node 2, 3, 4, 5"
Node 2, 3, 4: Vote for Node 1 ✅
Node 5: Offline (no vote)
Result: Node 1 becomes Leader (3/5 votes = majority)
```

**2. Log Replication:**
- Leader receives client write request
- Leader appends entry to its log
- Leader replicates entry to followers
- When majority acknowledges → Entry is committed
- Leader applies entry to state machine
- Leader notifies followers to apply entry

**Example: Log Replication**
```
Client writes: "SET x = 5"

Leader (Node 1):
  1. Append (term=2, index=10, "SET x=5") to log
  2. Send AppendEntries to Followers

Followers (Node 2, 3, 4, 5):
  3. Receive AppendEntries
  4. Append to log
  5. Respond with success

Leader:
  6. Receives 3/5 acknowledgments (majority)
  7. Commit entry (index=10)
  8. Apply to state machine: x = 5
  9. Notify followers to commit

Result: All nodes have x = 5
```

**Raft Example: Distributed Key-Value Store**
```python
class RaftNode:
    def __init__(self, node_id, nodes):
        self.node_id = node_id
        self.nodes = nodes  # All node addresses
        self.state = 'follower'  # follower, candidate, leader
        self.current_term = 0
        self.voted_for = None
        self.log = []  # Log entries
        self.commit_index = 0
        
    def request_vote(self, candidate_id, term):
        """Vote for candidate if term is higher"""
        if term > self.current_term:
            self.current_term = term
            self.voted_for = candidate_id
            return True
        return False
    
    def append_entries(self, leader_id, term, entries):
        """Append log entries from leader"""
        if term >= self.current_term:
            self.current_term = term
            self.log.extend(entries)
            # Acknowledge to leader
            return True
        return False
    
    def replicate_to_followers(self, entry):
        """Leader replicates entry to followers"""
        if self.state != 'leader':
            return False
        
        self.log.append(entry)
        acknowledgments = 0
        
        for node in self.nodes:
            if node != self.node_id:
                # Send AppendEntries RPC
                if node.append_entries(self.node_id, self.current_term, [entry]):
                    acknowledgments += 1
        
        # Commit if majority acknowledged
        if acknowledgments >= len(self.nodes) / 2:
            self.commit_index = len(self.log) - 1
            return True
        return False
```

**Raft Properties:**
- **Safety:** Never commit incorrect values
- **Liveness:** System continues operating if majority nodes are up
- **Election Safety:** At most one leader per term
- **Log Matching:** If two logs contain entry with same index and term, they're identical

**When Leader Fails:**
```
Time    Node 1 (Leader)    Node 2         Node 3         Node 4         Node 5
----    -----------------  -------------  -------------  -------------  -------------
T1      Heartbeat          ✓              ✓              ✓              ✓
T2      CRASH!             (no heartbeat)
T3      (down)             Timeout        Timeout        Timeout        Timeout
T4      (down)             Candidate      Candidate      Candidate      Candidate
T5      (down)             RequestVote    RequestVote    RequestVote    RequestVote
T6      (down)             Leader!        Follower       Follower       Follower
                          (got votes
                           from 3,4,5)
```

---

**Paxos Algorithm:**

Paxos is the foundational consensus algorithm. It's more complex but theoretically elegant. It handles consensus in phases: Prepare and Accept.

**How Paxos Works:**

**1. Prepare Phase:**
- Proposer chooses proposal number (n)
- Sends Prepare(n) to majority of acceptors
- Acceptors promise not to accept proposals < n
- Acceptors return highest-numbered proposal they've accepted (if any)

**2. Accept Phase:**
- If proposer receives majority responses:
  - If no previous value: propose any value
  - If previous value exists: propose that value
- Sends Accept(n, value) to acceptors
- Acceptors accept if n >= highest promised number
- If majority accepts → Value is chosen

**Example: Paxos Round**
```
Proposer wants to set value = "v1"

Phase 1: Prepare
Proposer → Acceptors: Prepare(proposal_id=5)
Acceptor 1: Promise(5, no_previous_value)
Acceptor 2: Promise(5, no_previous_value)
Acceptor 3: Promise(5, no_previous_value)
→ Majority received, proceed

Phase 2: Accept
Proposer → Acceptors: Accept(5, "v1")
Acceptor 1: Accepted(5, "v1")
Acceptor 2: Accepted(5, "v1")
Acceptor 3: Accepted(5, "v1")
→ Majority accepted, value "v1" is chosen
```

**Paxos with Conflict:**
```
Scenario: Two proposers trying to set different values

Proposer A: Prepare(5) → Gets promises from 1,2,3
Proposer B: Prepare(6) → Gets promises from 3,4,5 (higher number!)

Proposer A: Accept(5, "vA") → Rejected by acceptor 3 (promised 6)
Proposer B: Check responses → No previous value, propose "vB"
Proposer B: Accept(6, "vB") → Accepted by 3,4,5

Result: "vB" is chosen (higher proposal number wins)
```

---

**Raft vs Paxos Comparison:**

| Aspect | Raft | Paxos |
|--------|------|-------|
| **Complexity** | Easier to understand | More complex |
| **Leader** | Strong leader model | Leaderless (can have multiple proposers) |
| **Implementation** | More straightforward | More theoretical |
| **Performance** | Similar | Similar |
| **Use Cases** | etcd, Consul, CockroachDB | Chubby, some databases |
| **Log Structure** | Sequential log | Independent proposals |
| **Consistency** | Strong | Strong |

**Raft Advantages:**
- ✅ Easier to understand and implement
- ✅ Strong leader simplifies operations
- ✅ Better for replicated logs
- ✅ Clear separation of concerns

**Paxos Advantages:**
- ✅ More flexible (no single leader required)
- ✅ Theoretically elegant
- ✅ Can handle multiple concurrent proposals

---

**Real-World Examples:**

**1. etcd (Uses Raft)**
```bash
# etcd uses Raft for distributed key-value store
# 3-node etcd cluster
etcd1 --name node1 --initial-cluster-state new --initial-cluster-token etcd-cluster
etcd2 --name node2 --initial-cluster-state new --initial-cluster-token etcd-cluster
etcd3 --name node3 --initial-cluster-state new --initial-cluster-token etcd-cluster

# Write operation
etcdctl put mykey "myvalue"
# Raft ensures all 3 nodes agree on this value
```

**2. PostgreSQL High Availability (Patroni uses Raft)**
```yaml
# Patroni uses Raft (via etcd/Consul) for leader election
postgresql:
  parameters:
    wal_level: replica
    max_wal_senders: 3
    
# When primary fails, Raft elects new leader
# New leader promotes standby to primary
```

**3. Distributed Transaction Coordinator**
```python
# Two-phase commit with Raft
class DistributedTransaction:
    def commit(self, transaction_id, operations):
        # Phase 1: Prepare (using Raft)
        prepare_votes = []
        for node in self.nodes:
            vote = node.prepare(transaction_id, operations)
            prepare_votes.append(vote)
        
        # If majority votes yes, commit
        if sum(prepare_votes) >= len(self.nodes) / 2:
            # Phase 2: Commit (using Raft)
            for node in self.nodes:
                node.commit(transaction_id)
            return True
        else:
            # Abort
            for node in self.nodes:
                node.abort(transaction_id)
            return False
```

---

**Fault Tolerance:**

**Raft:**
- Tolerates (N-1)/2 node failures
- 5 nodes → Can lose 2 nodes
- 3 nodes → Can lose 1 node

**Example:**
```
5-node cluster:
- 3 nodes up: System works ✅
- 2 nodes down: System works ✅
- 3 nodes down: System stops ❌ (no majority)
```

**Paxos:**
- Same fault tolerance: (N-1)/2 failures
- Requires majority for consensus

---

**Best Practices:**

1. **Cluster Size:**
   - Use odd number of nodes (3, 5, 7)
   - Avoid even numbers (4, 6) - same fault tolerance as smaller odd number

2. **Network Partition Handling:**
   - Only majority partition can make progress
   - Minority partition blocks (prevents split-brain)

3. **Performance Optimization:**
   - Batch multiple operations
   - Use leader for all writes
   - Replicate asynchronously after commit

4. **Monitoring:**
   - Track leader elections
   - Monitor replication lag
   - Alert on partition scenarios

**Example: Monitoring Raft Health**
```python
def check_raft_health(raft_cluster):
    stats = {
        'leader': None,
        'followers': [],
        'term': 0,
        'commit_index': 0,
        'replication_lag': {}
    }
    
    for node in raft_cluster:
        if node.is_leader():
            stats['leader'] = node.id
            stats['term'] = node.current_term
            stats['commit_index'] = node.commit_index
            
            # Check replication lag
            for follower in node.followers:
                lag = node.commit_index - follower.match_index
                stats['replication_lag'][follower.id] = lag
    
    return stats
```

**3. Eventual consistency vs strong consistency.**

Consistency models define when updates become visible to readers in a distributed system. The choice between eventual and strong consistency is a fundamental trade-off in distributed database design.

---

**Strong Consistency:**

All reads see the most recent write. The system guarantees that once a write completes, all subsequent reads (even from different nodes) will see that write.

**Characteristics:**
- Linearizability: Operations appear to occur atomically
- All nodes see same data at same time
- Writes block until replicated to all nodes
- Higher latency for writes
- Lower availability during network partitions

**How it works:**
```
Write: x = 5
1. Write to primary node
2. Wait for all replicas to acknowledge
3. Return success to client

Read: x
1. Can read from any node
2. Guaranteed to see x = 5 (latest value)
```

**Example: Strong Consistency in PostgreSQL**
```sql
-- PostgreSQL with synchronous replication
-- postgresql.conf
synchronous_standby_names = 'standby1,standby2'

-- Write operation
BEGIN;
UPDATE accounts SET balance = 1000 WHERE id = 1;
COMMIT;  -- Waits for standby1 AND standby2 to confirm
-- Only returns after all replicas have the update

-- Read from any replica
SELECT balance FROM accounts WHERE id = 1;
-- Guaranteed to see balance = 1000 (latest value)
```

**Strong Consistency Scenario:**
```
Time    Client A (Write)         Node 1 (Primary)      Node 2 (Replica)      Node 3 (Replica)      Client B (Read)
----    --------------------     -------------------   -------------------   -------------------   ----------------
T1      UPDATE x = 5             x = 5 (writing)       (waiting)             (waiting)             
T2      (waiting)                x = 5 (committed)     x = 5 (syncing)       x = 5 (syncing)       
T3      (waiting)                x = 5                 x = 5 (synced)        x = 5 (syncing)       
T4      Success! ✅              x = 5                 x = 5                 x = 5 (synced)        SELECT x
T5                                                                                                Returns: 5 ✅
                                                                                                (Always sees latest)
```

**Use Cases for Strong Consistency:**
- Financial transactions (banking)
- Inventory management (prevent overselling)
- Medical records
- Legal documents
- Any system where stale data causes problems

**Example: E-commerce Inventory**
```sql
-- Strong consistency prevents overselling
-- Product has 1 item left

Transaction 1:
BEGIN;
SELECT stock FROM products WHERE id = 123;  -- Returns 1
UPDATE products SET stock = stock - 1 WHERE id = 123;  -- stock = 0
COMMIT;  -- Waits for all replicas

Transaction 2 (concurrent):
BEGIN;
SELECT stock FROM products WHERE id = 123;  -- Returns 0 (sees Transaction 1's update)
-- Cannot purchase - out of stock ✅
COMMIT;
```

---

**Eventual Consistency:**

The system guarantees that if no new updates are made, eventually all reads will return the same value. Replicas may temporarily have different values, but they'll converge over time.

**Characteristics:**
- Replicas may have different values temporarily
- Reads may return stale data
- Writes return immediately (don't wait for replication)
- Lower latency for writes
- Higher availability (can serve reads during partitions)
- Convergence happens "eventually"

**How it works:**
```
Write: x = 5
1. Write to primary node
2. Return success immediately (don't wait for replicas)
3. Replicate asynchronously in background

Read: x
1. Read from nearby replica
2. Might see x = 3 (old value) if replica hasn't updated yet
3. Eventually will see x = 5
```

**Example: Eventual Consistency in Cassandra**
```python
# Cassandra with eventual consistency
from cassandra.cluster import Cluster

cluster = Cluster(['node1', 'node2', 'node3'])
session = cluster.connect('mykeyspace')

# Write with QUORUM consistency (local quorum)
session.execute(
    "UPDATE users SET email = 'new@example.com' WHERE id = 1",
    consistency_level=ConsistencyLevel.QUORUM  # Local quorum, not all nodes
)
# Returns immediately, replicates asynchronously

# Read might see old value from stale replica
result = session.execute(
    "SELECT email FROM users WHERE id = 1",
    consistency_level=ConsistencyLevel.ONE  # Read from one node (might be stale)
)
# Could return old email temporarily
```

**Eventual Consistency Scenario:**
```
Time    Client A (Write)         Node 1 (Primary)      Node 2 (Replica)      Node 3 (Replica)      Client B (Read)
----    --------------------     -------------------   -------------------   -------------------   ----------------
T1      UPDATE x = 5             x = 5 (committed)     x = 3 (old)           x = 3 (old)           
T2      Success! ✅              x = 5                 x = 3 (replicating)   x = 3 (old)           SELECT x (Node 2)
T3                              x = 5                 x = 5 (updated)        x = 3 (replicating)   Returns: 3 ❌
                                                                                                   (Stale value!)
T4                              x = 5                 x = 5                 x = 5 (updated)       SELECT x (Node 3)
                                                                                                   Returns: 5 ✅
                                                                                                   (Eventually consistent)
```

**Use Cases for Eventual Consistency:**
- Social media feeds (likes, comments)
- DNS (domain name resolution)
- CDN content updates
- Analytics and reporting
- High-traffic web applications
- Systems where stale data is acceptable

**Example: Social Media Like Counter**
```python
# Eventual consistency acceptable for likes
def like_post(post_id, user_id):
    # Write to nearest node
    cassandra.execute(
        f"UPDATE posts SET likes = likes + 1 WHERE id = {post_id}"
    )
    # Returns immediately
    
    # Replicate to other nodes asynchronously
    # User sees update immediately
    # Other users might see old count temporarily

# User A likes post → sees 101 likes
# User B (different region) might see 100 likes temporarily
# Eventually both see 101 likes ✅
```

---

**Comparison Table:**

| Aspect | Strong Consistency | Eventual Consistency |
|--------|-------------------|---------------------|
| **Read Guarantee** | Always latest value | May see stale value |
| **Write Latency** | High (waits for all replicas) | Low (returns immediately) |
| **Availability** | Lower (blocks during partitions) | Higher (serves reads during partitions) |
| **Complexity** | Simpler (no conflict resolution) | More complex (conflict resolution needed) |
| **Use Cases** | Financial, inventory | Social media, analytics |
| **Network Partition** | Stops accepting writes | Continues operating |
| **Examples** | PostgreSQL (sync), Spanner | Cassandra, DynamoDB, MongoDB |

---

**Consistency Levels (Spectrum):**

**1. Strong Consistency (Linearizability)**
```
Write → All replicas updated → Read sees latest
Latency: High
Availability: Low
```

**2. Sequential Consistency**
```
All nodes see operations in same order
But operations may not be immediately visible
```

**3. Causal Consistency**
```
Causally related operations seen in order
Unrelated operations can be seen in different order
```

**4. Eventual Consistency**
```
Eventually all replicas converge
No guarantee when convergence happens
```

**Example: Consistency Levels in DynamoDB**
```python
import boto3

dynamodb = boto3.client('dynamodb')

# Strong consistency read
response = dynamodb.get_item(
    TableName='users',
    Key={'id': {'S': '123'}},
    ConsistentRead=True  # Strong consistency
)
# Waits for all replicas, sees latest value

# Eventual consistency read (default)
response = dynamodb.get_item(
    TableName='users',
    Key={'id': {'S': '123'}},
    ConsistentRead=False  # Eventual consistency
)
# Reads from one replica, might be stale
```

---

**CAP Theorem Context:**

**CAP Theorem:** In a distributed system, you can only guarantee 2 out of 3:
- **C**onsistency (all nodes see same data)
- **A**vailability (system responds)
- **P**artition tolerance (works during network failures)

**Strong Consistency:**
- Chooses **CP** (Consistency + Partition Tolerance)
- Sacrifices Availability during partitions
- Example: PostgreSQL with sync replication

**Eventual Consistency:**
- Chooses **AP** (Availability + Partition Tolerance)
- Sacrifices Consistency (temporarily)
- Example: Cassandra, DynamoDB

**Example: Network Partition**
```
Network splits cluster:
- Node 1, 2 in Partition A
- Node 3, 4 in Partition B

Strong Consistency (CP):
- Partition A: 2/4 nodes (minority) → Stops accepting writes ❌
- Partition B: 2/4 nodes (minority) → Stops accepting writes ❌
- System unavailable, but maintains consistency

Eventual Consistency (AP):
- Partition A: Continues accepting writes ✅
- Partition B: Continues accepting writes ✅
- System available, but data diverges temporarily
- When partition heals, data converges
```

---

**Hybrid Approaches:**

**1. Tunable Consistency**
```python
# Cassandra allows choosing consistency level per operation
session.execute(
    "UPDATE users SET email = 'new@example.com' WHERE id = 1",
    consistency_level=ConsistencyLevel.QUORUM  # Stronger
)

session.execute(
    "SELECT email FROM users WHERE id = 1",
    consistency_level=ConsistencyLevel.ONE  # Weaker (faster)
)
```

**2. Read-Your-Writes Consistency**
```python
# User always sees their own writes
def update_profile(user_id, data):
    # Write to primary
    db.update(user_id, data)
    
    # Subsequent reads from same user
    # Always read from primary (not replica)
    return db.read_from_primary(user_id)  # Sees own write
```

**3. Session Consistency**
```python
# All reads in same session see consistent view
session = db.create_session()

# First read
result1 = session.read(key)  # x = 5

# Write
session.write(key, 10)  # x = 10

# Second read (same session)
result2 = session.read(key)  # x = 10 (sees own write)
```

---

**Real-World Examples:**

**1. Strong Consistency: Banking System**
```sql
-- Transfer $100 from Account A to Account B
BEGIN TRANSACTION;

-- Check balance (strong consistency)
SELECT balance FROM accounts WHERE id = 'A';  -- Returns 500

-- Deduct from A
UPDATE accounts SET balance = balance - 100 WHERE id = 'A';

-- Add to B
UPDATE accounts SET balance = balance + 100 WHERE id = 'B';

COMMIT;  -- Waits for all replicas

-- Any subsequent read sees updated balances ✅
SELECT balance FROM accounts WHERE id = 'A';  -- Returns 400 (any replica)
SELECT balance FROM accounts WHERE id = 'B';  -- Returns 600 (any replica)
```

**2. Eventual Consistency: Social Media Feed**
```python
# User posts update
post_id = create_post(user_id, "Hello world!")

# Update appears immediately in user's own feed
user_feed = get_feed(user_id)  # Sees new post ✅

# Other users' feeds might not show it yet
friend_feed = get_feed(friend_id)  # Might not see post yet
# Eventually friend's feed updates ✅
```

**3. Eventual Consistency: View Count**
```python
# Video view count
def increment_views(video_id):
    # Write to nearest node
    cassandra.execute(
        f"UPDATE videos SET views = views + 1 WHERE id = {video_id}"
    )
    # Returns immediately
    
# User sees view count: 1,000,000
# Actual count might be 1,000,050 (being replicated)
# Eventually converges ✅
```

---

**Best Practices:**

**Choose Strong Consistency When:**
- ✅ Data correctness is critical
- ✅ Stale data causes problems
- ✅ Low write volume
- ✅ Can tolerate higher latency
- ✅ Financial, legal, medical data

**Choose Eventual Consistency When:**
- ✅ High write volume
- ✅ Global distribution needed
- ✅ Stale data acceptable
- ✅ Need high availability
- ✅ Social media, analytics, logs

**Hybrid Strategy:**
```python
# Use strong consistency for critical operations
def transfer_money(from_account, to_account, amount):
    # Strong consistency
    with strong_consistency():
        debit(from_account, amount)
        credit(to_account, amount)

# Use eventual consistency for non-critical operations
def update_user_profile(user_id, profile_data):
    # Eventual consistency
    with eventual_consistency():
        update_profile(user_id, profile_data)
        # OK if other users see old profile temporarily
```

---

**Monitoring Consistency:**

```python
def check_replication_lag():
    """Monitor how far behind replicas are"""
    primary_commit = get_primary_commit_index()
    
    for replica in replicas:
        replica_commit = get_replica_commit_index(replica)
        lag = primary_commit - replica_commit
        
        if lag > threshold:
            alert(f"Replica {replica} lagging: {lag} commits")
        
        # Strong consistency: lag should be 0
        # Eventual consistency: lag acceptable up to threshold
```

**Summary:**
- **Strong Consistency:** All reads see latest write, higher latency, lower availability
- **Eventual Consistency:** Reads may be stale, lower latency, higher availability
- Choose based on your application's requirements and tolerance for stale data

**4. How does database handle network partition?**

A network partition occurs when a distributed database cluster splits into multiple disconnected groups due to network failures. Databases handle partitions differently based on their consistency model and design philosophy.

**What is a Network Partition?**
```
Before Partition:
Node 1 ←→ Node 2 ←→ Node 3 ←→ Node 4 ←→ Node 5
(All connected)

After Partition:
Partition A:        Partition B:
Node 1 ←→ Node 2    Node 3 ←→ Node 4 ←→ Node 5
(Can't communicate)
```

---

**Handling Strategies:**

### 1. **Quorum-Based Approach (CAP Theorem - CP or AP)**

**CP Systems (Consistency + Partition Tolerance):**
- Stop accepting writes when partition occurs
- Only majority partition can operate
- Prevents split-brain (two nodes thinking they're leader)

**Example: Raft-Based Systems (etcd, Consul)**
```
5-node cluster splits:
- Partition A: 2 nodes (minority)
- Partition B: 3 nodes (majority)

Partition A:
- Cannot elect leader (needs 3 votes)
- Stops accepting writes ❌
- Can serve reads (stale data)

Partition B:
- Can elect leader (has 3 nodes = majority)
- Continues accepting writes ✅
- Maintains consistency ✅
```

**Example: PostgreSQL with Patroni**
```yaml
# Patroni configuration
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576

# Network partition scenario:
# 3-node cluster: primary (Node 1), standby (Node 2, 3)

# Partition splits: Node 1 | Node 2, 3
# Node 1: Loses quorum → Stops accepting writes (prevents split-brain)
# Node 2, 3: Has quorum → Elects new primary, continues serving
```

**AP Systems (Availability + Partition Tolerance):**
- Continue accepting writes in all partitions
- Resolve conflicts when partition heals
- May have temporary inconsistency

**Example: Cassandra**
```python
# Cassandra continues operating during partition
# Uses quorum for writes (not all nodes)

# 5-node cluster splits:
# Partition A: 2 nodes
# Partition B: 3 nodes

# Write with QUORUM (needs 3 nodes)
session.execute(
    "UPDATE users SET email = 'new@example.com' WHERE id = 1",
    consistency_level=ConsistencyLevel.QUORUM  # Needs 3 nodes
)

# Partition A: Cannot achieve QUORUM → Write fails ❌
# Partition B: Can achieve QUORUM → Write succeeds ✅

# After partition heals:
# - Data from Partition B replicates to Partition A
# - Conflict resolution handles any diverged data
```

---

### 2. **Leader-Follower Systems**

**Synchronous Replication:**
```sql
-- PostgreSQL synchronous replication
synchronous_standby_names = 'standby1,standby2'

-- During partition:
-- Primary can only commit if all sync standbys acknowledge
-- If partition isolates primary from standbys:
--   Primary blocks writes (waits for acknowledgments)
--   Writes timeout or fail
```

**Asynchronous Replication:**
```sql
-- PostgreSQL asynchronous replication
synchronous_standby_names = ''  -- No sync standbys

-- During partition:
-- Primary continues accepting writes ✅
-- Standbys lag behind (no replication)
-- When partition heals, catch-up replication occurs
-- Risk: Data loss if primary fails before replication
```

**Example: Automatic Failover**
```python
# Patroni handles partition with automatic failover
# Scenario: 3-node cluster, network partition

# Original state:
# Node 1: Primary
# Node 2: Standby
# Node 3: Standby

# Partition occurs:
# Partition A: Node 1 (isolated)
# Partition B: Node 2, 3 (connected)

# Patroni in Partition B:
# - Detects primary (Node 1) is unreachable
# - Waits for TTL timeout (30 seconds)
# - Elects Node 2 as new primary
# - Node 3 becomes standby of Node 2

# Result:
# - Partition A: Node 1 thinks it's primary (split-brain prevented by fencing)
# - Partition B: Node 2 is primary, serving traffic ✅
```

---

### 3. **Split-Brain Prevention**

**Problem:** Two nodes both think they're primary
```
Partition A: Node 1 (thinks it's primary)
Partition B: Node 2 (thinks it's primary)

Both accept writes → Data divergence!
```

**Solutions:**

**1. Fencing (STONITH - Shoot The Other Node In The Head)**
```python
# When partition occurs, majority partition "fences" minority
def handle_partition(partition_nodes):
    if len(partition_nodes) < quorum:
        # Minority partition
        # Fence ourselves (stop accepting writes)
        self.stop_accepting_writes()
        
        # Or: Majority partition fences minority nodes
        for node in minority_partition:
            fence_node(node)  # Power off or isolate
```

**2. Quorum Requirements**
```python
# Require majority for leader election
def elect_leader(nodes):
    votes_needed = len(nodes) // 2 + 1
    
    if len(current_partition) < votes_needed:
        # Cannot elect leader
        return None
    
    # Can elect leader
    return elect_from_partition(current_partition)
```

**3. Witness/Arbiter Node**
```
5-node cluster + 1 witness node (in different datacenter)
Witness doesn't store data, only votes

Partition A: 2 nodes + witness → 3 votes → Can operate
Partition B: 3 nodes → 3 votes → Can operate
(But witness is in third location, breaks tie)
```

---

### 4. **Conflict Resolution (AP Systems)**

**Last Write Wins (LWW)**
```python
# DynamoDB uses timestamps
def resolve_conflict(key, values):
    # Choose value with latest timestamp
    latest = max(values, key=lambda v: v.timestamp)
    return latest
```

**Vector Clocks**
```python
# Track causal relationships
class VectorClock:
    def __init__(self):
        self.clocks = {}  # {node_id: timestamp}
    
    def happens_before(self, other):
        # Check if this event happened before other
        for node in self.clocks:
            if self.clocks[node] > other.clocks.get(node, 0):
                return False
        return True

# During partition merge:
def merge_conflicts(conflicts):
    for conflict in conflicts:
        if conflict.vclock.happens_before(conflict.other.vclock):
            # Use other value
            return conflict.other.value
        else:
            # Use this value
            return conflict.value
```

**Application-Level Resolution**
```python
# Custom conflict resolution
def resolve_user_profile_conflict(partition_a_data, partition_b_data):
    # Merge non-conflicting fields
    merged = {}
    
    # For conflicting fields, use merge strategy
    if partition_a_data['last_updated'] > partition_b_data['last_updated']:
        merged.update(partition_a_data)
    else:
        merged.update(partition_b_data)
    
    # Or: Present both versions to user
    return {
        'merged': merged,
        'conflicts': find_conflicts(partition_a_data, partition_b_data)
    }
```

---

### 5. **Partition Detection and Recovery**

**Detection:**
```python
# Heartbeat mechanism
class PartitionDetector:
    def __init__(self):
        self.heartbeat_interval = 1  # seconds
        self.timeout = 5  # seconds
    
    def check_connectivity(self):
        for node in self.cluster_nodes:
            last_seen = self.get_last_heartbeat(node)
            if time.now() - last_seen > self.timeout:
                # Node unreachable
                self.mark_partitioned(node)
```

**Recovery:**
```python
# When partition heals
def recover_from_partition():
    # 1. Detect partition is healed
    if all_nodes_reachable():
        # 2. Determine which partition had writes
        primary_partition = get_partition_with_primary()
        
        # 3. Replicate missed data
        for node in other_partition:
            catch_up_replication(node)
        
        # 4. Resolve conflicts
        conflicts = detect_conflicts()
        for conflict in conflicts:
            resolve_conflict(conflict)
        
        # 5. Resume normal operation
        resume_normal_operation()
```

---

**Real-World Examples:**

**1. PostgreSQL with Patroni (CP)**
```yaml
# Patroni configuration
bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    retry_timeout: 30

# Partition handling:
# - Uses etcd/Consul for leader election (Raft-based)
# - Only majority partition can elect leader
# - Minority partition stops accepting writes
# - Automatic failover when partition detected
```

**2. MongoDB Replica Set (CP)**
```javascript
// MongoDB replica set (3 nodes)
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "node1:27017" },
    { _id: 1, host: "node2:27017" },
    { _id: 2, host: "node3:27017" }
  ]
})

// Partition scenario:
// Partition A: node1 (primary)
// Partition B: node2, node3

// node1: Loses majority → Steps down as primary
// node2, node3: Have majority → Elect new primary
// Result: Only one primary (in majority partition)
```

**3. Cassandra (AP)**
```python
# Cassandra with NetworkTopologyStrategy
CREATE KEYSPACE mykeyspace
WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3,
  'datacenter2': 3
}

# Partition scenario:
# Datacenter1 isolated from Datacenter2

# Writes with LOCAL_QUORUM:
# - Each datacenter can write independently
# - Data replicates within datacenter
# - When partition heals, cross-datacenter replication resumes
# - Conflict resolution via timestamps (LWW)
```

**4. DynamoDB (AP)**
```python
# DynamoDB global tables (multi-region)
# Partition scenario: US-East and EU-West isolated

# Both regions continue accepting writes ✅
# Each region replicates within itself
# When partition heals:
# - Cross-region replication resumes
# - Conflicts resolved by last-write-wins (timestamp)
```

---

**Best Practices:**

**1. Design for Partition Tolerance**
```python
# Always assume partitions can occur
def write_data(key, value):
    try:
        # Try to achieve quorum
        success = write_with_quorum(key, value)
        if not success:
            # Partition detected
            handle_partition_scenario(key, value)
    except PartitionException:
        # Handle gracefully
        log_partition_event()
        # Option: Queue for later, or fail fast
```

**2. Monitor Partition Events**
```python
# Track partition frequency and duration
def monitor_partitions():
    partition_events = get_partition_events()
    
    metrics = {
        'partition_count': len(partition_events),
        'avg_duration': calculate_avg_duration(partition_events),
        'data_loss_risk': calculate_data_loss_risk()
    }
    
    if metrics['partition_count'] > threshold:
        alert("High partition frequency detected")
```

**3. Test Partition Scenarios**
```python
# Chaos engineering: Test partition handling
def test_network_partition():
    # Simulate partition
    firewall.block_communication(node1, node2)
    
    # Verify behavior
    assert node1.stops_accepting_writes()  # If CP system
    assert node2.can_elect_leader()  # If has majority
    
    # Heal partition
    firewall.allow_communication(node1, node2)
    
    # Verify recovery
    assert data_replicated_correctly()
    assert conflicts_resolved()
```

**4. Choose Appropriate Consistency Level**
```python
# Balance consistency vs availability
def configure_consistency():
    # Critical data: Strong consistency (CP)
    critical_table.consistency = ConsistencyLevel.ALL
    
    # Non-critical data: Eventual consistency (AP)
    analytics_table.consistency = ConsistencyLevel.ONE
```

---

**Summary:**

**CP Systems (PostgreSQL, MongoDB):**
- Stop writes in minority partition
- Only majority partition operates
- Prevents split-brain
- May have downtime during partitions

**AP Systems (Cassandra, DynamoDB):**
- Continue operating in all partitions
- Resolve conflicts when partition heals
- Higher availability
- May have temporary inconsistency

**Key Strategies:**
1. Quorum-based operations
2. Leader election with majority requirement
3. Fencing to prevent split-brain
4. Conflict resolution for AP systems
5. Monitoring and automatic recovery

---

### B. Architecture Trade-offs

**1. When to denormalize data intentionally.**

Denormalization is the intentional introduction of redundancy into a database design to improve read performance, reduce JOIN operations, and optimize for specific query patterns. While normalization reduces redundancy, denormalization trades storage space and write complexity for faster reads.

**When to Denormalize:**

### 1. **Read-Heavy Workloads**

When reads significantly outnumber writes, denormalization can dramatically improve performance.

**Example: E-commerce Product Catalog**
```sql
-- Normalized (requires JOIN)
SELECT 
    p.product_id,
    p.name,
    p.price,
    c.category_name,
    b.brand_name
FROM products p
JOIN categories c ON p.category_id = c.category_id
JOIN brands b ON p.brand_id = b.brand_id
WHERE p.category_id = 5;
-- Requires 2 JOINs, slower for high-traffic reads

-- Denormalized (no JOINs)
SELECT 
    product_id,
    name,
    price,
    category_name,  -- Stored directly in products table
    brand_name      -- Stored directly in products table
FROM products
WHERE category_id = 5;
-- Single table scan, much faster ⚡
```

**Performance Impact:**
```
Normalized:  50ms (2 JOINs)
Denormalized: 5ms (single table)
Improvement: 10x faster
```

### 2. **Analytics and Reporting Systems**

OLAP systems benefit from denormalized star/snowflake schemas for complex aggregations.

**Example: Sales Analytics**
```sql
-- Normalized (fact table + dimension tables)
SELECT 
    d.year,
    d.month,
    l.region,
    p.category,
    SUM(f.sales_amount) as total_sales
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_location l ON f.location_id = l.location_id
JOIN dim_product p ON f.product_id = p.product_id
GROUP BY d.year, d.month, l.region, p.category;
-- Multiple JOINs slow down aggregation

-- Denormalized (star schema with pre-joined data)
SELECT 
    year,
    month,
    region,
    category,
    SUM(sales_amount) as total_sales
FROM sales_denormalized
GROUP BY year, month, region, category;
-- Single table, faster aggregation ⚡
```

**Star Schema Design:**
```sql
-- Fact table with denormalized dimensions
CREATE TABLE sales_fact (
    sale_id SERIAL PRIMARY KEY,
    sale_date DATE,
    year INT,              -- Denormalized from date
    month INT,              -- Denormalized from date
    quarter INT,            -- Denormalized from date
    customer_id INT,
    customer_name VARCHAR,  -- Denormalized from customers
    customer_region VARCHAR,-- Denormalized from customers
    product_id INT,
    product_name VARCHAR,   -- Denormalized from products
    product_category VARCHAR,-- Denormalized from products
    location_id INT,
    region VARCHAR,         -- Denormalized from locations
    country VARCHAR,        -- Denormalized from locations
    sales_amount DECIMAL,
    quantity INT
);

-- Much faster for analytics queries
```

### 3. **High JOIN Costs**

When JOINs are expensive due to large tables or complex relationships.

**Example: User Activity Feed**
```sql
-- Normalized (expensive JOINs)
SELECT 
    a.activity_id,
    a.activity_type,
    a.timestamp,
    u.username,
    u.avatar_url,
    p.post_title,
    p.post_image_url
FROM activities a
JOIN users u ON a.user_id = u.user_id
LEFT JOIN posts p ON a.post_id = p.post_id
WHERE a.user_id = 123
ORDER BY a.timestamp DESC
LIMIT 20;
-- JOINs across large tables, slow

-- Denormalized (embedded data)
SELECT 
    activity_id,
    activity_type,
    timestamp,
    username,        -- Denormalized
    avatar_url,     -- Denormalized
    post_title,     -- Denormalized
    post_image_url  -- Denormalized
FROM activities_denormalized
WHERE user_id = 123
ORDER BY timestamp DESC
LIMIT 20;
-- Single table query, much faster ⚡
```

### 4. **Geographic Distribution**

For globally distributed systems where JOINs across regions are expensive.

**Example: Multi-Region Application**
```python
# Normalized: Requires cross-region JOIN
# User in US, data in EU → Slow cross-region query

# Denormalized: All data in one region
def get_user_profile(user_id):
    # All data co-located, fast read
    return db.query("""
        SELECT 
            user_id,
            username,
            email,
            profile_picture_url,  -- Denormalized
            bio,                  -- Denormalized
            follower_count,       -- Denormalized (updated via triggers)
            following_count       -- Denormalized (updated via triggers)
        FROM users_denormalized
        WHERE user_id = %s
    """, user_id)
```

### 5. **Materialized Views**

Pre-computed aggregations stored as denormalized tables.

**Example: Dashboard Metrics**
```sql
-- Expensive query run frequently
SELECT 
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as user_count,
    COUNT(DISTINCT country) as country_count,
    AVG(age) as avg_age
FROM users
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY date;

-- Denormalized materialized view
CREATE MATERIALIZED VIEW daily_user_stats AS
SELECT 
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as user_count,
    COUNT(DISTINCT country) as country_count,
    AVG(age) as avg_age
FROM users
GROUP BY date;

-- Fast read from materialized view
SELECT * FROM daily_user_stats
WHERE date >= CURRENT_DATE - INTERVAL '30 days';
-- Pre-computed, instant results ⚡

-- Refresh periodically
REFRESH MATERIALIZED VIEW daily_user_stats;
```

### 6. **Time-Series Data**

Denormalize metadata into time-series records for faster queries.

**Example: IoT Sensor Data**
```sql
-- Normalized (requires JOIN for metadata)
SELECT 
    m.timestamp,
    m.value,
    s.sensor_name,
    s.location,
    s.sensor_type
FROM measurements m
JOIN sensors s ON m.sensor_id = s.sensor_id
WHERE s.location = 'Building A'
  AND m.timestamp >= '2024-01-01';

-- Denormalized (metadata in each row)
SELECT 
    timestamp,
    value,
    sensor_name,    -- Denormalized
    location,       -- Denormalized
    sensor_type     -- Denormalized
FROM measurements_denormalized
WHERE location = 'Building A'
  AND timestamp >= '2024-01-01';
-- Faster, especially with time-series indexes
```

### 7. **Real-Time Applications**

When sub-second response times are critical.

**Example: Gaming Leaderboard**
```sql
-- Normalized (slow for real-time)
SELECT 
    u.user_id,
    u.username,
    u.avatar_url,
    s.score,
    s.rank
FROM scores s
JOIN users u ON s.user_id = u.user_id
ORDER BY s.score DESC
LIMIT 100;
-- JOIN adds latency

-- Denormalized (fast for real-time)
SELECT 
    user_id,
    username,      -- Denormalized
    avatar_url,   -- Denormalized
    score,
    rank
FROM leaderboard_denormalized
ORDER BY score DESC
LIMIT 100;
-- Single table, instant results ⚡
```

---

**Denormalization Patterns:**

### 1. **Copy Columns (One-to-Many)**
```sql
-- Copy frequently accessed columns from parent table
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR,      -- Denormalized from customers
    customer_email VARCHAR,     -- Denormalized from customers
    order_date DATE,
    total_amount DECIMAL
);

-- Benefit: No JOIN needed for customer info
-- Trade-off: If customer name changes, update all orders
```

### 2. **Pre-computed Aggregates**
```sql
-- Store aggregated values instead of computing on-the-fly
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR,
    price DECIMAL,
    total_orders INT,           -- Denormalized aggregate
    total_revenue DECIMAL,      -- Denormalized aggregate
    avg_rating DECIMAL          -- Denormalized aggregate
);

-- Updated via triggers or application logic
CREATE TRIGGER update_product_stats
AFTER INSERT ON order_items
FOR EACH ROW
EXECUTE FUNCTION update_product_aggregates();
```

### 3. **Flattened Hierarchies**
```sql
-- Flatten category hierarchy
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    name VARCHAR,
    category_id INT,
    category_name VARCHAR,      -- Denormalized
    parent_category_id INT,
    parent_category_name VARCHAR -- Denormalized
);

-- Benefit: No recursive query needed
-- Query: SELECT * FROM products WHERE parent_category_name = 'Electronics'
```

### 4. **Duplicate Data Across Tables**
```sql
-- Duplicate data for different access patterns
CREATE TABLE user_posts (
    post_id SERIAL PRIMARY KEY,
    user_id INT,
    username VARCHAR,           -- Denormalized
    content TEXT,
    created_at TIMESTAMP
);

CREATE TABLE user_comments (
    comment_id SERIAL PRIMARY KEY,
    post_id INT,
    user_id INT,
    username VARCHAR,           -- Denormalized (duplicate)
    content TEXT,
    created_at TIMESTAMP
);

-- Benefit: Fast queries without JOINs
-- Trade-off: Username stored in multiple places
```

---

**Trade-offs:**

| Aspect | Normalized | Denormalized |
|--------|-----------|--------------|
| **Read Performance** | Slower (JOINs) | Faster (no JOINs) |
| **Write Performance** | Faster | Slower (update multiple places) |
| **Storage** | Less space | More space (redundancy) |
| **Data Integrity** | Easier (single source) | Harder (sync multiple places) |
| **Query Complexity** | More complex (JOINs) | Simpler (single table) |
| **Maintenance** | Easier | Harder (keep in sync) |

---

**Best Practices:**

### 1. **Identify Hot Paths**
```python
# Analyze query patterns
def analyze_query_patterns():
    slow_queries = get_slow_queries(min_duration=100)  # > 100ms
    
    for query in slow_queries:
        if query.has_multiple_joins():
            # Candidate for denormalization
            analyze_denormalization_benefit(query)
```

### 2. **Use Triggers for Consistency**
```sql
-- Keep denormalized data in sync
CREATE OR REPLACE FUNCTION sync_customer_name()
RETURNS TRIGGER AS $$
BEGIN
    -- Update denormalized customer_name in orders
    UPDATE orders
    SET customer_name = NEW.name,
        customer_email = NEW.email
    WHERE customer_id = NEW.customer_id;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER sync_customer_to_orders
AFTER UPDATE ON customers
FOR EACH ROW
EXECUTE FUNCTION sync_customer_name();
```

### 3. **Gradual Denormalization**
```sql
-- Start normalized, denormalize as needed
-- Step 1: Monitor query performance
-- Step 2: Identify slow queries with JOINs
-- Step 3: Denormalize specific columns
-- Step 4: Measure improvement
-- Step 5: Adjust as needed
```

### 4. **Document Denormalization Decisions**
```sql
-- Add comments explaining denormalization
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR,  -- DENORMALIZED: Updated via trigger from customers.name
    customer_email VARCHAR   -- DENORMALIZED: Updated via trigger from customers.email
);
```

### 5. **Use Views for Flexibility**
```sql
-- Provide normalized view for applications that need it
CREATE VIEW orders_normalized AS
SELECT 
    o.order_id,
    o.customer_id,
    c.name as customer_name,  -- From normalized table
    c.email as customer_email -- From normalized table
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- Use denormalized table for performance-critical queries
-- Use normalized view for data integrity checks
```

---

**When NOT to Denormalize:**

❌ **Don't denormalize when:**
- Write performance is critical
- Data changes frequently
- Storage is very limited
- Strong data consistency required
- Queries don't benefit (no JOINs in hot path)

**Example: When Normalization is Better**
```sql
-- Frequently updated data
CREATE TABLE user_preferences (
    user_id INT,
    theme VARCHAR,      -- Changes often
    language VARCHAR,   -- Changes often
    notifications JSON  -- Changes often
);

-- Don't denormalize into every table
-- JOIN cost is acceptable for rarely-read data
```

---

**Summary:**

Denormalize when:
- ✅ Read-heavy workloads (90%+ reads)
- ✅ JOINs are performance bottlenecks
- ✅ Analytics/reporting systems
- ✅ Real-time requirements
- ✅ Geographic distribution makes JOINs expensive
- ✅ Can accept eventual consistency for denormalized data

Avoid denormalization when:
- ❌ Write-heavy workloads
- ❌ Data changes frequently
- ❌ Strong consistency required
- ❌ Storage constraints
- ❌ No performance benefit

**2. Data lake vs database vs warehouse.**

These are three different data storage paradigms serving different purposes in modern data architectures. Understanding their differences helps choose the right solution for specific use cases.

---

**Database (OLTP - Online Transaction Processing):**

Optimized for transactional workloads: fast reads/writes, ACID compliance, real-time operations.

**Characteristics:**
- **Purpose:** Day-to-day operational data
- **Schema:** Structured, normalized (3NF)
- **Data Type:** Current, transactional data
- **Size:** GB to low TB
- **Latency:** Milliseconds
- **Users:** Applications, end-users
- **Operations:** INSERT, UPDATE, DELETE, simple SELECT

**Example: PostgreSQL OLTP Database**
```sql
-- E-commerce order processing
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP,
    total_amount DECIMAL(10,2),
    status VARCHAR(20)
);

-- Fast transactional operations
INSERT INTO orders (customer_id, order_date, total_amount, status)
VALUES (123, NOW(), 99.99, 'pending');

UPDATE orders SET status = 'shipped' WHERE order_id = 456;
SELECT * FROM orders WHERE customer_id = 123;
```

**Use Cases:**
- E-commerce transactions
- User authentication
- Banking operations
- Inventory management
- CRM systems

---

**Data Warehouse (OLAP - Online Analytical Processing):**

Optimized for analytical queries: aggregations, reporting, business intelligence on historical data.

**Characteristics:**
- **Purpose:** Historical analysis and reporting
- **Schema:** Denormalized (star/snowflake schema)
- **Data Type:** Historical, aggregated data
- **Size:** TB to PB
- **Latency:** Seconds to minutes
- **Users:** Analysts, data scientists, BI tools
- **Operations:** Complex SELECT (aggregations, GROUP BY)

**Example: Snowflake Data Warehouse**
```sql
-- Star schema for analytics
-- Fact table
CREATE TABLE fact_sales (
    sale_id INT,
    date_id INT,
    customer_id INT,
    product_id INT,
    location_id INT,
    sales_amount DECIMAL,
    quantity INT
);

-- Dimension tables
CREATE TABLE dim_date (
    date_id INT PRIMARY KEY,
    date DATE,
    year INT,
    quarter INT,
    month INT,
    day_of_week VARCHAR
);

-- Analytical query
SELECT 
    d.year,
    d.quarter,
    SUM(f.sales_amount) as total_revenue,
    COUNT(DISTINCT f.customer_id) as unique_customers,
    AVG(f.sales_amount) as avg_order_value
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
WHERE d.year = 2024
GROUP BY d.year, d.quarter
ORDER BY d.quarter;
```

**Use Cases:**
- Business intelligence dashboards
- Financial reporting
- Sales analytics
- Trend analysis
- Data mining

**Examples:** Snowflake, Amazon Redshift, Google BigQuery, Azure Synapse

---

**Data Lake:**

Centralized repository storing raw data in its native format (structured, semi-structured, unstructured) at scale.

**Characteristics:**
- **Purpose:** Store all data for future use
- **Schema:** Schema-on-read (no predefined schema)
- **Data Type:** Raw, unprocessed data (all formats)
- **Size:** PB to EB
- **Latency:** Minutes to hours (batch processing)
- **Users:** Data engineers, data scientists
- **Operations:** Batch processing, ETL/ELT

**Example: S3 Data Lake**
```python
# Store raw data in S3
# Structured data (CSV, Parquet)
s3://data-lake/raw/orders/2024/01/orders_20240101.csv
s3://data-lake/raw/users/2024/01/users_20240101.parquet

# Semi-structured data (JSON)
s3://data-lake/raw/logs/2024/01/app_logs_20240101.json

# Unstructured data
s3://data-lake/raw/documents/2024/01/report_20240101.pdf
s3://data-lake/raw/images/2024/01/product_images/

# Process with Spark/Presto (schema-on-read)
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("DataLake").getOrCreate()

# Read with schema defined at query time
df = spark.read.parquet("s3://data-lake/raw/orders/")
df.createOrReplaceTempView("orders")

# Query with Spark SQL
results = spark.sql("""
    SELECT 
        DATE_TRUNC('month', order_date) as month,
        COUNT(*) as order_count,
        SUM(total_amount) as revenue
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY month
""")
```

**Use Cases:**
- Big data analytics
- Machine learning training data
- Data archival
- Multi-format data storage
- Exploratory data analysis

**Examples:** AWS S3, Azure Data Lake Storage, Google Cloud Storage, Hadoop HDFS

---

**Comparison Table:**

| Aspect | Database (OLTP) | Data Warehouse (OLAP) | Data Lake |
|--------|----------------|----------------------|-----------|
| **Purpose** | Operational transactions | Analytics & reporting | Store all raw data |
| **Schema** | Schema-on-write (normalized) | Schema-on-write (denormalized) | Schema-on-read (no schema) |
| **Data Type** | Current, structured | Historical, structured | Raw, all formats |
| **Data Size** | GB - low TB | TB - PB | PB - EB |
| **Latency** | Milliseconds | Seconds - minutes | Minutes - hours |
| **Users** | Applications | Analysts, BI tools | Data engineers, scientists |
| **Optimization** | Fast writes, simple reads | Fast aggregations | Scale, cost |
| **ACID** | Yes (strong) | Limited | No |
| **Cost** | Medium-High | High | Low (object storage) |
| **Examples** | PostgreSQL, MySQL | Snowflake, Redshift | S3, ADLS, HDFS |

---

**Data Pipeline Architecture:**

```
┌─────────────┐
│   Sources   │
│ (Apps, APIs)│
└──────┬──────┘
       │
       ▼
┌─────────────┐      ┌──────────────┐      ┌─────────────┐
│  Database   │─────▶│ Data Warehouse│      │  Data Lake  │
│   (OLTP)    │      │    (OLAP)    │      │  (Raw Data) │
│             │      │              │      │             │
│ Real-time   │      │ Analytics    │      │ ML, ETL     │
│ Transactions│      │ & Reporting   │      │ Processing   │
└─────────────┘      └──────────────┘      └─────────────┘
```

**Example: E-commerce Data Architecture**
```python
# 1. Database (OLTP) - Real-time operations
# PostgreSQL for orders, users, products
def create_order(customer_id, items):
    with db.transaction():
        order = db.execute("""
            INSERT INTO orders (customer_id, total)
            VALUES (%s, %s) RETURNING order_id
        """, customer_id, calculate_total(items))
        return order

# 2. Data Warehouse (OLAP) - Analytics
# Snowflake for business intelligence
def get_sales_report(start_date, end_date):
    return warehouse.query("""
        SELECT 
            DATE_TRUNC('day', order_date) as date,
            SUM(total_amount) as daily_revenue,
            COUNT(DISTINCT customer_id) as customers
        FROM fact_orders
        WHERE order_date BETWEEN %s AND %s
        GROUP BY date
        ORDER BY date
    """, start_date, end_date)

# 3. Data Lake - Machine Learning
# S3 for storing raw data for ML training
def train_recommendation_model():
    # Read raw order data from data lake
    raw_orders = spark.read.parquet("s3://data-lake/raw/orders/")
    raw_users = spark.read.parquet("s3://data-lake/raw/users/")
    
    # Feature engineering
    features = engineer_features(raw_orders, raw_users)
    
    # Train model
    model = train_model(features)
    
    # Save model
    model.save("s3://data-lake/models/recommendation/")
```

---

**When to Use Each:**

**Use Database (OLTP) when:**
- ✅ Real-time transactional operations
- ✅ ACID compliance required
- ✅ Fast single-row operations
- ✅ Application needs immediate consistency
- ✅ Structured data only

**Use Data Warehouse (OLAP) when:**
- ✅ Business intelligence and reporting
- ✅ Historical data analysis
- ✅ Complex aggregations
- ✅ Predefined analytical queries
- ✅ Structured data with known schema

**Use Data Lake when:**
- ✅ Store all data (future-proofing)
- ✅ Multiple data formats (structured, unstructured)
- ✅ Machine learning and data science
- ✅ Cost-effective long-term storage
- ✅ Exploratory analysis on raw data
- ✅ Large-scale batch processing

---

**Modern Architecture: Data Lakehouse**

Combines benefits of data lake and data warehouse:
- Store raw data (like data lake)
- Provide SQL interface (like data warehouse)
- ACID transactions
- Schema enforcement options

**Example: Delta Lake**
```python
# Write to Delta Lake (on S3)
df.write.format("delta").mode("overwrite").save("s3://data-lake/delta/orders")

# Query with SQL (like data warehouse)
spark.sql("""
    SELECT 
        DATE_TRUNC('month', order_date) as month,
        SUM(total_amount) as revenue
    FROM delta.`s3://data-lake/delta/orders`
    WHERE order_date >= '2024-01-01'
    GROUP BY month
""")

# ACID transactions
spark.sql("""
    UPDATE delta.`s3://data-lake/delta/orders`
    SET status = 'cancelled'
    WHERE order_id = 123
""")
```

---

**Summary:**

- **Database (OLTP):** Fast transactions, real-time operations, structured data
- **Data Warehouse (OLAP):** Analytics, reporting, historical data, structured data
- **Data Lake:** Raw data storage, all formats, big data, cost-effective
- **Modern Trend:** Data Lakehouse combines lake + warehouse benefits

**3. Batch vs streaming ingestion.**

Data ingestion refers to how data is collected and loaded into storage systems. The choice between batch and streaming depends on latency requirements, data volume, and use case.

---

**Batch Ingestion:**

Processes data in discrete chunks at scheduled intervals (hourly, daily, etc.). Data is collected over a period, then processed together.

**Characteristics:**
- **Latency:** High (minutes to hours)
- **Processing:** Periodic, scheduled
- **Volume:** Large batches
- **Complexity:** Simpler to implement
- **Cost:** Lower (resource-efficient)
- **Use Cases:** Reporting, analytics, ETL pipelines

**Example: Daily ETL Pipeline**
```python
# Batch ingestion: Process once per day
import schedule
import time

def daily_etl():
    # 1. Extract: Read all transactions from yesterday
    transactions = extract_from_source(
        source='postgresql',
        query='SELECT * FROM transactions WHERE date = CURRENT_DATE - 1'
    )
    
    # 2. Transform: Clean and aggregate
    cleaned_data = transform(transactions)
    
    # 3. Load: Write to data warehouse
    load_to_warehouse(
        destination='snowflake',
        table='fact_transactions',
        data=cleaned_data
    )
    
    print(f"Processed {len(transactions)} transactions")

# Schedule daily at 2 AM
schedule.every().day.at("02:00").do(daily_etl)

while True:
    schedule.run_pending()
    time.sleep(60)
```

**Batch Processing Architecture:**
```
Source Systems → Batch Collector → Buffer → ETL Pipeline → Data Warehouse
                (Collects data)    (S3/SFTP)   (Daily/Hourly)  (Snowflake)
```

**Example: Apache Airflow Batch Pipeline**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime, timedelta

def extract_data():
    # Extract from source
    data = read_from_postgres("SELECT * FROM orders WHERE date = '{{ ds }}'")
    return data

def transform_data(**context):
    # Transform data
    data = context['ti'].xcom_pull(task_ids='extract')
    transformed = clean_and_aggregate(data)
    return transformed

def load_data(**context):
    # Load to warehouse
    data = context['ti'].xcom_pull(task_ids='transform')
    write_to_snowflake(data)

dag = DAG(
    'daily_etl',
    schedule_interval='0 2 * * *',  # Daily at 2 AM
    start_date=datetime(2024, 1, 1)
)

extract_task = PythonOperator(
    task_id='extract',
    python_callable=extract_data,
    dag=dag
)

transform_task = PythonOperator(
    task_id='transform',
    python_callable=transform_data,
    dag=dag
)

load_task = PythonOperator(
    task_id='load',
    python_callable=load_data,
    dag=dag
)

extract_task >> transform_task >> load_task
```

**When to Use Batch:**
- ✅ Historical analysis (not real-time)
- ✅ Scheduled reporting
- ✅ Large volumes (cost-effective)
- ✅ Complex transformations
- ✅ No immediate action needed

---

**Streaming Ingestion:**

Processes data continuously as it arrives, enabling real-time or near-real-time processing.

**Characteristics:**
- **Latency:** Low (milliseconds to seconds)
- **Processing:** Continuous, event-driven
- **Volume:** Individual events or small batches
- **Complexity:** More complex (state management, ordering)
- **Cost:** Higher (always-on resources)
- **Use Cases:** Real-time dashboards, fraud detection, monitoring

**Example: Real-Time Event Processing**
```python
# Streaming ingestion: Process events as they arrive
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

def process_event(event):
    # Process immediately
    user_id = event['user_id']
    event_type = event['event_type']
    
    # Update real-time dashboard
    update_dashboard(user_id, event_type)
    
    # Detect anomalies
    if detect_fraud(event):
        alert_security_team(event)
    
    # Write to database
    write_to_postgres(event)

# Process events as they arrive
for message in consumer:
    event = message.value
    process_event(event)  # Process immediately ⚡
```

**Streaming Architecture:**
```
Source → Message Queue → Stream Processor → Real-time DB → Dashboard
        (Kafka/Kinesis)   (Flink/Spark)     (Redis/ClickHouse)  (Grafana)
```

**Example: Apache Flink Streaming**
```python
from pyflink.datastream import StreamExecutionEnvironment
from pyflink.table import StreamTableEnvironment

env = StreamExecutionEnvironment.get_execution_environment()
table_env = StreamTableEnvironment.create(env)

# Read from Kafka
table_env.execute_sql("""
    CREATE TABLE events (
        user_id BIGINT,
        event_type STRING,
        timestamp TIMESTAMP(3),
        WATERMARK FOR timestamp AS timestamp - INTERVAL '5' SECOND
    ) WITH (
        'connector' = 'kafka',
        'topic' = 'user-events',
        'properties.bootstrap.servers' = 'localhost:9092',
        'format' = 'json'
    )
""")

# Process in real-time
table_env.execute_sql("""
    CREATE TABLE dashboard_stats AS
    SELECT 
        TUMBLE_START(timestamp, INTERVAL '1' MINUTE) as window_start,
        event_type,
        COUNT(*) as event_count
    FROM events
    GROUP BY TUMBLE(timestamp, INTERVAL '1' MINUTE), event_type
""")

# Write to ClickHouse for real-time queries
table_env.execute_sql("""
    CREATE TABLE clickhouse_sink (
        window_start TIMESTAMP,
        event_type STRING,
        event_count BIGINT
    ) WITH (
        'connector' = 'clickhouse',
        'url' = 'http://localhost:8123',
        'database-name' = 'analytics',
        'table-name' = 'realtime_stats'
    )
""")

table_env.execute_sql("INSERT INTO clickhouse_sink SELECT * FROM dashboard_stats")
```

---

**Comparison Table:**

| Aspect | Batch Ingestion | Streaming Ingestion |
|--------|----------------|---------------------|
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Processing** | Scheduled intervals | Continuous |
| **Data Volume** | Large batches | Individual events |
| **Complexity** | Simpler | More complex |
| **Cost** | Lower (efficient) | Higher (always-on) |
| **Use Cases** | Reporting, analytics | Real-time dashboards, alerts |
| **Tools** | Airflow, Spark Batch | Kafka, Flink, Spark Streaming |
| **Error Handling** | Easier (retry batch) | Harder (state management) |

---

**Hybrid Approach: Lambda Architecture**

Combines batch and streaming for comprehensive data processing:

```
┌─────────────┐
│  Data Source│
└──────┬──────┘
       │
       ├──────────────┬──────────────┐
       ▼              ▼              ▼
  ┌─────────┐   ┌──────────┐   ┌─────────┐
  │ Stream  │   │  Batch   │   │ Stream  │
  │ (Speed)  │   │ (Batch)  │   │ (Speed) │
  └────┬────┘   └─────┬────┘   └────┬────┘
       │              │              │
       ▼              ▼              ▼
  ┌──────────────────────────────────┐
  │     Serving Layer (Merge)        │
  │  (Real-time + Batch = Complete) │
  └──────────────────────────────────┘
```

**Example: Lambda Architecture**
```python
# Speed Layer: Real-time streaming
def speed_layer():
    # Process events immediately (may have errors)
    stream_processor.process_kafka_stream()
    # Write to Redis for real-time queries
    write_to_redis(real_time_data)

# Batch Layer: Accurate batch processing
def batch_layer():
    # Process all data daily (accurate)
    batch_processor.process_yesterday_data()
    # Write to data warehouse
    write_to_warehouse(accurate_data)

# Serving Layer: Merge both
def serving_layer():
    # Query real-time data (speed layer)
    real_time = read_from_redis()
    
    # Query historical data (batch layer)
    historical = read_from_warehouse()
    
    # Merge for complete view
    return merge(real_time, historical)
```

---

**Real-World Examples:**

**1. E-commerce Analytics (Batch)**
```python
# Daily sales report (batch)
def daily_sales_report():
    # Process all orders from yesterday
    orders = db.query("""
        SELECT 
            DATE_TRUNC('day', order_date) as date,
            SUM(total_amount) as revenue,
            COUNT(*) as order_count
        FROM orders
        WHERE order_date >= CURRENT_DATE - 1
        GROUP BY date
    """)
    
    # Generate report
    generate_report(orders)
    # Send email report
    send_email_report(orders)
```

**2. Fraud Detection (Streaming)**
```python
# Real-time fraud detection (streaming)
def detect_fraud_stream():
    kafka_consumer = KafkaConsumer('transactions')
    
    for transaction in kafka_consumer:
        # Check fraud in real-time
        if is_fraudulent(transaction):
            # Block transaction immediately
            block_transaction(transaction.id)
            # Alert security team
            alert_security(transaction)
```

**3. IoT Sensor Monitoring (Streaming)**
```python
# Real-time sensor alerts (streaming)
def monitor_sensors():
    mqtt_client.subscribe('sensors/#')
    
    @mqtt_client.on_message()
    def handle_message(client, userdata, msg):
        sensor_data = json.loads(msg.payload)
        
        # Check threshold in real-time
        if sensor_data['temperature'] > 100:
            # Immediate alert
            send_alert(f"High temperature: {sensor_data['temperature']}")
            # Trigger cooling system
            trigger_cooling(sensor_data['sensor_id'])
```

**4. Business Intelligence (Batch)**
```python
# Weekly business metrics (batch)
def weekly_metrics():
    # Aggregate all data for the week
    metrics = warehouse.query("""
        SELECT 
            week,
            SUM(revenue) as weekly_revenue,
            COUNT(DISTINCT customer_id) as active_customers,
            AVG(order_value) as avg_order_value
        FROM fact_sales
        WHERE week = EXTRACT(WEEK FROM CURRENT_DATE)
        GROUP BY week
    """)
    
    # Update dashboard
    update_bi_dashboard(metrics)
```

---

**When to Choose:**

**Choose Batch when:**
- ✅ Reporting and analytics (not real-time)
- ✅ Large data volumes (cost-effective)
- ✅ Complex transformations needed
- ✅ Scheduled processing acceptable
- ✅ Historical analysis

**Choose Streaming when:**
- ✅ Real-time alerts needed
- ✅ Fraud detection
- ✅ Monitoring and observability
- ✅ Real-time dashboards
- ✅ Immediate action required

**Choose Hybrid (Lambda) when:**
- ✅ Need both real-time and accurate views
- ✅ Can afford complexity
- ✅ Comprehensive data architecture

---

**Summary:**

- **Batch:** Scheduled, large volumes, minutes to hours latency, simpler, cost-effective
- **Streaming:** Continuous, real-time, milliseconds latency, complex, higher cost
- **Hybrid:** Combines both for complete solution
- Choose based on latency requirements and use case

**4. Designing schema for analytics-heavy systems.**

Analytics-heavy systems require schemas optimized for complex queries, aggregations, and reporting rather than transactional operations. Key principles: denormalization, star/snowflake schemas, and columnar storage.

---

**Core Principles:**

### 1. **Star Schema Design**

Separates data into fact tables (measurable events) and dimension tables (descriptive attributes).

**Structure:**
```
         Dimension Tables (Descriptive)
              │
              │ Foreign Keys
              │
    ┌─────────┼─────────┐
    │         │         │
Fact Table (Measures/Events)
```

**Example: E-commerce Analytics**
```sql
-- Fact Table: Sales transactions
CREATE TABLE fact_sales (
    sale_id SERIAL PRIMARY KEY,
    -- Foreign keys to dimensions
    date_id INT NOT NULL,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    location_id INT NOT NULL,
    -- Measures (what we're analyzing)
    sales_amount DECIMAL(10,2),
    quantity INT,
    discount_amount DECIMAL(10,2),
    shipping_cost DECIMAL(10,2),
    -- Foreign key constraints
    CONSTRAINT fk_date FOREIGN KEY (date_id) REFERENCES dim_date(date_id),
    CONSTRAINT fk_customer FOREIGN KEY (customer_id) REFERENCES dim_customer(customer_id),
    CONSTRAINT fk_product FOREIGN KEY (product_id) REFERENCES dim_product(product_id),
    CONSTRAINT fk_location FOREIGN KEY (location_id) REFERENCES dim_location(location_id)
);

-- Dimension Table: Date
CREATE TABLE dim_date (
    date_id INT PRIMARY KEY,
    date DATE NOT NULL UNIQUE,
    year INT,
    quarter INT,
    month INT,
    week INT,
    day_of_month INT,
    day_of_week INT,
    day_name VARCHAR(10),
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
);

-- Dimension Table: Customer
CREATE TABLE dim_customer (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    email VARCHAR(100),
    age INT,
    gender VARCHAR(10),
    city VARCHAR(50),
    state VARCHAR(50),
    country VARCHAR(50),
    customer_segment VARCHAR(50),
    registration_date DATE
);

-- Dimension Table: Product
CREATE TABLE dim_product (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(200),
    category VARCHAR(50),
    subcategory VARCHAR(50),
    brand VARCHAR(50),
    price DECIMAL(10,2),
    cost DECIMAL(10,2)
);

-- Dimension Table: Location
CREATE TABLE dim_location (
    location_id INT PRIMARY KEY,
    store_name VARCHAR(100),
    city VARCHAR(50),
    state VARCHAR(50),
    country VARCHAR(50),
    region VARCHAR(50)
);
```

**Analytical Query Example:**
```sql
-- Fast aggregation using star schema
SELECT 
    d.year,
    d.quarter,
    p.category,
    l.region,
    SUM(f.sales_amount) as total_revenue,
    COUNT(DISTINCT f.customer_id) as unique_customers,
    AVG(f.sales_amount) as avg_order_value,
    SUM(f.quantity) as total_units_sold
FROM fact_sales f
JOIN dim_date d ON f.date_id = d.date_id
JOIN dim_product p ON f.product_id = p.product_id
JOIN dim_location l ON f.location_id = l.location_id
WHERE d.year = 2024
GROUP BY d.year, d.quarter, p.category, l.region
ORDER BY total_revenue DESC;
```

### 2. **Denormalization for Performance**

Pre-join frequently accessed dimension attributes into fact table.

**Example: Denormalized Fact Table**
```sql
-- Denormalized fact table (faster queries, more storage)
CREATE TABLE fact_sales_denormalized (
    sale_id SERIAL PRIMARY KEY,
    sale_date DATE NOT NULL,
    -- Denormalized date attributes
    year INT,
    quarter INT,
    month INT,
    -- Denormalized customer attributes
    customer_id INT,
    customer_segment VARCHAR(50),
    customer_region VARCHAR(50),
    -- Denormalized product attributes
    product_id INT,
    product_category VARCHAR(50),
    product_brand VARCHAR(50),
    -- Denormalized location attributes
    location_id INT,
    store_region VARCHAR(50),
    -- Measures
    sales_amount DECIMAL(10,2),
    quantity INT
);

-- Faster query (no JOINs needed)
SELECT 
    year,
    quarter,
    product_category,
    store_region,
    SUM(sales_amount) as revenue
FROM fact_sales_denormalized
WHERE year = 2024
GROUP BY year, quarter, product_category, store_region;
```

### 3. **Columnar Storage**

Optimize for column-based queries (aggregations, filtering).

**Example: TimescaleDB (Columnar-like)**
```sql
-- TimescaleDB hypertable for time-series analytics
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INT NOT NULL,
    temperature DECIMAL(5,2),
    humidity DECIMAL(5,2),
    pressure DECIMAL(8,2)
);

-- Convert to hypertable
SELECT create_hypertable('sensor_readings', 'time');

-- Efficient time-based queries
SELECT 
    time_bucket('1 hour', time) as hour,
    sensor_id,
    AVG(temperature) as avg_temp,
    MAX(temperature) as max_temp,
    MIN(temperature) as min_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '24 hours'
GROUP BY hour, sensor_id
ORDER BY hour DESC;
```

### 4. **Pre-computed Aggregates**

Store frequently accessed aggregations.

**Example: Materialized Aggregates**
```sql
-- Pre-computed daily aggregates
CREATE TABLE daily_sales_aggregates (
    date DATE PRIMARY KEY,
    total_revenue DECIMAL(12,2),
    total_orders INT,
    unique_customers INT,
    avg_order_value DECIMAL(10,2),
    top_product_id INT,
    top_category VARCHAR(50)
);

-- Update via trigger or scheduled job
CREATE OR REPLACE FUNCTION update_daily_aggregates()
RETURNS void AS $$
BEGIN
    INSERT INTO daily_sales_aggregates
    SELECT 
        DATE_TRUNC('day', sale_date) as date,
        SUM(sales_amount) as total_revenue,
        COUNT(*) as total_orders,
        COUNT(DISTINCT customer_id) as unique_customers,
        AVG(sales_amount) as avg_order_value,
        MODE() WITHIN GROUP (ORDER BY product_id) as top_product_id,
        MODE() WITHIN GROUP (ORDER BY product_category) as top_category
    FROM fact_sales_denormalized
    WHERE sale_date = CURRENT_DATE - 1
    GROUP BY DATE_TRUNC('day', sale_date)
    ON CONFLICT (date) DO UPDATE SET
        total_revenue = EXCLUDED.total_revenue,
        total_orders = EXCLUDED.total_orders,
        unique_customers = EXCLUDED.unique_customers,
        avg_order_value = EXCLUDED.avg_order_value,
        top_product_id = EXCLUDED.top_product_id,
        top_category = EXCLUDED.top_category;
END;
$$ LANGUAGE plpgsql;

-- Fast dashboard query
SELECT * FROM daily_sales_aggregates
WHERE date >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY date DESC;
```

### 5. **Partitioning for Performance**

Partition large fact tables by time or other dimensions.

**Example: Range Partitioning**
```sql
-- Partition fact table by month
CREATE TABLE fact_sales (
    sale_id SERIAL,
    sale_date DATE NOT NULL,
    customer_id INT,
    product_id INT,
    sales_amount DECIMAL(10,2),
    PRIMARY KEY (sale_id, sale_date)
) PARTITION BY RANGE (sale_date);

-- Create monthly partitions
CREATE TABLE fact_sales_2024_01 PARTITION OF fact_sales
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
CREATE TABLE fact_sales_2024_02 PARTITION OF fact_sales
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Query only scans relevant partition
SELECT SUM(sales_amount)
FROM fact_sales
WHERE sale_date >= '2024-01-01' AND sale_date < '2024-02-01';
-- Only scans fact_sales_2024_01 partition
```

### 6. **Indexing Strategy**

Indexes optimized for analytical queries.

**Example: Bitmap Indexes for Low Cardinality**
```sql
-- Bitmap index on low-cardinality columns
CREATE INDEX idx_sales_category ON fact_sales_denormalized(product_category);
CREATE INDEX idx_sales_region ON fact_sales_denormalized(store_region);
CREATE INDEX idx_sales_segment ON fact_sales_denormalized(customer_segment);

-- Composite index for common query patterns
CREATE INDEX idx_sales_date_category ON fact_sales_denormalized(year, quarter, product_category);
```

---

**Complete Analytics Schema Example:**

```sql
-- 1. Fact Table: Sales Transactions
CREATE TABLE fact_sales (
    sale_id BIGSERIAL,
    -- Time dimension (denormalized)
    sale_timestamp TIMESTAMPTZ NOT NULL,
    sale_date DATE NOT NULL,
    year INT NOT NULL,
    quarter INT NOT NULL,
    month INT NOT NULL,
    week INT NOT NULL,
    -- Customer dimension (denormalized)
    customer_id INT NOT NULL,
    customer_segment VARCHAR(50),
    customer_age_group VARCHAR(20),
    customer_region VARCHAR(50),
    -- Product dimension (denormalized)
    product_id INT NOT NULL,
    product_category VARCHAR(50),
    product_subcategory VARCHAR(50),
    product_brand VARCHAR(50),
    -- Location dimension (denormalized)
    location_id INT NOT NULL,
    store_name VARCHAR(100),
    store_region VARCHAR(50),
    -- Measures
    sales_amount DECIMAL(12,2) NOT NULL,
    quantity INT NOT NULL,
    discount_amount DECIMAL(10,2),
    shipping_cost DECIMAL(10,2),
    profit DECIMAL(12,2),
    PRIMARY KEY (sale_id, sale_date)
) PARTITION BY RANGE (sale_date);

-- Create monthly partitions
CREATE TABLE fact_sales_2024_01 PARTITION OF fact_sales
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- 2. Aggregation Tables (Pre-computed)
CREATE TABLE daily_sales_summary (
    date DATE PRIMARY KEY,
    total_revenue DECIMAL(12,2),
    total_orders BIGINT,
    unique_customers INT,
    avg_order_value DECIMAL(10,2),
    top_category VARCHAR(50),
    top_product_id INT
);

CREATE TABLE monthly_sales_summary (
    year INT,
    month INT,
    total_revenue DECIMAL(12,2),
    total_orders BIGINT,
    unique_customers INT,
    avg_order_value DECIMAL(10,2),
    PRIMARY KEY (year, month)
);

-- 3. Indexes for Analytics
CREATE INDEX idx_sales_date ON fact_sales(sale_date);
CREATE INDEX idx_sales_category ON fact_sales(product_category);
CREATE INDEX idx_sales_region ON fact_sales(store_region);
CREATE INDEX idx_sales_date_category ON fact_sales(sale_date, product_category);

-- 4. Materialized Views
CREATE MATERIALIZED VIEW mv_category_sales AS
SELECT 
    product_category,
    DATE_TRUNC('month', sale_date) as month,
    SUM(sales_amount) as revenue,
    COUNT(*) as order_count
FROM fact_sales
GROUP BY product_category, DATE_TRUNC('month', sale_date);

CREATE UNIQUE INDEX ON mv_category_sales(product_category, month);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_category_sales;
```

---

**Best Practices:**

1. **Denormalize Dimensions:** Store frequently accessed dimension attributes in fact table
2. **Partition by Time:** Partition large fact tables by date for query performance
3. **Pre-compute Aggregates:** Create summary tables for common queries
4. **Use Columnar Storage:** Consider columnar databases (Redshift, BigQuery) for analytics
5. **Index Strategically:** Index columns used in WHERE, GROUP BY, ORDER BY
6. **Materialized Views:** Pre-compute expensive aggregations
7. **Separate OLTP and OLAP:** Use different schemas for transactional vs analytical workloads

---

**Summary:**

Analytics schema design prioritizes:
- ✅ Denormalization over normalization
- ✅ Star/snowflake schemas
- ✅ Columnar storage for aggregations
- ✅ Partitioning by time
- ✅ Pre-computed aggregates
- ✅ Optimized indexes for analytical queries

---

### C. Real System Design Scenarios

**1. Design a scalable logging system (PostgreSQL + Elasticsearch).**

**Requirements:**
- Handle millions of log entries per day
- Fast search and filtering
- Long-term storage (1+ years)
- Real-time log tailing
- Structured and unstructured logs

**Architecture:**

```
Applications → Log Collector → Message Queue → Processing → PostgreSQL (Metadata) + Elasticsearch (Logs)
                              (Kafka)          (Logstash)  └─ Search & Analytics
```

**Component Design:**

### 1. **PostgreSQL: Metadata Storage**

```sql
-- Log metadata table
CREATE TABLE log_metadata (
    log_id BIGSERIAL PRIMARY KEY,
    application_name VARCHAR(100) NOT NULL,
    log_level VARCHAR(20) NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    hostname VARCHAR(100),
    environment VARCHAR(50),
    elasticsearch_id VARCHAR(100) NOT NULL,
    CONSTRAINT idx_metadata_app_time UNIQUE (application_name, timestamp, log_id)
);

-- Partition by date
CREATE TABLE log_metadata_2024_01 PARTITION OF log_metadata
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE INDEX idx_metadata_app ON log_metadata(application_name);
CREATE INDEX idx_metadata_level ON log_metadata(log_level);
CREATE INDEX idx_metadata_timestamp ON log_metadata(timestamp);
```

### 2. **Elasticsearch: Full-Text Search**

```json
{
  "mappings": {
    "properties": {
      "application_name": { "type": "keyword" },
      "log_level": { "type": "keyword" },
      "timestamp": { "type": "date" },
      "message": { "type": "text", "analyzer": "standard" }
    }
  }
}
```

**Benefits:**
- PostgreSQL: Fast metadata queries, referential integrity
- Elasticsearch: Full-text search, aggregations, real-time search
- Scalable: Kafka handles high throughput
- Efficient: Separate storage based on access patterns


**2. Design a recommendation engine database layer.**

**Requirements:**
- Real-time recommendations
- User-item interactions
- Collaborative filtering
- Content-based filtering
- Fast retrieval (< 100ms)

**Architecture:**

```
User Actions → Event Stream → Feature Store → ML Models → Recommendation DB → API
              (Kafka)        (Redis/PostgreSQL)  (Training)  (Redis/PostgreSQL)
```

**Database Schema:**

### 1. **PostgreSQL: User-Item Interactions**

```sql
-- User-item interactions
CREATE TABLE user_item_interactions (
    interaction_id BIGSERIAL PRIMARY KEY,
    user_id INT NOT NULL,
    item_id INT NOT NULL,
    interaction_type VARCHAR(20) NOT NULL,  -- view, click, purchase, rating
    rating DECIMAL(2,1),
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    context JSONB,
    UNIQUE(user_id, item_id, interaction_type, DATE(timestamp))
);

CREATE INDEX idx_interactions_user ON user_item_interactions(user_id, timestamp DESC);
CREATE INDEX idx_interactions_item ON user_item_interactions(item_id, timestamp DESC);

-- Item features
CREATE TABLE item_features (
    item_id INT PRIMARY KEY,
    category VARCHAR(50),
    tags TEXT[],
    features JSONB,
    popularity_score DECIMAL(10,2)
);
```

### 2. **Redis: Real-Time Recommendations**

```python
import redis

redis_client = redis.Redis(host='localhost', port=6379)

# Store pre-computed recommendations
def store_recommendations(user_id, recommendations):
    key = f"recommendations:user:{user_id}"
    pipe = redis_client.pipeline()
    pipe.delete(key)
    for item_id, score in recommendations:
        pipe.zadd(key, {str(item_id): score})
    pipe.expire(key, 3600)
    pipe.execute()

# Retrieve recommendations
def get_recommendations(user_id, limit=10):
    key = f"recommendations:user:{user_id}"
    item_ids = redis_client.zrevrange(key, 0, limit - 1)
    return [int(item_id) for item_id in item_ids]
```

**Key Design Decisions:**
- PostgreSQL: Persistent storage, complex queries, user-item interactions
- Redis: Fast real-time recommendations, caching pre-computed results
- Hybrid approach: Combines collaborative + content-based filtering


**3. Handle billions of time-series events per day (TimescaleDB vs Cassandra).**

**Requirements:**
- 1 billion+ events per day (~11,500 events/second)
- Fast time-range queries
- High write throughput
- Long-term retention (years)
- Real-time analytics

**Comparison:**

| Aspect | TimescaleDB | Cassandra |
|--------|------------|-----------|
| **Write Throughput** | High (PostgreSQL-based) | Very High (distributed) |
| **Time-Range Queries** | Excellent (optimized) | Good (with proper clustering) |
| **SQL Support** | Full PostgreSQL SQL | CQL (limited) |
| **Scalability** | Vertical + horizontal | Horizontal (built-in) |
| **Consistency** | Strong (ACID) | Tunable (eventual) |

**TimescaleDB Solution:**

```sql
-- Create hypertable for time-series data
CREATE TABLE sensor_events (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INT NOT NULL,
    metric_name VARCHAR(50) NOT NULL,
    value DOUBLE PRECISION,
    tags JSONB,
    PRIMARY KEY (time, sensor_id, metric_name)
);

-- Convert to hypertable (automatic partitioning)
SELECT create_hypertable('sensor_events', 'time',
    chunk_time_interval => INTERVAL '1 day'
);

-- Compression for old data
ALTER TABLE sensor_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id, metric_name'
);

-- Continuous aggregates (pre-computed)
CREATE MATERIALIZED VIEW sensor_hourly_stats
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) as hour,
    sensor_id,
    metric_name,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value
FROM sensor_events
GROUP BY hour, sensor_id, metric_name;
```

**Cassandra Solution:**

```sql
-- Table optimized for time-range queries
CREATE TABLE sensor_events (
    sensor_id INT,
    date_bucket TEXT,  -- '2024-01-15' (partition key)
    timestamp TIMESTAMP,
    metric_name TEXT,
    value DOUBLE,
    tags MAP<TEXT, TEXT>,
    PRIMARY KEY ((sensor_id, date_bucket), timestamp, metric_name)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**Recommendation:**

**Use TimescaleDB when:**
- ✅ Need SQL and complex queries
- ✅ Strong consistency required
- ✅ Time-range queries are primary use case
- ✅ PostgreSQL ecosystem preferred

**Use Cassandra when:**
- ✅ Extremely high write throughput needed
- ✅ Global distribution required
- ✅ Eventual consistency acceptable
- ✅ Need horizontal scaling from start


**4. Design fault-tolerant architecture for global read replicas.**

**Requirements:**
- Global read replicas (multi-region)
- High availability (99.99%)
- Low latency reads
- Automatic failover
- Data consistency

**Architecture:**

```
Primary (US-East)
    │
    ├── Replica 1 (US-West) ── Reads
    ├── Replica 2 (EU-West) ── Reads
    ├── Replica 3 (Asia-Pacific) ── Reads
    └── Replica 4 (EU-East) ── Reads
```

**PostgreSQL Setup:**

```sql
-- Primary Configuration
-- postgresql.conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
hot_standby = on

-- Create replication user
CREATE USER replicator WITH REPLICATION PASSWORD 'secure_password';
```

**Application-Level Routing:**

```python
from sqlalchemy import create_engine

# Connection pools for each region
primary_pool = create_engine('postgresql://user:pass@primary-us-east:5432/db')

replica_pools = {
    'us-west': create_engine('postgresql://user:pass@replica-us-west:5432/db'),
    'eu-west': create_engine('postgresql://user:pass@replica-eu-west:5432/db'),
    'asia': create_engine('postgresql://user:pass@replica-asia:5432/db')
}

def get_read_connection(user_region=None):
    """Route reads to nearest replica"""
    if user_region:
        region_map = {'us': 'us-west', 'eu': 'eu-west', 'asia': 'asia'}
        replica_region = region_map.get(user_region, 'us-west')
        return replica_pools[replica_region]
    return random.choice(list(replica_pools.values()))

def get_write_connection():
    """Always use primary for writes"""
    return primary_pool
```

**Automatic Failover (Patroni):**

```yaml
# patroni.yml
scope: my-cluster
name: primary-us-east

etcd:
  hosts: etcd1:2379,etcd2:2379,etcd3:2379

bootstrap:
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576
```

**Health Monitoring:**

```python
def monitor_replica_health():
    """Monitor replica lag and health"""
    for region, pool in replica_pools.items():
        try:
            conn = pool.connect()
            cursor = conn.cursor()
            
            # Check replication lag
            cursor.execute("""
                SELECT EXTRACT(EPOCH FROM (NOW() - pg_last_xact_replay_timestamp())) as lag
            """)
            lag = cursor.fetchone()[0]
            
            # Remove from pool if lag too high
            if lag > 30:  # 30 seconds
                mark_replica_unhealthy(region)
            else:
                mark_replica_healthy(region)
            
            conn.close()
        except Exception as e:
            mark_replica_unhealthy(region)
```

**Circuit Breaker Pattern:**

```python
class ReplicaCircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = {}
        self.state = {}
    
    def is_available(self, region):
        """Check if replica is available"""
        if self.state.get(region) == 'open':
            if time.time() - self.failures[region] > self.timeout:
                self.state[region] = 'half_open'
                return True
            return False
        return True
    
    def record_success(self, region):
        self.state[region] = 'closed'
        self.failures[region] = 0
    
    def record_failure(self, region):
        self.failures[region] = self.failures.get(region, 0) + 1
        if self.failures[region] >= self.failure_threshold:
            self.state[region] = 'open'
            self.failures[region] = time.time()
```

**Summary:**
- ✅ Multi-region replicas for low latency
- ✅ Automatic failover with Patroni
- ✅ Health monitoring and circuit breakers
- ✅ Read routing to nearest replica
- ✅ Write routing always to primary
- ✅ Read-after-write consistency for critical operations

