# 8. Distributed Systems Theory for Practitioners

> Phase 3 · Week 11 · [← Phase index](README.md)

**Why this matters:** Theory explains *why* the patterns in guides 5–7 exist. When someone proposes "just use a distributed lock" or "read from the replica", you need the vocabulary to spot the hidden failure modes. System design interviews lean on exactly these concepts.

---

## 8.1 Partial failure: the defining property

On one machine, an operation either works or the whole process crashes. In a distributed system, **some parts fail while others keep running**, and often **you can't tell which**:

> You sent a request and got no response. Did the request get lost? Did the server crash before processing it? After processing it? Is the response just slow?

**From the caller's side, "slow" and "dead" look the same.** The only tools you have are **timeouts** (to decide when to stop waiting) and **idempotency** (to make retrying safe).

---

## 8.2 Replication

Copies of data on several nodes give you availability and read scale.

| Model | How | Trade-off |
|---|---|---|
| **Single leader** (PostgreSQL, SQL Server primary + replicas) | writes go to the leader; followers replicate | simple; the leader is a bottleneck; failover takes care |
| **Multi-leader** (multi-region writes) | several nodes accept writes | write conflicts need resolution |
| **Leaderless** (Cassandra, DynamoDB-style) | clients write and read several replicas (quorums) | tunable consistency; complex semantics |

### Replication lag anomalies (single leader with async replicas)

| Anomaly | Example | Fix |
|---|---|---|
| **Read-your-writes violation** | the user updates their profile, reloads, and sees the old data (served by a lagging replica) | read from the primary for the user's own recent writes |
| **Monotonic reads violation** | refreshing shows a new comment, then it disappears (a different, more stale replica) | stick a user's reads to one replica |
| **Consistent prefix violation** | you see an answer before the question | causal ordering within a partition |

> In .NET apps this shows up as "a read model, cache or replica says X, but we just wrote Y". Choose deliberately which reads may be stale.

---

## 8.3 Partitioning (sharding)

Splitting data across nodes lets you scale writes and storage.

- **By key range:** efficient range queries, but **hot spots** (all of today's orders land on one shard).
- **By hash of the key:** even distribution, but range queries scatter across shards.
- **Hot keys** (a celebrity user, a sold-out concert) break even distribution. Split the key or add a cache.
- Cross-partition operations (joins, transactions) become expensive. This is the same lesson as "a database per service".

---

## 8.4 CAP and PACELC, stated precisely

**CAP:** during a **network partition**, a system must choose between

- **Consistency** (linearizability: every read sees the latest write), by refusing some requests, or
- **Availability**: answering every request, possibly with stale data.

It's **not** "pick 2 of 3 at all times". Partitions aren't optional, so the choice only matters *during* a partition.

**PACELC** adds the everyday case: **if Partition**, choose A or C; **Else**, choose **Latency or Consistency**. Even without failures, strong consistency costs latency (coordination), and that's the trade-off you face daily.

| System | Typical stance |
|---|---|
| Single-node PostgreSQL/SQL Server | consistent; unavailable if the node is down |
| Azure Cosmos DB | **tunable**: strong, bounded staleness, session (default), consistent prefix, eventual |
| DNS, CDN caches | available and eventually consistent |
| etcd / ZooKeeper / Consul | consistent (consensus); minority partitions become unavailable |

---

## 8.5 Consistency models: a ladder

From strongest to weakest (simplified):

1. **Linearizable:** behaves like a single copy; every read sees the latest completed write. Expensive.
2. **Sequential / causal:** operations that are causally related are seen in order by everyone.
3. **Session guarantees:** read-your-writes and monotonic reads, *for one client*. Often the sweet spot (Cosmos DB's default).
4. **Eventual:** if writes stop, all replicas converge… eventually. No ordering promises in between.

**In practice:** use strong consistency for money, inventory and seat allocation. Use session consistency for user-facing reads. Use eventual consistency for analytics, search indexes, recommendations and counters.

---

## 8.6 Transactions across nodes

- **ACID** within one database is cheap and reliable. Use it.
- **Two-phase commit (2PC)** coordinates several resources: *prepare* → *commit*. If the coordinator dies after prepare, participants **block** holding locks. That's poor for availability, and it's unsupported by most brokers and cloud services.
- So distributed business processes use **sagas** with compensation ([guide 6](06-outbox-inbox-sagas.md)), accepting temporary inconsistency in exchange for availability.

---

## 8.7 Time and clocks

- **Wall clocks drift and jump.** NTP adjusts them, sometimes backward. VMs pause. Two machines' clocks can differ by tens of milliseconds or much more.
- **Never order events from different machines by `DateTime`.** "Last write wins" by timestamp **silently loses data** when clocks are skewed.
- Use **monotonic clocks** (`Stopwatch`, `TimeProvider.GetTimestamp()`) for measuring durations, never `DateTime.Now` subtraction.
- **Logical clocks:** Lamport timestamps give a total order consistent with causality. Version vectors detect concurrent updates. In application code, a **version number per aggregate** (optimistic concurrency) is the everyday form of this idea.

---

## 8.8 Distributed locks: the dangerous convenience

A typical Redis lock: `SET resource my-token NX PX 30000`. "I hold the lock for 30 s."

**The failure:** process A takes the lock, then pauses (a GC pause, a VM stall, a slow network) for 40 s. The lock expires, process B takes it, and **both A and B now believe they hold the lock** and write to the resource.

**The fix: fencing tokens.** The lock service returns a monotonically increasing token with each grant, and the **resource itself rejects writes with an older token** (`UPDATE … WHERE last_token < @token`). Without the resource checking the token, the lock is only an optimization.

| Need | Recommendation |
|---|---|
| **Efficiency** (avoid duplicate work; occasional duplicates are OK) | a Redis lock is fine |
| **Correctness** (double booking, double charge must never happen) | **database constraints, conditional updates, optimistic concurrency**, or a lock with fencing that the resource enforces |

> This is why the seat-hold design in Phase 2 relies on database concurrency tokens and not on a cache lock.

---

## 8.9 Consensus, in one paragraph

Algorithms such as **Raft** and **Paxos** let a group of nodes agree on a value (a leader, a log entry) despite failures, as long as a **majority** is up. They power etcd (Kubernetes' brain), Consul and ZooKeeper. **You almost never implement consensus.** You *use* systems built on it for leader election, configuration and locks with fencing. Know that "majority" is why clusters have 3 or 5 nodes, and why a 2-node cluster can't survive one failure safely.

---

## 8.10 Back-pressure and load shedding

When demand exceeds capacity, something has to give. Choose *what*:

- **Bounded queues** (bounded `Channel<T>`, broker queue limits) push back on producers instead of growing until memory runs out.
- **Load shedding:** reject early (`429`/`503` from rate limiters) rather than accepting work you can't finish in time.
- **Prioritization:** keep checkout working and shed recommendations.
- **Deadline propagation:** pass the remaining time budget downstream (a `CancellationToken` with a timeout), so nobody works on a request whose caller already gave up.

---

## 8.11 The end-to-end argument

Reliability guarantees in lower layers (TCP retransmission, broker "exactly-once") **don't remove the need for end-to-end checks** in the application. Only the endpoints know what "done correctly once" means. That's why **idempotency keys and deduplication at the application level** remain necessary no matter what the infrastructure promises.

---

## Interview questions

<details><summary><b>1. Explain CAP correctly.</b></summary>

During a network partition, a distributed data system must either refuse some requests to stay consistent (linearizable), or answer them all and risk stale data. Without a partition there's no forced choice. PACELC adds that, even then, you trade latency for consistency.
</details>

<details><summary><b>2. What's read-your-writes consistency, and how do you provide it with read replicas?</b></summary>

A user always sees their own completed writes. Route that user's reads to the primary for a period after they write, track the replication position (LSN) and wait for a replica to catch up, or read from the primary for user-owned data.
</details>

<details><summary><b>3. Why are distributed locks with expiry unsafe on their own?</b></summary>

A paused or slow lock holder can outlive its lease while another process acquires the lock, so two processes act simultaneously. Correctness needs fencing tokens that the protected resource checks, or a resource-level guard (constraints, conditional updates).
</details>

<details><summary><b>4. Why not order events by timestamp across services?</b></summary>

Clocks drift, jump and differ between machines, so timestamps don't reflect causality, and last-write-wins silently drops updates. Use per-entity versions, sequence numbers from one source or logical clocks.
</details>

<details><summary><b>5. What happens when a client times out waiting for a response?</b></summary>

The operation may have failed, may still be running or may have succeeded with the response lost. The client can't know. So the operation should be idempotent (or carry an idempotency key), and the client can safely retry or query the status.
</details>

---

## Exercises

1. Read the DDIA chapters on replication, partitioning, transactions, distributed-systems trouble, and consistency and consensus. Write a one-page summary per chapter in your own words.
2. Simulate the GC-pause lock bug and fix it with fencing tokens ([Lab 10](10-labs.md#lab-10-distributed-lock-pitfalls)).
3. With a PostgreSQL primary and a streaming replica in Docker, create a read-your-writes violation and fix it.
4. For each feature of your product at work, decide the consistency it needs (strong, session or eventual) and justify it.

## Self-check

- [ ] I can explain partial failure and why timeouts plus idempotency are the basic tools.
- [ ] I can state CAP/PACELC precisely and place real systems on them.
- [ ] I know the replication-lag anomalies and their fixes.
- [ ] I can explain why distributed locks need fencing, and what to use instead for correctness.
- [ ] I never order cross-machine events by wall-clock time.

## Further reading

- *Designing Data-Intensive Applications* (Martin Kleppmann): the core reading for this guide.
- Martin Kleppmann, *"How to do distributed locking"* (blog post).
- Werner Vogels, *"Eventually Consistent"* (article).
- The Raft visualization at raft.github.io.
