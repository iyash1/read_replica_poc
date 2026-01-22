# PostgreSQL Read Replica (Primary → Replica) — Local Docker Setup

> A clean, reproducible project to **build, observe, and verify PostgreSQL streaming replication** locally using Docker.

This repo demonstrates how a **primary (read/write)** PostgreSQL database continuously streams changes to a **replica (read-only)** database, exactly like production systems (e.g., AWS RDS Read Replicas).

---

## Architecture (at a glance)

**Key ideas**

* The **primary** accepts writes.
* Every change is written to a **WAL (Write-Ahead Log)**.
* The **replica** replays WAL to stay in sync.
* A **replication slot** ensures the primary doesn’t delete WAL before the replica reads it.

---

## What you’ll get from this project

* ✅ Primary + replica running locally
* ✅ Near-real-time replication
* ✅ Read-only enforcement on the replica
* ✅ Deterministic verification steps
* ✅ A setup that mirrors production behavior

---

## Prerequisites

* Docker
* Docker Compose (v2+)
* `psql` (recommended for verification)

Check:

```bash
docker --version
docker compose version
psql --version
```

---

## Repository structure

```
.
├── docker-compose.yml
├── pg_hba.conf
└── README.md
```

---

## Ports & roles

| Service    | Role         | Port  |
| ---------- | ------------ | ----- |
| pg-primary | Read / Write | 55432 |
| pg-replica | Read-only    | 55433 |

---

## Step-by-step: Build & observe replication

### 1) Clone the repository

```bash
git clone <your-repo-url>
cd Read_Replicas
```

---

### 2) Review access rules (`pg_hba.conf`)

This file controls **who can connect** and **who can replicate**.

```conf
# Local connections
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust

# Allow connections from Docker host (macOS)
host    all             all             192.168.65.1/32         trust

# Allow streaming replication from Docker network
host    replication     postgres        172.18.0.0/16           trust
```

> If your Docker subnet differs, adjust `172.18.0.0/16` accordingly.

---

### 3) Start **only** the primary

```bash
docker compose up -d primary
```

Verify:

```bash
docker ps | grep pg-primary
```

---

### 4) Confirm the primary role

```bash
psql -h localhost -p 55432 -U postgres -d appdb
```

```sql
SELECT pg_is_in_recovery();
```

**Expected**

```
f
```

(`f` = primary)

---

### 5) Create a replication slot (critical)

Replication slots prevent WAL loss.

```sql
SELECT pg_create_physical_replication_slot('replica_slot');
```

Verify:

```sql
SELECT slot_name, active FROM pg_replication_slots;
```

**Expected**

```
replica_slot | false
```

---

### 6) Prepare a clean replica data directory

The replica **must** start from an empty directory.

```bash
docker compose stop replica
docker volume rm read_replicas_replica_data
docker volume create read_replicas_replica_data
```

---

### 7) Take a base backup from the primary

This copies the primary’s data and configures the replica to follow it.

```bash
docker run --rm \
  --network read_replicas_default \
  -e PGPASSWORD=strong_pass \
  -v read_replicas_replica_data:/var/lib/postgresql/data \
  postgres:15 \
  pg_basebackup \
    -h pg-primary \
    -U postgres \
    -D /var/lib/postgresql/data \
    -Fp -Xs -P -R \
    --slot=replica_slot
```

**You should see**

```
waiting for checkpoint
checkpoint completed
```

---

### 8) Start the replica

```bash
docker compose up -d replica
```

Verify both containers:

```bash
docker ps
```

---

## Verify replication (authoritative checks)

### From the **primary**

```bash
psql -h localhost -p 55432 -U postgres -d appdb
```

```sql
SELECT pid, state, sync_state FROM pg_stat_replication;
```

**Expected**

```
streaming | async
```

---

### From the **replica**

```bash
psql -h localhost -p 55433 -U postgres -d appdb
```

```sql
SELECT pg_is_in_recovery();
```

**Expected**

```
t
```

Check lag:

```sql
SELECT now() - pg_last_xact_replay_timestamp();
```

**Expected**

* Milliseconds (near-real-time)

---

## Observe replication in action

### Write on the primary

```sql
CREATE TABLE test (id INT, name TEXT);
INSERT INTO test VALUES (1, 'hello from primary');
```

### Read on the replica

```sql
SELECT * FROM test;
```

You should see the row almost instantly.

---

## Confirm the replica is read-only

```sql
INSERT INTO test VALUES (2, 'should fail');
```

**Expected**

```
ERROR: cannot execute INSERT in a read-only transaction
```

---

## How replication works (conceptual)

![Image](https://miro.medium.com/v2/resize%3Afit%3A1400/1%2A_UfCOH_HlSqzTMvodsvLLA.png)

![Image](https://www.postgresql.fastware.com/hubfs/Images/Blogs/img-dgm-logical-replcation-architecture-01.svg)

1. Primary writes changes to WAL
2. WAL is streamed to the replica
3. Replica replays WAL locally
4. Replication slot prevents WAL loss

---

## Cleanup

```bash
docker compose down
```

(Optional, to rebuild from scratch)

```bash
docker volume rm read_replicas_primary_data
docker volume rm read_replicas_replica_data
```

---

## What success looks like

* Primary: `pg_is_in_recovery() = f`
* Replica: `pg_is_in_recovery() = t`
* Primary shows `streaming` in `pg_stat_replication`
* New writes appear on the replica quickly
* Writes on the replica fail

---

## Notes & best practices

* Replicas are **disposable** — rebuild if WAL is lost.
* Always trust the **primary’s view** (`pg_stat_replication`).
* Replication slots trade disk space for safety.
* This mirrors how managed services (e.g., RDS) work internally.

---