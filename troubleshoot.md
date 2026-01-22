# Troubleshooting Guide — PostgreSQL Read Replica (Docker)

This guide helps you diagnose **why replication is not working** and how to fix it **quickly and safely**.

> **Golden rule**
> Always debug from the **primary’s perspective first**.
> If the primary doesn’t see a replica, replication is not active.

---

## Quick mental model (keep this in mind)

![Image](https://scalegrid.io/wp-content/uploads/postgres_streaming.jpg)

![Image](https://www.enterprisedb.com/sites/default/files/DisplayImage8.png)

* Replica does **not** query the primary for data
* Replica **replays WAL (change logs)**
* If WAL is missing → replica must be rebuilt
* Replicas do **not self-heal**

---

## 60-second health check (run these first)

### On the **PRIMARY**

```sql
SELECT pid, state, sync_state FROM pg_stat_replication;
```

* ✅ 1+ rows → replica connected
* ❌ 0 rows → replication NOT active

---

### On the **REPLICA**

```sql
SELECT pg_is_in_recovery();
```

* ✅ `t` → replica mode
* ❌ `f` → not a replica

```sql
SELECT now() - pg_last_xact_replay_timestamp();
```

* ✅ small interval → healthy
* ❌ `NULL` → WAL replay never started

---

## Common problems and fixes

---

## 1️⃣ Replica is running but data never updates

### Symptoms

* Replica starts normally
* Old data is visible
* New inserts never appear

### What it means (plain language)

The replica took a **snapshot photo**, but never started watching the **live video**.

### Technical cause

* WAL streaming never started
* `pg_stat_replication` is empty on the primary

### How to fix

1. Check `pg_stat_replication` on primary
2. Fix replica connection (`primary_conninfo`)
3. If WAL is missing → rebuild replica

---

## 2️⃣ `pg_stat_replication` returns no rows

### Symptoms

```sql
SELECT * FROM pg_stat_replication;
-- (0 rows)
```

### What it means

The primary does **not see any replica connected**.

### Common causes

* Replica cannot reach primary
* `pg_hba.conf` blocks replication
* Wrong hostname in `primary_conninfo`
* Replica is asking for missing WAL

### How to fix

* Verify `pg_hba.conf` includes:

  ```conf
  host replication postgres <docker-subnet> trust
  ```
* Verify `primary_conninfo`
* If WAL error appears → rebuild replica

---

## 3️⃣ `pg_last_xact_replay_timestamp()` returns `NULL`

### Symptoms

```sql
SELECT now() - pg_last_xact_replay_timestamp();
-- NULL
```

### What it means (plain language)

The replica has **never replayed a single change**.

### Technical cause

* WAL streaming never began
* Replica is disconnected
* WAL already deleted

### How to fix

* Check replica logs for WAL errors
* If WAL is gone → rebuild replica with slot

---

## 4️⃣ WAL error: “requested WAL segment has already been removed”

### Symptoms (in replica logs)

```
requested WAL segment XXXXX has already been removed
```

### What it means (plain language)

The replica is asking for **history that no longer exists**.

This is like asking for CCTV footage that was already deleted.

### Technical cause

* No replication slot
* Primary deleted old WAL
* Replica fell behind

### Correct fix (only one)

1. Create replication slot on primary
2. **Delete replica data**
3. Take fresh base backup using slot
4. Restart replica

> ⚠️ Waiting longer will NEVER fix this.

---

## 5️⃣ Replica accepts writes (this is bad)

### Symptoms

```sql
INSERT INTO table VALUES (...);
-- succeeds on replica
```

### What it means

The replica is **not actually a replica**.

### Technical cause

* `standby.signal` missing
* Replica was initialized as a primary
* Wrong data directory reused

### How to fix

* Stop replica
* Delete replica volume
* Recreate via `pg_basebackup -R`

---

## 6️⃣ `pg_is_in_recovery()` returns `f` on replica

### Symptoms

```sql
SELECT pg_is_in_recovery();
-- f
```

### What it means

This database thinks it is a **primary**.

### How to fix

* Replica setup is invalid
* Rebuild replica from scratch

---

## 7️⃣ Authentication errors / `no pg_hba.conf entry`

### Symptoms

```
no pg_hba.conf entry for host ...
```

### What it means (plain language)

The door exists, but the **security guard says no**.

### Technical cause

* Missing rule in `pg_hba.conf`
* Client IP not allowed
* Replication traffic not allowed

### How to fix

Add rules for:

```conf
# Host machine (macOS Docker)
host all all 192.168.65.1/32 trust

# Replication from Docker network
host replication postgres 172.18.0.0/16 trust
```

Restart primary.

---

## 8️⃣ Replica volume reused accidentally

### Symptoms

* Replication behaves inconsistently
* Old data appears
* WAL errors persist after fixes

### What it means (plain language)

You reused a **dirty hard drive**.

### Technical cause

* Docker volume contains old replica data
* PostgreSQL refuses to reconcile it

### How to fix

```bash
docker compose stop replica
docker volume rm <replica_volume>
docker volume create <replica_volume>
```

Then re-run base backup.

---

## 9️⃣ Primary shows replica briefly, then disappears

### Symptoms

* Replica appears in `pg_stat_replication`
* Then vanishes

### Technical cause

* Replica connects, requests missing WAL
* Primary rejects it
* Connection drops

### Fix

* Rebuild replica
* Use replication slot

---

## Decision tree (when stuck)

![Image](https://www.postgresql.fastware.com/hubfs/Images/Blogs/img-dgm-logical-replcation-architecture-01.svg)

![Image](https://a.storyblok.com/f/187930/2048x1019/4627fcf0b5/synchronous-replication-disgram.png)

**Ask in order:**

1. Does primary show a replica?
2. Is replica in recovery?
3. Is WAL replay timestamp moving?
4. Are there WAL removal errors?
5. Is the replica volume clean?

If WAL is missing → **rebuild immediately**.

---

## Final rules to remember (print-worthy)

* Replicas are **disposable**
* WAL loss is **fatal** to a replica
* Rebuilding a replica is normal
* Primary’s view is the source of truth
* If in doubt: **delete replica data and rebuild**

---

## When NOT to troubleshoot (just rebuild)

* WAL segment missing
* Replica never replayed WAL
* Volume reused accidentally
* Replica promoted accidentally

Rebuilding is faster and safer.