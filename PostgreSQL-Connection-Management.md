# PostgreSQL Connection Management: From Zero to Production Ready

> **What you'll learn:** How PostgreSQL connections really work, how to monitor them, and how to prevent production outages.

## 🎯 Why This Matters

In production, **connection exhaustion is one of the most common PostgreSQL failures**. One bad app can take down ALL your databases. This guide teaches you to see, understand, and prevent that.

> 📌 **Real-world impact:** A single misconfigured app with a connection leak can crash your entire database instance, affecting ALL applications sharing that PostgreSQL server — even ones that were working perfectly.

---

## 🧠 1. Core Idea

In **PostgreSQL**, connections are **NOT per database**.

Instead:

> A connection belongs to the PostgreSQL **server instance**, and it operates inside **one database at a time**.

> A connection is a session created by the PostgreSQL server process, and it is always bound to exactly one database at a time.

> 📌 **What is a "server instance"?**  
> The instance is the running PostgreSQL process on your machine (or container). It manages everything: memory, background processes, authentication, all databases, and connection handling. When PostgreSQL is running, you don't run "a database" — you run ONE PostgreSQL instance that contains many databases.

---

## 🏗️ 2. Mental Model

Think of PostgreSQL like this:

```
PostgreSQL Server (Instance) ← The running process
│
├── Database: auth_system_db
├── Database: scolar_db
├── Database: tracktus_db
```

### 🔌 Connections:

* Live in the **server (instance)**
* "Sit inside" one database at a time
* Can switch databases using `\c`

> 📌 **Hotel Analogy (best way to remember):**
> - **PostgreSQL server = hotel building** (the running instance)
> - **Connection = guest** (enters the building)
> - **Database = room** (where the guest stays)
> 
> You enter the hotel first, then you're placed in a room. You can change rooms (`\c other_db`), but you're still the same guest in the same hotel.

---

## 🔬 3. What Actually Happens When You Run `psql`

This is the **most important example** to understand. Let me trace exactly what happens.

### The Command
```bash
sudo -u postgres psql
```

### What Happens Behind the Scenes (Step by Step)

```
Step 1: psql client starts on your terminal
Step 2: psql reads defaults → user=postgres, database=postgres
Step 3: psql connects to localhost:5432 (THE INSTANCE)
Step 4: PostgreSQL instance accepts the connection
Step 5: Instance authenticates user 'postgres'
Step 6: Instance assigns this connection to database 'postgres'
Step 7: You are now inside the 'postgres' database
```

### Prove It With Commands

```sql
-- This shows which database you're INSIDE
SELECT current_database();
```
**Output:**
```
postgres
```

```sql
-- This shows your connection's process ID (PID)
SELECT pg_backend_pid();
```
**Output:**
```
9693
```

**That PID (9693) is a backend process spawned by the instance JUST for your connection.**

> 📌 **Key Insight:** You NEVER connect directly to a database. You connect to the instance, and the INSTANCE places you inside a database.

---

## 🏗️ 4. What Is a "PostgreSQL Server Instance"?

A **server instance** (also called a **cluster**) is simply:

> **The running PostgreSQL process that manages everything**

### Check It Yourself
```bash
sudo systemctl status postgresql
```

You'll see something like:
```
● postgresql.service - PostgreSQL RDBMS
     Loaded: loaded (/usr/lib/systemd/system/postgresql.service; enabled)
     Active: active (running) since ...  ← THIS IS THE INSTANCE
   Main PID: 1234 (postgres)              ← THIS IS THE INSTANCE PROCESS
```

### What the Instance Includes

| Component | What It Does |
|-----------|--------------|
| Memory (shared_buffers, work_mem) | Stores cached data, sorts queries |
| Background processes | checkpointer, walwriter, autovacuum |
| All your databases | postgres, auth_system_db, scolar_db, etc. |
| Connection listener | Listens on port 5432 |
| Authentication system | Validates users and passwords |
| `max_connections` setting | Global limit across ALL databases |

### What the Instance Does NOT Include

- Individual database files (those are data on disk, not the running process)
- Your application code

### Your Actual Setup Visualized

```
Running Process: postgres (PID 1234) ← THIS IS THE INSTANCE
│
├── Manages: postgres database
├── Manages: auth_system_db database  
├── Manages: scolar_db database
├── Manages: tracktus_db database
│
├── Listens on: port 5432
├── Has max_connections: 100 (GLOBAL limit)
└── Has shared_buffers: 128MB
```

**One instance. Multiple databases. All inside one running process.**

---

## 🏨 5. The Hotel Analogy (Complete Version)

This is the **best way to internalize** how PostgreSQL works.

| PostgreSQL Term | Hotel Analogy |
|----------------|---------------|
| **Instance** | The entire hotel building (running, staffed, operating) |
| **Connection** | A guest who checks in |
| **Database** | The specific room the guest stays in |

### What This Means:

```
You can't enter a room without first entering the hotel
→ You can't use a database without connecting to the instance first

The hotel exists whether rooms are occupied or not
→ The instance runs even with zero connections

One guest stays in one room at a time
→ One connection = one database at a time

Guests can switch rooms
→ You can use \c other_db to switch databases

The hotel has a maximum capacity
→ max_connections limits total guests (connections)

All guests share hotel resources
→ All connections share memory, CPU, and connection slots
```

### Run This Example:

```sql
-- Terminal 1: Check into the hotel
\c auth_system_db
SELECT pg_backend_pid();  -- Your room key (PID)
SELECT current_database(); -- Your room number

-- Same connection, switch rooms
\c scolar_db
SELECT pg_backend_pid();  -- SAME key!
SELECT current_database(); -- DIFFERENT room!
```

---

## 🔬 6. Visual Diagram of Your System

```
┌─────────────────────────────────────────────────────────────┐
│           PostgreSQL INSTANCE (running process)             │
│                                                              │
│  max_connections = 100                                       │
│  port = 5432                                                 │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Connection 1 (PID 9693)                             │   │
│  │  User: postgres │ Database: postgres │ State: idle   │   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  Connection 2 (PID 10344)                            │   │
│  │  User: postgres │ Database: scolar_db │ State: active│   │
│  ├──────────────────────────────────────────────────────┤   │
│  │  Connection 3 (PID 10345)                            │   │
│  │  User: emma │ Database: auth_system_db │ State: idle │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ postgres DB  │ │ scolar_db    │ │ auth_system  │        │
│  │ (default)    │ │              │ │ _db          │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**Each connection is a process INSIDE the instance. Each connection points to ONE database at a time.**

---

## ❓ 7. Your Question Answered Directly

> "When you do `sudo -u postgres psql`, are you not connecting to a database?"

**Yes — you ARE connecting to a database. BUT the connection itself lives at the instance level first.**

### The Two-Step Process:

1. **First:** You connect to the PostgreSQL instance (the running program on port 5432)
2. **Then:** The instance automatically places you inside a specific database (default: `postgres`)

### The Critical Distinction:

| What you connect TO | What you are INSIDE |
|---------------------|---------------------|
| The **instance** (server) | A **database** (context) |

**You never connect to a database without first connecting to the instance.**

---

## 🔥 8. Proof That Connection ≠ Database

Run this in ONE terminal:

```sql
-- Step 1: Get your connection ID
SELECT pg_backend_pid();
-- Output: 9693

-- Step 2: Check which database you're in
SELECT current_database();
-- Output: postgres

-- Step 3: Switch databases (same connection!)
\c scolar_db

-- Step 4: Check your connection ID (SAME!)
SELECT pg_backend_pid();
-- Output: 9693

-- Step 5: Check which database you're in (CHANGED!)
SELECT current_database();
-- Output: scolar_db
```

**Conclusion:** The connection (PID 9693) stayed exactly the same. Only the database context changed.

👉 **This proves:** The connection belongs to the instance, not to a specific database.

---

## 📊 9. The Distinction Table

| Question | Answer |
|----------|--------|
| Is PostgreSQL running? | Yes → that's the **instance** |
| Does the instance manage databases? | Yes — all of them |
| When you connect, what do you connect to? | The **instance** (port 5432) |
| Does the instance put you in a database? | Yes — default or specified |
| Is your connection tied to that database? | Yes — at that moment |
| Can you switch databases without reconnecting? | Yes — `\c other_db` |
| Does the connection count against the limit? | Yes — regardless of which database |
| Can two connections use the same database? | Yes — many connections to one DB |
| Can one connection use two databases at once? | No — one database at a time |

---

## 🐳 10. How Docker Changes This (Important)

In traditional setup (your current machine):
- **One instance** → All databases share 100 connections

In Docker with microservices:

```yaml
services:
  auth_db:
    image: postgres:16
    environment:
      POSTGRES_DB: auth_system_db
  
  school_db:
    image: postgres:16
    environment:
      POSTGRES_DB: scolar_db
```

Now you have **two separate instances**:

| Aspect | Traditional | Docker Microservices |
|--------|-------------|----------------------|
| Number of instances | 1 | 1 per service |
| `max_connections` | 100 total | 100 per service (200 total) |
| Memory | Shared (128MB) | Separate per container |
| Port | 5432 | 5432:5432 (different host ports) |
| Isolation | Databases share | Databases are truly isolated |

**This is why Docker/microservices scale better — databases are truly independent.**

---

## 📊 11. Prerequisites

```bash
# PostgreSQL installed and running
sudo systemctl status postgresql

# Access to postgres user
sudo -u postgres psql
```

> 📌 **Note about `sudo -u postgres psql`:**  
> When you run this command, you are:
> 1. **First** connecting to the PostgreSQL instance (the running server)
> 2. **Then** being placed inside a default database (called `postgres`)
> 
> You never connect to a database without first connecting to the instance. The connection lives at the instance level; the database is just context.

---

## 🔍 12. Understanding Your Instance

### Check global connection limit
```sql
SHOW max_connections;
```
**Typical output:** `100`

👉 **This is your HARD LIMIT** across ALL databases on this instance.

> 📌 **Important:** This limit applies to the ENTIRE instance, not per database. If you have 4 databases, they all share these 100 connection slots.

### List all databases
```sql
\l
```

**Example output:**
```
      Name      |  Owner   
----------------+----------
 auth_system_db | emma     
 postgres       | postgres
 scolar_db      | emma     
 tracktus_db    | emma     
```

---

## 🔄 13. Switching Databases on the Same Connection

One of the most misunderstood concepts: **You can change databases without creating a new connection**.

### Check your current database
```sql
SELECT current_database();
```

### Switch to a different database (same connection!)
```sql
\c scolar_db
```

**Output:**
```
You are now connected to database "scolar_db" as user "postgres".
```

### Verify the switch
```sql
SELECT current_database();
```

**Result:** `scolar_db`

### Prove it's the same connection
```sql
SELECT pg_backend_pid();
```

**Note the PID** — it hasn't changed!

👉 **This proves:** Connection belongs to the instance, not to a specific database

> 📌 **Why this matters in production:**  
> While you CAN switch databases on the same connection, most application frameworks (like Django) don't do this by design. Each app typically maintains its own connection pool to its specific database for security and isolation.

---

## 🔌 14. Live Connection Monitoring

### View ALL current connections
```sql
SELECT pid, usename, datname, state, backend_type
FROM pg_stat_activity;
```

**What each column means:**
| Column | Meaning |
|--------|---------|
| `pid` | Process ID (kill connections using this) |
| `usename` | Database user |
| `datname` | Which database they're connected to |
| `state` | `active` (running query) / `idle` (waiting) |
| `backend_type` | `client backend` = real app connection |

### Count total connections
```sql
SELECT count(*) FROM pg_stat_activity;
```

### Count ONLY real app connections (exclude system processes)
```sql
SELECT count(*) FROM pg_stat_activity 
WHERE backend_type = 'client backend';
```

> 📌 **Why exclude system processes?**  
> PostgreSQL runs internal background workers (autovacuum, checkpointer, walwriter, etc.). These are NOT user connections and don't count toward your `max_connections` limit. Filtering them out shows you only the connections your apps are using.

### Group connections by state (CRITICAL for debugging)
```sql
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;
```

**Interpretation:**
- `active` → currently running queries
- `idle` → open connections doing nothing (wasting slots!)
- empty → PostgreSQL internal processes (ignore)

### See which database is consuming connections
```sql
SELECT datname, count(*) as connections
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY datname
ORDER BY connections DESC;
```

---

## 🔑 15. Connection Identity Test

### Step 1: Get your current connection PID
```sql
SELECT pg_backend_pid();
```
**Example output:** `9693`

### Step 2: Switch databases
```sql
\c auth_system_db
```

### Step 3: Check PID again
```sql
SELECT pg_backend_pid();
```
**Output:** `9693` (same!)

### Step 4: Open a new terminal and connect
```bash
sudo -u postgres psql -d scolar_db
```

### Step 5: Check its PID
```sql
SELECT pg_backend_pid();
```
**Output:** `10344` (different!)

🎉 **What you proved:**
- Same terminal = same connection (even after switching DBs)
- Different terminal = different connection (even to same DB)

> 📌 **Key insight:** A connection is identified by its PID. That PID stays the same even when you switch databases, proving the connection belongs to the instance, not to a specific database.

---

## 🧪 16. The 2-Terminal Live Experiment (CRITICAL)

This is the **most important hands-on test** to prove connections are per instance, not per database.

### What You'll Prove:
- Each new terminal = new connection
- Connections are independent of which database you're using
- The global `max_connections` limit applies across ALL terminals

### Step-by-Step Instructions:

**Step 1: Open Terminal 1**
```bash
sudo -u postgres psql
```

**Step 2: Open Terminal 2**
```bash
sudo -u postgres psql
```

**Step 3: In Terminal 1, check current connections**
```sql
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';
```
**Output:** `2`

**Step 4: In Terminal 2, connect to a different database**
```sql
\c auth_system_db
SELECT current_database();
```
**Output:** `auth_system_db`

**Step 5: Back in Terminal 1, check connections again**
```sql
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';
```
**Output:** `2` (still! same connections, just one changed database)

**Step 6: Open Terminal 3**
```bash
sudo -u postgres psql -d scolar_db
```

**Step 7: Back in Terminal 1, watch the count grow**
```sql
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';
```
**Output:** `3`

### What This Proves:

| Observation | Conclusion |
|-------------|------------|
| Terminal 1 + Terminal 2 = 2 connections | Each terminal = independent connection |
| Switching DBs in Terminal 2 didn't change count | Connection belongs to instance, not database |
| Terminal 3 added 1 connection | New terminal = new connection regardless of database |
| All connections visible from any terminal | All connections share same global pool |

### The Final Proof:

```sql
-- Run this in Terminal 1 to see all connections
SELECT pid, usename, datname, state 
FROM pg_stat_activity 
WHERE backend_type = 'client backend'
ORDER BY pid;
```

**Example output:**
```
 pid  | usename  |   datname    | state  
------+----------+--------------+--------
 9693 | postgres | postgres     | idle   ← Terminal 1
 10344| postgres | auth_system_db| idle  ← Terminal 2 (different DB!)
 10345| postgres | scolar_db    | idle   ← Terminal 3 (different DB!)
```

🎉 **You just proved:** Three terminals = three connections. Each connection can be in a different database. All share the same global limit.

> 📌 **Real-world application:** Every time your Django app spawns a new thread or process that needs database access, it creates a new connection (unless you're using connection pooling). 100 gunicorn workers = up to 100 connections, regardless of which database they use!

---

## ⚠️ 17. The Dangerous States

### Check for IDLE IN TRANSACTION (⚠️ PRODUCTION KILLER)
```sql
SELECT pid, state, query, now() - query_start AS idle_duration
FROM pg_stat_activity
WHERE state = 'idle in transaction';
```

**Why this is dangerous:**
- Transaction left open
- Locks tables
- Blocks other queries
- Consumes connection slot

> 📌 **Common cause:** Forgetting to call `commit()` or `rollback()` after a transaction, or having unclosed `with transaction.atomic():` blocks in Django.

### Check for long-running active queries
```sql
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
ORDER BY duration DESC;
```

> 📌 **Example output interpretation:**  
> If you see a query running for `00:15:00` (15 minutes), that connection stays `active` for 15 minutes, blocking other queries and consuming a connection slot the entire time.

---

## ❌ 18. Common Misconceptions (Cleared Up)

| Misconception | Reality |
|---------------|---------|
| "Each database has its own connections" | **NO** — All databases share one global pool |
| "Connections are per database" | **NO** — Connections belong to the instance |
| "You can query across databases" | **NO** — `SELECT * FROM db1.table JOIN db2.table` fails |
| "Empty databases are safe" | **NO** — Another DB can block your empty DB |
| "Monitoring per DB means limits per DB" | **NO** — Monitoring ≠ enforcement |

> 📌 **Example of the empty database trap:**  
> You have `max_connections = 100`. Database A uses 95 connections. Database B has 0 connections. A new user tries to connect to Database B → gets `FATAL: too many clients already`. Database B is empty but unreachable!

---

## 🔫 19. Killing Problem Connections

### Kill a single connection
```sql
SELECT pg_terminate_backend(PID_HERE);
```

### Kill ALL idle connections to a specific database
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle' 
  AND datname = 'auth_system_db';
```

> 📌 **When to use this:** After a deployment that left many idle connections, or when you need to free up slots quickly without restarting PostgreSQL.

### Kill connections from a specific user
```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'emma';
```

---

## 🛡️ 20. Production Protection (Per-Database Limits)

### The problem: One database can starve ALL others
```sql
-- Global limit is 100
-- If auth_system_db uses 95, scolar_db can only use 5
```

### The solution: Set per-database limits
```sql
-- Protect each database
ALTER DATABASE auth_system_db CONNECTION LIMIT 40;
ALTER DATABASE scolar_db CONNECTION LIMIT 30;
ALTER DATABASE tracktus_db CONNECTION LIMIT 20;

-- Verify limits
SELECT datname, datconnlimit 
FROM pg_database 
WHERE datname IN ('auth_system_db', 'scolar_db', 'tracktus_db');
```

**Result:**
```
    datname     | datconnlimit 
----------------+--------------
 auth_system_db |           40
 scolar_db      |           30
 tracktus_db    |           20
```

### Limit per user
```sql
ALTER ROLE emma CONNECTION LIMIT 50;
```

### Remove a limit
```sql
ALTER DATABASE auth_system_db CONNECTION LIMIT -1;  -- -1 = unlimited
```

> 📌 **Best practice:** Always set per-database limits in production, even if they're high. This prevents one noisy neighbor from taking down everything.

---

## 💥 21. Real Production Failure Example

### The Setup
```
PostgreSQL instance with max_connections = 100
├── auth_system_db (authentication service)
├── scolar_db (school management app)
└── analytics_db (reporting service)
```

### The Problem
- Authentication service has a connection leak (opens connections, never closes)
- Over 24 hours: 85 idle connections from auth_system_db
- School app tries to connect during peak hours: needs 20 connections
- Analytics tries to run daily report: needs 15 connections

### The Failure
```
Total needed: 85 + 20 + 15 = 120 connections
Max allowed: 100

Result: 
- School app users see "too many clients already"
- Analytics report fails
- Auth service still works (already connected)
```

### The Fix Applied Immediately
```sql
-- Kill idle auth connections
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle' 
  AND datname = 'auth_system_db';

-- Set per-database limits to prevent recurrence
ALTER DATABASE auth_system_db CONNECTION LIMIT 50;
ALTER DATABASE scolar_db CONNECTION LIMIT 30;
ALTER DATABASE analytics_db CONNECTION LIMIT 20;
```

> 📌 **Post-mortem:** The auth service had a bug where it wasn't closing database connections after each request. Each request left an `idle` connection. After 24 hours of traffic → 85 idle connections → outage.

---

## 🔧 22. The Solution: Connection Pooling (PgBouncer)

The production failure above happened because each request created a new connection. **Connection pooling** solves this.

### What is PgBouncer?

> **PgBouncer is a lightweight connection pooler for PostgreSQL.** It sits between your app and the database, reusing connections instead of creating new ones for every request.

### How It Works (Simple Explanation)

```
Without PgBouncer:
App → 1000 requests → 1000 connections → PostgreSQL 💥

With PgBouncer:
App → 1000 requests → PgBouncer (reuses 20 connections) → PostgreSQL ✅
```

### The Mental Model

| Without Pooling | With PgBouncer |
|----------------|----------------|
| Each request = new connection | Many requests = few connections |
| `max_connections = 100` limits you | Handle 10,000+ users with 20 connections |
| Connections open/close constantly | Connections stay open and get reused |

### Basic PgBouncer Example

```ini
# pgBouncer.ini (simplified)
[databases]
auth_system_db = host=localhost port=5432 dbname=auth_system_db
scolar_db = host=localhost port=5432 dbname=scolar_db

[pgbouncer]
listen_port = 6432
listen_addr = *
auth_type = md5
auth_file = userlist.txt

# CRITICAL: Only 20 connections to PostgreSQL!
default_pool_size = 20
max_client_conn = 1000   # Apps can connect here
```

### What Changes With PgBouncer

```sql
-- WITHOUT PgBouncer (1000 users = 1000 connections)
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';
-- Output: 1000 (💥 near limit)

-- WITH PgBouncer (1000 users = 20 connections)
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';
-- Output: 20 (✅ safe and stable)
```

### How Django/Your App Connects

```python
# Without PgBouncer
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'PORT': 5432,  # Direct to PostgreSQL
    }
}

# With PgBouncer (just change the port!)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'PORT': 6432,  # Point to PgBouncer instead
    }
}
```

### Why This Matters

| Problem | Without PgBouncer | With PgBouncer |
|---------|-------------------|----------------|
| 1000 concurrent users | 1000 connections → CRASH | 20 connections → WORKS |
| Connection leak | DB fills up slowly | PgBouncer limits damage |
| App restarts | Idle connections linger | PgBouncer cleans them up |
| Multiple apps | Each app needs its own limit | Share the pool efficiently |

> 📌 **The Golden Rule:**  
> `max_connections` in PostgreSQL should be set to the number of connections PgBouncer will use (e.g., 20-50), NOT the number of users. PgBouncer handles the user-to-connection mapping.

### When You Need PgBouncer

- ✅ Your app has more than **100 concurrent users**
- ✅ You see `too many clients already` errors
- ✅ You have multiple apps connecting to the same database
- ✅ You're deploying to production (seriously, use it)

> 📌 **Bottom line:** PgBouncer lets you handle **thousands of users** with a PostgreSQL instance limited to **100 connections**. It's the standard solution for connection exhaustion in production.

### One-Line Summary

> **"PgBouncer is a traffic cop that lets 1000 apps share 20 database connections without crashing."**

---

## 📊 23. Quick Debug Queries Reference

```sql
-- See connections grouped by database (monitoring only!)
SELECT datname, count(*)
FROM pg_stat_activity
WHERE backend_type = 'client backend'
GROUP BY datname;

-- See your current connection info
SELECT 
    pg_backend_pid() as my_pid,
    current_database() as my_db,
    current_user as my_user;

-- UNDERSTAND YOUR CONNECTION (Additional)
SELECT pg_backend_pid();           -- Your connection ID
SELECT current_database();         -- Your current database
SELECT inet_server_addr();         -- Instance IP you're connected to
SELECT inet_server_port();         -- Instance port (usually 5432)

-- PROVE CONNECTION SURVIVES DATABASE SWITCH
SELECT pg_backend_pid();           -- Note this PID
\c scolar_db
SELECT pg_backend_pid();           -- Same PID!

-- See connection age (find long-running connections)
SELECT pid, usename, datname, 
       now() - backend_start as connection_age,
       now() - query_start as query_age,
       state
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY connection_age DESC;

-- Check global limit vs current usage
SELECT 
    (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') as max_connections,
    (SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend') as current_connections,
    round(100.0 * (SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend') / 
          (SELECT setting::int FROM pg_settings WHERE name = 'max_connections'), 2) as usage_percent;
```

---

## 🏆 24. Production Best Practices

### ✅ DO THIS

```sql
-- 1. Set per-database limits immediately
ALTER DATABASE auth_system_db CONNECTION LIMIT 40;
ALTER DATABASE scolar_db CONNECTION LIMIT 30;

-- 2. Set per-user limits
ALTER ROLE app_user CONNECTION LIMIT 50;

-- 3. Monitor regularly
SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname;

-- 4. Alert at 80% usage
SELECT 
    CASE 
        WHEN count(*) > (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') * 0.8 
        THEN 'WARNING' 
        ELSE 'OK' 
    END as status
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

### ❌ DON'T DO THIS

```sql
-- 1. Don't rely only on global max_connections
-- (One database can still starve others)

-- 2. Don't ignore idle connections
-- SELECT * FROM pg_stat_activity WHERE state = 'idle';  -- CHECK THIS!

-- 3. Don't use postgres user for apps
-- (Use dedicated users with limits)

-- 4. Don't forget to close connections in app code
-- (Connection leaks are the #1 cause of outages)
```

### Architecture Best Practices

| Scenario | Recommendation |
|----------|----------------|
| Multiple apps sharing one DB | Set per-database and per-user limits |
| Microservices with separate DBs | Use separate PostgreSQL instances if possible |
| High traffic (>1000 concurrent users) | Add PgBouncer connection pooler |
| Reporting + Transactional workloads | Use separate read replicas |
| Development vs Production | Different instances, never share |

> 📌 **Golden Rule:** In production, your database should never run out of connections. If it does, that's a design problem, not a resource problem.

---

## 🧠 25. The Mental Model (Memorize This)

```
PostgreSQL Instance (max_connections = 100)
│
├── Database: auth_system_db (limit: 40)
│   └── 35 connections (active + idle) ✅
│
├── Database: scolar_db (limit: 30)  
│   └── 25 connections ✅
│
├── Database: tracktus_db (limit: 20)
│   └── 20 connections ✅
│
└── Remaining: 20 connections available
    (100 - 35 - 25 - 20 = 20)

ONE INSTANCE. ONE GLOBAL POOL. PER-DATABASE PROTECTION.
```

---

## 🧠 26. One-Line Memory Tricks

> **"One server, one connection pool, many databases sharing it"**

> **"Connections live in the instance, but sit inside a database"**

> **"Monitoring shows per DB, but limits apply to all"**

> **"Your empty database isn't safe from your noisy neighbor"**

> **"You connect to PostgreSQL first, then PostgreSQL puts you inside a database"**

> **"The instance is the hotel. The connection is the guest. The database is the room."**

> **"Your connection ID (PID) proves you belong to the instance, not to a database."**

> **"In traditional PostgreSQL: one instance, one connection pool, many databases sharing it."**

> **"In Docker: one container, one instance, one database — true isolation."**

> **"PgBouncer is a traffic cop that lets 1000 apps share 20 database connections without crashing."**

> **"Open 2 terminals, watch 2 connections. That's the proof."**

---

## 📋 27. Quick Reference Card

```sql
-- UNDERSTAND YOUR CONNECTION
SELECT pg_backend_pid();           -- Your connection ID
SELECT current_database();         -- Your current database
SELECT inet_server_addr();         -- Instance IP you're connected to
SELECT inet_server_port();         -- Instance port (usually 5432)

-- SWITCH DATABASES (SAME CONNECTION!)
\c auth_system_db                  -- Move to another database
\c postgres                        -- Move back to default

-- PROVE CONNECTION SURVIVES DATABASE SWITCH
SELECT pg_backend_pid();           -- Note this PID
\c scolar_db
SELECT pg_backend_pid();           -- Same PID!

-- 2-TERMINAL TEST (run in separate terminals)
-- Terminal 1: sudo -u postgres psql
-- Terminal 2: sudo -u postgres psql
-- In Terminal 1:
SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';
-- Output: 2 (proves each terminal = new connection)

-- SHOW LIMITS
SHOW max_connections;                           -- Global limit
SELECT datname, datconnlimit FROM pg_database;  -- Per-database limits
SELECT rolname, rolconnlimit FROM pg_roles;     -- Per-user limits

-- MONITORING
SELECT * FROM pg_stat_activity;                          -- Everything
SELECT count(*) FROM pg_stat_activity;                   -- Total
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;  -- By state
SELECT datname, count(*) FROM pg_stat_activity GROUP BY datname; -- By DB

-- KILL CONNECTIONS
SELECT pg_terminate_backend(pid) WHERE ...;  -- General pattern
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle';  -- Kill idle
SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = 'x';    -- Kill by DB

-- SET LIMITS
ALTER DATABASE name CONNECTION LIMIT 50;
ALTER ROLE name CONNECTION LIMIT 50;
```

---

## 🎓 28. What You Should Remember

| Concept | Truth |
|---------|-------|
| Where do connections live? | **PostgreSQL instance** (server level) |
| What database is a connection attached to? | **One at a time** |
| Where is the limit applied? | **Instance level** (global) |
| Can one database starve others? | **YES** — without per-database limits |
| How to prevent starvation? | `ALTER DATABASE ... CONNECTION LIMIT N` |
| How to handle thousands of users? | **PgBouncer** connection pooler |
| How to prove connections are per instance? | **Open 2 terminals** → see 2 connections |

---

## 🤔 29. FAQ

**Q: When I run `psql`, am I connecting to a database directly?**  
A: Not exactly. You first connect to the PostgreSQL instance, then the instance places you inside a default database (usually `postgres`).

**Q: What is a "PostgreSQL instance"?**  
A: The running PostgreSQL process on your machine that manages everything — memory, background workers, all databases, authentication, and connections.

**Q: Can one application crash my entire database server?**  
A: Yes — if it uses all available connections with `max_connections`, no other database (or even the same database with different users) can connect.

**Q: Why do I see connections in `pg_stat_activity` with no `datname`?**  
A: Those are PostgreSQL internal system processes (autovacuum, checkpointer, etc.) — they don't count toward `max_connections`.

**Q: Can I have multiple databases in one PostgreSQL instance?**  
A: Yes — that's the default. One instance can (and often does) manage many databases.

**Q: Should I put multiple apps in one database or multiple databases?**  
A: Multiple databases (one per app) is better for isolation, security, and per-database limits.

**Q: What's better for scaling: one instance with many DBs or many instances?**  
A: Many instances (e.g., Docker containers) give you true isolation — each has its own `max_connections`, memory, and CPU.

**Q: What is PgBouncer and when do I need it?**  
A: PgBouncer is a connection pooler that lets thousands of users share a few database connections. You need it when you have more than ~100 concurrent users.

**Q: How do I prove connections are per instance, not per database?**  
A: Open 2 terminals, run `psql` in both, then run `SELECT count(*) FROM pg_stat_activity WHERE backend_type = 'client backend';` — you'll see 2 connections, proving each terminal = new connection regardless of database.

---

## 🚀 30. Next Steps

After mastering connection management, you're ready for:

1. **Connection Pooling** (PgBouncer) — Handle thousands of connections (see Section 22 above)
2. **Read Replicas** — Distribute query load
3. **Query Optimization** — Reduce active connection time
