# Logical Replication: Troubleshooting and Validation

Common issues, validation queries and cleanup for the PostgreSQL 16 logical replication test.

> The issues in Part 1 are the ones most commonly hit with this setup. After running your own test, add anything you actually faced that is not listed.

Run SQL with `sudo -u postgres psql` on the publisher (port 5432) and `sudo -u postgres psql -p 5433` on the subscriber.

## Part 1: Common issues and fixes

### 1. `logical decoding requires wal_level >= logical`
**Cause:** `wal_level` is still `replica` on the publisher, or the restart was skipped.
**Fix:** set `wal_level = logical` in `/etc/postgresql/16/main/postgresql.conf` and restart (a reload is not enough):
```bash
sudo systemctl restart postgresql@16-main
sudo -u postgres psql -c "SHOW wal_level;"
```

### 2. `no pg_hba.conf entry for host ..., user "logrepl", database "lr_demo"`
**Cause:** the `pg_hba.conf` entry is missing, or it uses the `replication` keyword. That keyword only matches physical replication connections. Logical replication connects to a named database.
**Fix:** add the entry with the real database name and reload:
```
host  lr_demo  logrepl  <subscriber_private_ip>/32  scram-sha-256
```
```bash
sudo systemctl reload postgresql@16-main
```

### 3. `permission denied to start WAL sender` / `must be superuser or replication role`
**Cause:** the subscription's role lacks the `REPLICATION` attribute.
**Fix:**
```sql
ALTER ROLE logrepl WITH REPLICATION;
```

### 4. `permission denied for table orders` during the initial copy
**Cause:** the role has no `SELECT` on the published table.
**Fix (on the publisher, in `lr_demo`):**
```sql
GRANT SELECT ON orders TO logrepl;
```

### 5. `logical replication target relation "public.orders" does not exist` (subscriber log)
**Cause:** the table was not created on the subscriber. DDL is not replicated.
**Fix:** create the table on the subscriber with the same columns, then refresh:
```sql
ALTER SUBSCRIPTION lr_sub REFRESH PUBLICATION;
```

### 6. `cannot update table "orders" because it does not have a replica identity and publishes updates`
**Cause:** the published table has no primary key, so `UPDATE`/`DELETE` cannot be identified on the subscriber.
**Fix:** add a primary key, or as a fallback:
```sql
ALTER TABLE orders REPLICA IDENTITY FULL;
```

### 7. `CREATE SUBSCRIPTION` hangs
**Cause:** creating the logical slot waits for open transactions on the publisher to finish, or the connection details point at the wrong host/port.
**Fix:** check for long-running transactions on the publisher and wait or end them:
```sql
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```
Also re-run the connectivity test from the README (step 6).

### 8. Subscription stuck, `duplicate key value violates unique constraint` (subscriber log)
**Cause:** a row already exists on the subscriber and conflicts with an incoming change. The apply worker retries in a loop.
**Fix (option A):** delete or correct the conflicting row on the subscriber.
**Fix (option B):** skip the failing transaction using the finish LSN shown in the subscriber log:
```sql
ALTER SUBSCRIPTION lr_sub SKIP (lsn = '<lsn_from_log>');
```
Check the log: `sudo tail -50 /var/log/postgresql/postgresql-16-sub.log`

### 9. Second cluster does not start / port already in use
**Cause:** another cluster is using port 5433, or the cluster was created without `--start`.
**Fix:**
```bash
pg_lsclusters
sudo pg_ctlcluster 16 sub start
sudo tail -50 /var/log/postgresql/postgresql-16-sub.log
```
Create it on a free port with `sudo pg_createcluster 16 sub --port=<free_port> --start`.

### 10. Column added on the publisher, replication stops
**Cause:** DDL is not replicated, so incoming rows no longer match the subscriber's table.
**Fix:** add the same column on the subscriber. Replication resumes automatically. To avoid it, apply DDL on the subscriber first, then the publisher.

### 11. `DROP SUBSCRIPTION` fails (`could not drop replication slot ... on publisher`)
**Cause:** the publisher is unreachable or the slot is in use.
**Fix (on the subscriber):**
```sql
ALTER SUBSCRIPTION lr_sub DISABLE;
ALTER SUBSCRIPTION lr_sub SET (slot_name = NONE);
DROP SUBSCRIPTION lr_sub;
```
Then drop the slot manually on the publisher:
```sql
SELECT pg_drop_replication_slot('lr_sub');
```

### 12. Publisher disk filling with WAL
**Cause:** an inactive logical slot (subscriber down or subscription dropped without removing the slot) makes the publisher retain WAL.
**Fix:** monitor with the retained-WAL query in Part 2, bring the subscriber back, or drop the unused slot.

## Part 2: Validation queries

### On the PUBLISHER (port 5432)

WAL level (expect `logical`):
```sql
SHOW wal_level;
```

Publications and published tables:
```sql
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;
```

Logical slot status (expect `lr_sub`, `pgoutput`, `active = t`):
```sql
SELECT slot_name, slot_type, plugin, database, active,
       restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

Walsender connections (the physical standby and the logical subscriber `lr_sub`, both `streaming`):
```sql
SELECT application_name, client_addr, state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```

Logical lag in bytes:
```sql
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)) AS lag
FROM pg_replication_slots
WHERE slot_type = 'logical';
```

WAL retained by each slot (watch for inactive slots with large values):
```sql
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_wal
FROM pg_replication_slots;
```

### On the SUBSCRIBER (port 5433)

Subscription definition:
```sql
SELECT subname, subenabled, subslotname, subpublications
FROM pg_subscription;
```

Apply worker status (a row with a non-null `pid` means it is running):
```sql
SELECT subname, pid, received_lsn,
       last_msg_send_time, last_msg_receipt_time,
       latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```

Per-table sync state (expect `r` = ready; `i` init, `d` copying, `f` copy finished, `s` synchronized):
```sql
SELECT srrelid::regclass AS table_name, srsubstate
FROM pg_subscription_rel;
```

Error counters:
```sql
SELECT subname, apply_error_count, sync_error_count
FROM pg_stat_subscription_stats;
```

### End-to-end test

Publisher:
```sql
INSERT INTO orders VALUES (10, 'monitor', 1);
UPDATE orders SET qty = 3 WHERE id = 10;
DELETE FROM orders WHERE id = 2;
SELECT count(*) FROM orders;
```

Subscriber (the data and the count should match the publisher):
```sql
SELECT * FROM orders ORDER BY id;
SELECT count(*) FROM orders;
```

Writable subscriber check (works, unlike a physical standby, but avoid writing to replicated tables):
```sql
CREATE TABLE local_notes (id int, note text);
INSERT INTO local_notes VALUES (1, 'only on subscriber');
```

## Part 3: Cleanup

On the **subscriber** (this also drops the slot on the publisher):
```sql
DROP SUBSCRIPTION lr_sub;
```

On the **publisher**:
```sql
DROP PUBLICATION lr_pub;
```

Confirm no leftover slot remains on the publisher:
```sql
SELECT slot_name, active FROM pg_replication_slots;
```

To remove the second cluster entirely (on the standby node):
```bash
sudo pg_dropcluster 16 sub --stop
```