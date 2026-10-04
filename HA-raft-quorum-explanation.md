# etcd, Raft, Quorum & Consensus — A Practical Guide

---

## Table of Contents

1. [Why this document exists](#1-why-this-document-exists)
2. [What etcd actually is](#2-what-etcd-actually-is)
3. [The problem: keeping multiple machines in sync](#3-the-problem-keeping-multiple-machines-in-sync)
4. [The log vs the state — the single most important distinction](#4-the-log-vs-the-state--the-single-most-important-distinction)
5. [Watching one write happen, step by step](#5-watching-one-write-happen-step-by-step)
6. [What "committed" really means](#6-what-committed-really-means)
7. [Quorum — the minimum majority](#7-quorum--the-minimum-majority)
8. [Raft vs Quorum vs Consensus — clearing up the terms](#8-raft-vs-quorum-vs-consensus--clearing-up-the-terms)
9. [Leader election — how a leader is chosen](#9-leader-election--how-a-leader-is-chosen)
10. [Heartbeats, timeouts, and terms](#10-heartbeats-timeouts-and-terms)
11. [What prevents two leaders at the same time](#11-what-prevents-two-leaders-at-the-same-time)
12. [Split votes and how they resolve](#12-split-votes-and-how-they-resolve)
13. [Failure scenarios walkthrough](#13-failure-scenarios-walkthrough)
14. [Why 2 etcd nodes is a bad idea](#14-why-2-etcd-nodes-is-a-bad-idea)
15. [How this maps to Kubernetes HA](#15-how-this-maps-to-kubernetes-ha)
16. [Common misconceptions](#16-common-misconceptions)
17. [Quick reference cheat sheet](#17-quick-reference-cheat-sheet)

---

## 1. Why this document exists

Most explanations of etcd, Raft, and consensus fall into one of two traps:

- **Too abstract** — "Raft is a consensus algorithm that guarantees linearizability" (helpful to nobody who is trying to understand what actually happens when you `kubectl apply`).
- **Too shallow** — "etcd replicates data to all nodes" (which is wrong in an important way, and builds a mental model you have to unlearn later).

This document takes a third path: we walk through **one single write**, in slow motion, on a 3-node etcd cluster, and connect every concept to that concrete example. By the end, you'll understand the log, the commit, the apply, elections, quorum, and why certain cluster sizes are recommended.

---

## 2. What etcd actually is

etcd is a **distributed key-value store**. In Kubernetes, it's the single source of truth for the entire cluster state:

- Which nodes exist
- Which pods are scheduled where
- What deployments, services, configmaps, secrets exist
- What the desired replica count is

When you run `kubectl apply`, the API server writes to etcd. When the scheduler decides where a pod goes, it writes to etcd. When a controller reconciles state, it reads from and writes to etcd.

So when we say "the cluster state," we mean "what etcd currently holds."

### The architecture

A production etcd deployment is almost always an **odd-numbered cluster** of 3, 5, or 7 members:

```
        etcd1
       /     \
      /       \
   etcd2 --- etcd3
```

Each member is a full copy of etcd running on a separate machine. They communicate over the network to stay in sync.

But here's the crucial thing:

> They are not three independent databases that happen to share data.
> They are **one logical cluster** made of three physical members.

The software that makes them behave as one cluster is called **Raft**.

---

## 3. The problem: keeping multiple machines in sync

Imagine you have three servers, each holding a copy of the same key-value data. A client sends a write:

```
set nginx replicas = 5
```

The naive approach is:

1. Leader changes its own state to `replicas=5`
2. Leader tells the followers "my state is now `replicas=5`, copy it"

This breaks in dozens of ways:

- What if two servers both think they're the leader and both accept writes?
- What if a follower receives the state update but the leader dies before telling the others?
- What if the network partitions and two halves of the cluster both think they have the latest state?
- How does a node that was offline for an hour catch up?

The real problem is not "how do we copy data." The real problem is:

> **How do multiple machines agree on a single, consistent history of operations — even when some of them fail?**

That's the consensus problem. Raft is one solution to it.

---

## 4. The log vs the state — the single most important distinction

Before we go any further, you need to separate two things that beginners almost always blur together:

### The state

The state is the **current snapshot of the data**.

```
nginx replicas = 3
service nginx exists
namespace default exists
```

If you query etcd right now and ask "what are the replicas of nginx?", the state gives you the answer.

### The log

The log is the **ordered history of operations that produced the state**.

```
100 → create namespace default
101 → create deployment nginx
102 → set nginx replicas = 3
```

If you replay the log from the beginning, you arrive at the state.

### The relationship

```
        LOG                    STATE
   (history of ops)      (current values)

   100 create ns
   101 create nginx   ──apply──▶   nginx exists
   102 replicas = 3   ──apply──▶   replicas = 3
```

The state is **derived** from the log. The log is the source of truth.

This matters because:

- Two nodes can have the same state but arrived via different logs (bad — we can't tell which is correct)
- Two nodes with the same log **must** have the same state (good — this is what Raft guarantees)

> **Raft's entire job is to make the members agree on the ordered log.**
> The state is just what you get when you apply that log locally.

This distinction is the foundation. Everything else in this document builds on it.

---

## 5. Watching one write happen, step by step

Let's watch a single `kubectl scale` request flow through the entire cluster.

### Initial situation

```
        etcd1
       LEADER
      /      \
     ↓        ↓
  etcd2     etcd3
 follower  follower
```

Current log on all three:

```
100 → create namespace
101 → create deployment nginx
102 → set replicas = 3
```

Current state on all three:

```
nginx replicas = 3
```

Now you run:

```bash
kubectl scale deployment nginx --replicas=5
```

---

### Step 1 — The request arrives at the leader

The Kubernetes API server talks to etcd. The write lands on **etcd1** because etcd1 is the leader.

```
kubectl
   │
   │ "scale nginx to 5"
   ↓
API server
   │
   ↓
etcd1 (leader)
```

At this instant:

- The leader has **received** the request.
- Nothing has been written anywhere yet.
- State is still `replicas=3` everywhere.

---

### Step 2 — The leader creates a log entry

The leader does **not** immediately change its state.

Instead, it appends a new entry to its **log**:

```
Entry 103:
    set nginx replicas = 5
```

Now:

```
etcd1 (leader)          etcd2 (follower)        etcd3 (follower)

LOG:                    LOG:                    LOG:
100                     100                     100
101                     101                     101
102                     102                     102
103 ← new               (nothing)               (nothing)

STATE:                  STATE:                  STATE:
replicas=3              replicas=3              replicas=3
```

**Important:** the state on the leader is still `replicas=3`. The log changed. The state did not.

The leader has written down the **intention**, not the result.

---

### Step 3 — The leader replicates the log entry

The leader sends Entry 103 to the followers:

```
              etcd1 (leader)
                 │
        "Here is Entry 103"
           ┌─────┴─────┐
           ↓           ↓
        etcd2       etcd3
```

Suppose both receive it and store it durably:

```
etcd1 (leader)          etcd2 (follower)        etcd3 (follower)

LOG:                    LOG:                    LOG:
100                     100                     100
101                     101                     101
102                     102                     102
103                     103                     103

STATE:                  STATE:                  STATE:
replicas=3              replicas=3              replicas=3
```

Now all three have the **log entry**. But **none** of them has applied it to state yet.

- Log replicated? **Yes.**
- Committed? **Not yet.**
- State changed? **No.**

---

### Step 4 — Majority reached, entry committed

Quorum for 3 members is **2**. Who has Entry 103?

```
etcd1 ✓
etcd2 ✓
etcd3 ✓
```

That's 3/3. Or even 2/3 would be enough. Either way, a majority has it.

The leader can now **advance the commit point**:

```
Entry 103 is COMMITTED.
```

Committed means:

> "A majority has this entry stored durably. It is now part of the authoritative history. It will survive the failure of a minority."

Now:

```
etcd1 (leader)          etcd2 (follower)        etcd3 (follower)

LOG:                    LOG:                    LOG:
100                     100                     100
101                     101                     101
102                     102                     102
103 ✓ committed         103 ✓ committed         103 ✓ committed

STATE:                  STATE:                  STATE:
replicas=3              replicas=3              replicas=3
```

Notice: **still no state change anywhere.** Only the commit point moved.

---

### Step 5 — The committed entry is applied

Now each member applies Entry 103 to its local state machine:

```
Entry 103: set nginx replicas = 5

STATE before: replicas = 3
STATE after:  replicas = 5
```

Result:

```
etcd1 (leader)          etcd2 (follower)        etcd3 (follower)

LOG:                    LOG:                    LOG:
100                     100                     100
101                     101                     101
102                     102                     102
103 committed           103 committed           103 committed

STATE:                  STATE:                  STATE:
replicas=5              replicas=5              replicas=5
```

Now the state has changed. Everywhere.

And it changed **because** everyone applied the same committed log entry — not because the leader pushed its state to the followers.

---

### The complete lifecycle

```
kubectl scale nginx --replicas=5
              │
              ↓
        API server
              │
              ↓
        etcd1 LEADER
              │
              │ 1. receive request
              │ 2. create log entry
              ↓
     ┌──────────────────┐
     │  Entry 103       │
     │  replicas = 5    │
     └────────┬─────────┘
              │
              │ 3. replicate
       ┌──────┴──────┐
       ↓             ↓
    etcd2         etcd3
    stored ✓      stored ✓
       │             │
       └──────┬──────┘
              ↓
        4. MAJORITY
              ↓
        5. COMMITTED
              ↓
        6. APPLY
              ↓
        7. STATE = replicas 5
```

### The five separate moments

| Moment | What changed |
|---|---|
| 1. Leader receives request | Nothing yet |
| 2. Leader creates log entry | **Log** on leader |
| 3. Leader replicates | **Log** on followers |
| 4. Majority reached | **Commit point** |
| 5. Apply | **State** on each member |

Log, commit, and state are three different things, and they change at three different times.

---

## 6. What "committed" really means

This is the term that trips people up most, so let's be precise.

**Committed does NOT mean:**

- "Every node has the entry"
- "Every node has applied the entry"
- "The state has changed everywhere"
- "The cluster voted on whether the change is correct"

**Committed means:**

> A majority of members have durably stored the entry, so it is now part of the cluster's authoritative history and will survive the failure of any minority.

### Why durability matters

If a majority has the entry stored **on disk**, then even if those nodes crash and restart, the entry will still be there. It cannot be lost as long as a majority remains available.

This is what makes committed history safe across:

- Leader crashes
- Follower crashes
- Network partitions
- Power failures
- Full cluster restarts

### The commit index

Each member tracks a number called the **commit index** — the highest log entry that has been committed.

```
LOG:
100  ✓ committed
101  ✓ committed
102  ✓ committed
103  ✓ committed  ← commit index = 103
104  (not committed yet)
105  (not committed yet)
```

Entries below the commit index are safe. Entries above are still in flight.

When the leader learns that a majority has stored entry 103, it advances its commit index to 103, and followers learn about this through subsequent heartbeat messages.

### Applied vs committed

There is one more subtle distinction:

- **Committed** — the entry is safely part of history
- **Applied** — the entry has been fed into the state machine

Normally these happen back-to-back, but a follower that's catching up might have committed entries it hasn't applied yet. The **apply index** (or `lastApplied`) tracks this.

```
commit index  = 103   (history safely ends here)
apply index   = 102   (state currently reflects this)
```

The gap between them is small and closes quickly. But conceptually they're separate.

---

## 7. Quorum — the minimum majority

Quorum is the minimum number of members that must participate for the cluster to make progress.

### The formula

```
quorum = floor(members / 2) + 1
```

In plain terms: **a strict majority**.

| Members | Quorum | Can tolerate |
|---|---|---|
| 1 | 1 | 0 failures |
| 2 | 2 | 0 failures |
| 3 | 2 | 1 failure |
| 4 | 3 | 1 failure |
| 5 | 3 | 2 failures |
| 6 | 4 | 2 failures |
| 7 | 4 | 3 failures |

### Why majority, not "all"?

Because requiring all members means any single failure stops the cluster.

Requiring a majority means:

- A minority can fail without stopping progress
- Two majorities must always overlap, which prevents two conflicting decisions

That last point is the deep reason quorum works. Let's look at it.

### The majority intersection property

With 3 members, any two majorities overlap:

```
Majority A: {etcd1, etcd2}
Majority B: {etcd2, etcd3}
Overlap:    {etcd2}
```

With 5 members:

```
Majority A: {etcd1, etcd2, etcd3}
Majority B: {etcd3, etcd4, etcd5}
Overlap:    {etcd3}
```

Two majorities **always** share at least one member. That member acts as a witness — it has seen both decisions, so they can't contradict each other.

This is why Raft can guarantee safety with only a majority: even if the leader dies and a new leader is elected, the new leader's election required a majority, and that majority must overlap with the majority that stored the committed entries. So the new leader is guaranteed to have the committed history.

### "No quorum" means "stop, don't guess"

If the cluster cannot reach quorum, it **stops accepting writes**.

This is intentional. It is far better to stop than to risk two diverging histories. Raft chooses **consistency over availability** in this situation — this is what makes it a CP system in CAP theorem terms.

---

## 8. Raft vs Quorum vs Consensus — clearing up the terms

These three words get used interchangeably in casual conversation, but they mean different things.

| Term | What it is |
|---|---|
| **Raft** | The protocol/algorithm that coordinates the cluster |
| **Quorum** | The minimum majority required to make decisions |
| **Consensus** | The actual agreement on the ordered log |
| **Leader** | The Raft member that coordinates writes |
| **Follower** | A Raft member that follows the leader |
| **Candidate** | A Raft member currently trying to become leader |

### In one sentence each

- **Raft** is the mechanism: "how do the nodes coordinate?"
- **Quorum** is the requirement: "how many nodes must participate?"
- **Consensus** is the result: "the agreement on the log."

### A parliament analogy

Imagine 5 people making decisions, and any decision requires at least 3 votes.

- **Raft** = the rules of parliamentary procedure (how debates happen, how votes are called, how the speaker is chosen)
- **Quorum** = the rule that at least 3 people must be present
- **Consensus** = the decision that actually gets made

You can't have consensus without quorum. You can't reach quorum without some protocol to coordinate. Raft provides that protocol.

### A common confusion

People sometimes say "the nodes vote on whether to commit a change." This is misleading.

Nodes do **not** vote on whether a change is correct. They don't evaluate the content.

Nodes **do** vote in two situations:

1. **Leader election** — "who should be leader for this term?"
2. **Log replication (implicitly)** — "have you stored this entry?" (acknowledgment)

When you see "vote" in Raft, think "acknowledgment of log entries" or "election vote." Never "opinion on whether the change is good."

---

## 9. Leader election — how a leader is chosen

So far we've assumed there's a leader. But how does a leader get chosen in the first place?

### The three states

Every member is in exactly one of three states at any moment:

```
FOLLOWER   — passive, listens to the leader
CANDIDATE  — wants to become leader, asking for votes
LEADER     — the coordinator
```

### The big picture

```
Leader dies
     │
     ↓
Heartbeats stop
     │
     ↓
Some follower's election timer fires first
     │
     ↓
That follower becomes CANDIDATE
     │
     │ increments term
     │ votes for itself
     │ requests votes from others
     ↓
Other followers VOTE (they don't campaign)
     │
     ↓
Candidate counts votes
     │
   ┌─┴─┐
majority  no majority
   │        │
   ↓        ↓
LEADER   split vote → retry in next term
```

### What triggers an election

Followers expect to hear from the leader regularly. Specifically, they expect **heartbeats** — periodic "I'm still alive" messages.

Each follower has an **election timeout** — a countdown timer. Every time a valid heartbeat arrives, the timer resets.

If the timer reaches zero (no heartbeat arrived in time), the follower assumes the leader is dead and starts an election.

### The three rules of voting

Raft's voting has exactly three rules:

1. **A member votes at most once per term.** Once you vote for candidate X in term 5, you cannot vote for anyone else in term 5.
2. **A candidate needs a majority to win.** With 3 members, that's 2 votes. With 5, that's 3.
3. **A member only votes for a candidate whose log is at least as up-to-date as its own.** This prevents a stale node from becoming leader and losing committed entries.

These three rules together guarantee that at most one leader is elected per term.

### The election, step by step

Suppose the leader dies:

```
BEFORE:
etcd1 = LEADER
etcd2 = FOLLOWER  (election timeout 150ms)
etcd3 = FOLLOWER  (election timeout 200ms)

etcd1 💀
```

**Step 1 — Heartbeats stop.**

```
etcd2: timer running...
etcd3: timer running...
```

**Step 2 — The first timer fires (etcd2, because 150 < 200).**

```
etcd2: "No heartbeat. I'm becoming a CANDIDATE."
       → term: 1 → 2
       → votes for itself
       → sends RequestVote(term=2) to etcd3
```

At this moment:

```
etcd2 = CANDIDATE (term 2)
etcd3 = FOLLOWER  (term 1, timer still running)
```

**Step 3 — etcd3 receives the vote request.**

etcd3 checks three things:

```
1. Is the candidate's term >= mine?     2 >= 1 ✓
2. Have I voted in term 2?              No ✓
3. Is the candidate's log up-to-date?   Yes ✓
```

All pass. etcd3 grants the vote:

```
etcd3 → etcd2: "I vote for you in term 2"
```

**Step 4 — etcd2 counts votes.**

```
etcd2 (self-vote): 1
etcd3 (granted):   1
Total:             2

Quorum = 2 → win!
```

**Step 5 — etcd2 becomes leader and sends heartbeats.**

```
etcd2 = LEADER (term 2)
etcd3 = FOLLOWER (term 2)

etcd2 → etcd3: heartbeat(term 2)
etcd3: "Leader alive. Reset timer."
```

Election complete.

### Why randomization matters

If all followers had the same election timeout, they would all fire at the same moment and all become candidates — a split vote (see section 12).

So Raft **randomizes** the election timeout within a range:

```
etcd2: 150ms
etcd3: 200ms
```

Now one of them usually fires first and wins. The randomized gap is the mechanism that breaks the symmetry and lets elections complete quickly.

---

## 10. Heartbeats, timeouts, and terms

Three timing-related concepts glue the whole protocol together.

### Heartbeats

The leader periodically sends an empty `AppendEntries` message to every follower. This is called a **heartbeat**.

```
etcd1 (leader)
   │
   │ heartbeat every ~50–100ms
   ├──────────────→ etcd2
   └──────────────→ etcd3
```

Its only purpose is to say: "I'm still alive. Don't start an election."

### Election timeout

Each follower runs a countdown timer. Any valid heartbeat resets it. If it reaches zero, the follower starts an election.

Typical values:

```
heartbeat interval:  ~50–100 ms
election timeout:    ~150–300 ms (randomized per node)
```

The election timeout is always significantly larger than the heartbeat interval. That way:

- A few missed heartbeats don't trigger a false election
- A truly dead leader is detected within a few hundred milliseconds

### Terms

Time in Raft is divided into **terms** — numbered periods, each starting with an election.

```
Term 1     Term 2     Term 3     Term 4
────────   ────────   ────────   ────────
etcd1      etcd2      etcd3      etcd2
leader     leader     leader     leader
```

**Each term has at most one leader.** The term number increments after every election.

Every message in Raft carries the sender's term. If a member receives a message with a higher term than its own, it immediately updates its term and (if it was leader or candidate) steps down to follower.

This is how a stale leader is removed. If an old leader sends a heartbeat with term 1 while the cluster is already on term 2, everyone ignores it:

```
stale etcd1 (term 1): heartbeat(1)
etcd2 (term 2):       "Your term is old. Ignored."
etcd1:                "Oh. I'm no longer leader."
                      → steps down to follower
```

### The timing relationship

```
heartbeat interval     ≈ 50–100 ms
election timeout       ≈ 150–300 ms
```

The election timeout is at least 3× the heartbeat interval, giving plenty of margin for network jitter.

---

## 11. What prevents two leaders at the same time

This is the safety question. Two leaders accepting writes would be catastrophic — diverging logs, conflicting state.

Raft prevents this with three rules:

### Rule 1 — One vote per member per term

```
etcd2: "Vote for me in term 2"
etcd3: "OK, I voted for etcd2 in term 2"

etcd2: "Vote for me in term 2"  (retry, same term)
etcd3: "No, I already voted in term 2"
```

Once etcd3 votes, it cannot vote again in the same term. So two candidates cannot both receive etcd3's vote in term 2.

### Rule 2 — A leader needs a majority

With 3 members, quorum is 2:

```
etcd2 needs: itself + one more = 2 votes
etcd3 needs: itself + one more = 2 votes
```

Can both get 2 votes? No. There are only 3 members. If etcd2 got 2 votes (itself + etcd3), then etcd3 already voted for etcd2, so etcd3 can't vote for itself. Two majorities would have to overlap, and the overlapping member can only vote once.

### Rule 3 — Higher term wins

If an old leader from term 1 sends a message while the cluster is on term 2:

```
stale leader (term 1): "I'm the leader!"
current members (term 2): "Your term is old. Ignored."
```

Followers reject messages from lower terms. So a stale leader is neutralized automatically.

### Combined result

These three rules guarantee: **at most one leader per term**.

And because a new term's election requires a majority, and any two majorities overlap, the new leader inherits all committed entries from the previous term. The committed history is preserved across leadership changes.

---

## 12. Split votes and how they resolve

Sometimes the randomization isn't enough. Two candidates campaign in the same term.

### The scenario

```
etcd2 timer expires at t=150ms → CANDIDATE (term 2)
etcd3 timer expires at t=151ms → CANDIDATE (term 2)
```

Both vote for themselves:

```
etcd2 votes for itself: 1 vote
etcd3 votes for itself: 1 vote
```

Then:

```
etcd2 → etcd3: "Vote for me in term 2"
etcd3 → etcd2: "Vote for me in term 2"
```

But each has already voted — for themselves. So:

```
etcd2: "I already voted in term 2. Can't vote for etcd3."
etcd3: "I already voted in term 2. Can't vote for etcd2."
```

Result:

```
etcd2: 1 vote
etcd3: 1 vote
Quorum = 2 → nobody wins
```

This is a **split vote**. No leader in term 2.

### How it resolves

Each candidate times out again, with a **fresh randomized timeout**:

```
etcd2: new timeout 180ms
etcd3: new timeout 220ms
```

Now etcd2 fires first:

```
etcd2: CANDIDATE (term 3)
       → votes for itself
       → RequestVote(term=3) → etcd3
```

etcd3:

```
etcd3: "term 3 > my term 2. I haven't voted in term 3. OK."
```

etcd2 wins with 2 votes. Election resolves.

**The key insight:** split votes are self-correcting because randomization eventually breaks the symmetry. In practice, split votes are rare and resolve within one or two rounds.

---

## 13. Failure scenarios walkthrough

Let's see the system handle various failures.

### Scenario A — A follower dies

```
etcd1 (leader)   etcd2 (follower)   etcd3 💀
```

Quorum is still 2 (etcd1 + etcd2). The cluster keeps operating normally. etcd3 catches up when it returns.

### Scenario B — The leader dies

```
etcd1 💀   etcd2 (follower)   etcd3 (follower)
```

Heartbeats stop. A follower's election timer fires. A new leader is elected from {etcd2, etcd3}. The committed history is preserved because the new leader's majority overlaps with the previous majority.

### Scenario C — A follower falls behind

Suppose etcd3 was offline for a while. The cluster committed entries 103 and 104 without it.

```
etcd1: 100 101 102 103 104
etcd2: 100 101 102 103 104
etcd3: 100 101 102  (behind)
```

When etcd3 comes back, the leader sends it the missing entries. etcd3 catches up:

```
etcd3: 100 101 102 103 104
```

It then applies them, and its state matches. This is why the log is critical — you can repair a lagging node by sending missing log entries, not by copying state.

### Scenario D — Network partition

The cluster splits into two groups:

```
Group A: etcd1, etcd2   (majority)
Group B: etcd3          (minority)
```

**Group A** has quorum (2/3). It continues operating, electing a leader if needed, committing writes.

**Group B** has only 1/3. It cannot reach quorum. It cannot commit writes. It cannot elect a leader. It sits idle, waiting to be reconnected.

When the partition heals, Group B catches up from Group A's log.

### Scenario E — Two partitions of equal size (4-node cluster)

With 4 nodes:

```
Group A: etcd1, etcd2  (2/4)
Group B: etcd3, etcd4  (2/4)
```

Quorum is 3. Neither group has quorum. Both sides stop. The cluster is unavailable until the partition heals.

This is another reason 4-node clusters are discouraged. Odd numbers avoid ties.

### Scenario F — The write is committed but the leader dies before applying

Suppose:

```
Entry 103 committed (majority has it)
etcd1 (leader) 💀 before applying it
```

etcd2 and etcd3 still have Entry 103 in their logs. They elect a new leader. The new leader has Entry 103. It will apply it and eventually everything converges.

The write is **not lost**, because "committed" means a majority has it durably stored — not that anyone has applied it yet.

---

## 14. Why 2 etcd nodes is a bad idea

Let's count carefully.

### 2-node cluster

```
etcd1   etcd2

Quorum = floor(2/2) + 1 = 2
```

That means **both** must be alive for anything to work.

If one dies:

```
etcd1 💀
etcd2
   → 1/2 alive
   → no quorum
   → cluster stops
```

A 2-node cluster has **zero fault tolerance**. It's strictly worse than a single-node cluster, because a single node at least keeps working while it's alive.

### 3-node cluster

```
etcd1   etcd2   etcd3

Quorum = floor(3/2) + 1 = 2
```

If one dies:

```
etcd2   etcd3
   → 2/3 alive
   → quorum ✓
   → cluster keeps working
```

3-node clusters tolerate 1 failure.

### Why odd numbers win

| Members | Quorum | Tolerated failures |
|---|---|---|
| 2 | 2 | 0 |
| 3 | 2 | 1 |
| 4 | 3 | 1 |
| 5 | 3 | 2 |
| 6 | 4 | 2 |
| 7 | 4 | 3 |

Notice: 4 nodes tolerate the same number of failures as 3, but with one extra machine to pay for. 6 tolerates the same as 5. **Even numbers are a waste.**

The rule: **use odd numbers, typically 3 or 5.**

- 3 for most production clusters
- 5 for very large or geographically distributed clusters

Beyond 5, replication cost grows faster than the reliability gain, and network latency between members starts to dominate.

### Why even numbers are especially bad

Beyond wasted resources, even-numbered clusters are vulnerable to **50/50 splits**:

```
4-node cluster, partition:
Group A: 2 nodes   (2/4)
Group B: 2 nodes   (2/4)
Quorum = 3 → neither side can proceed
```

With 3 nodes, a partition can only be 2/1 or 1/2 — one side always has quorum.

---

## 15. How this maps to Kubernetes HA

A production HA Kubernetes control plane looks like this:

```
                  kubectl
                     │
                     ↓
              Load Balancer
              /     |      \
             ↓      ↓       ↓
           CP1     CP2     CP3
            │       │       │
          etcd1   etcd2   etcd3
            │       │       │
            └───────┼───────┘
                    │
                 Raft
                    │
              quorum = 2
```

Each control-plane node runs:

- kube-apiserver
- kube-controller-manager
- kube-scheduler
- etcd

The etcd members form a Raft cluster. Writes go through the etcd leader. Committed writes are applied to each member's local state.

### Failure tolerance in HA

| etcd members | Tolerate | Cluster survives |
|---|---|---|
| 3 | 1 failure | 1 node down |
| 5 | 2 failures | 2 nodes down |

With 3 etcd members:

- 1 node down → still healthy
- 2 nodes down → no quorum, cluster stops accepting writes

### The recommended sizing

- **Development / small clusters:** 3 control-plane nodes, 3 etcd members
- **Production:** 3 or 5 control-plane nodes, matching etcd members
- **Never:** 2 or 4 etcd members

### What "cluster unavailable" means

If etcd loses quorum:

- The API server can't write
- New pods can't be scheduled
- Deployments can't be updated
- Existing pods keep running, but nothing can be changed

This is the "etcd quorum lost" failure mode that brings down an HA cluster. It's why quorum preservation is the #1 operational priority for etcd.

---

## 16. Common misconceptions

### Misconception 1 — "The leader changes state and pushes it to followers"

**Reality:** The leader creates a log entry, replicates the entry, waits for a majority, commits, and then everyone applies the committed entry. The state is never directly replicated.

### Misconception 2 — "Followers vote on whether a change is correct"

**Reality:** Followers acknowledge that they've stored the log entry. They don't evaluate the content. The content is decided by the client (Kubernetes). Raft only decides whether it's part of the ordered history.

### Misconception 3 — "Committed means every node has the entry"

**Reality:** Committed means a majority has the entry durably stored. A lagging minority can catch up later.

### Misconception 4 — "You need all nodes for the cluster to work"

**Reality:** You need a majority. A minority can be offline, slow, or partitioned, and the cluster keeps working.

### Misconception 5 — "2-node etcd is more HA than 1-node"

**Reality:** 2-node etcd has zero fault tolerance. Lose one, and you lose quorum. 1-node at least keeps working while it's alive.

### Misconception 6 — "Split votes break Raft"

**Reality:** Split votes are a normal part of Raft. They resolve in the next term due to randomized timeouts. No data is lost.

### Misconception 7 — "The leader's state is the truth, followers copy it"

**Reality:** The **log** is the truth. State is derived. The leader is just the member currently allowed to append new entries.

### Misconception 8 — "Consensus means everyone agrees on the value"

**Reality:** Consensus means everyone agrees on the **ordered log**. The state is what you get when you apply that log. Nobody "votes" on whether the state is correct.

### Misconception 9 — "If etcd loses quorum, data is lost"

**Reality:** Data is not lost. The cluster simply stops accepting writes until quorum returns. Committed entries on majority members are still safe.

### Misconception 10 — "More etcd nodes = more reliable"

**Reality:** Beyond 5 nodes, reliability gains are minimal while replication cost and latency grow. 3 or 5 is the sweet spot. Even numbers are strictly worse than the next-lower odd number.

---

## 17. Quick reference cheat sheet

### The write lifecycle

```
REQUEST → LOG ENTRY → REPLICATE → MAJORITY → COMMIT → APPLY → STATE
```

### The distinctions

| Thing | What it is |
|---|---|
| **Log** | Ordered history of operations |
| **Commit** | Majority has the entry; it's safe |
| **Apply** | Entry has been fed into state machine |
| **State** | Current data, derived from applied entries |

### The three states of a member

```
FOLLOWER  →  CANDIDATE  →  LEADER
   ↑                          │
   └──────────────────────────┘
        (higher term seen, or leader dies)
```

### Quorum table

| Members | Quorum | Tolerate |
|---|---|---|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

### Timing

| Parameter | Typical value |
|---|---|
| Heartbeat interval | 50–100 ms |
| Election timeout | 150–300 ms (randomized) |

### Rules of Raft

1. One vote per member per term
2. Majority required to win an election
3. Majority required to commit an entry
4. Higher term always wins
5. Followers only vote for up-to-date candidates
6. At most one leader per term

### Failure cheat sheet

| Failure | Effect |
|---|---|
| 1 follower down (3-node) | Cluster works normally |
| 1 leader down (3-node) | New leader elected within a few hundred ms |
| 2 nodes down (3-node) | No quorum → cluster stops writes |
| Network partition 2/1 (3-node) | Majority side works, minority side stalls |
| Network partition 2/2 (4-node) | Both sides stall |
| Follower behind | Catches up from leader's log when reconnected |
| Leader dies after commit | Write is safe; new leader has it |

### The single most important sentence

> Raft is the protocol that makes the members agree on the ordered log; quorum is the minimum majority required to make progress; and consensus is the agreement on that log. The state is just what you get when each member applies the committed log locally.

---

## Appendix — Further reading

- **The Raft paper:** *In Search of an Understandable Consensus Algorithm* — Diego Ongaro & John Ousterhout (2014). The original, surprisingly readable.
- **The Raft website:** [raft.github.io](https://raft.github.io) — has visualizations and implementations
- **etcd documentation:** [etcd.io/docs](https://etcd.io/docs) — the practical side
- **"Raft lecture" by John Ousterhout** — a video walkthrough, very clear
- **The Secret Lives of Data — Raft visualization** — an interactive explanation at [thesecretlivesofdata.com/raft](http://thesecretlivesofdata.com/raft)

---
