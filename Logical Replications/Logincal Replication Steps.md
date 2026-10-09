# PostgreSQL 16 Logical Replication on Ubuntu (AWS EC2)

Steps to test logical replication (publication/subscription) using the same two EC2 instances that already run physical streaming replication.

See [TROUBLESHOOTING_AND_VALIDATION.md](TROUBLESHOOTING_AND_VALIDATION.md) for common issues, validation queries and cleanup.

## Topology

The physical standby is read-only (always in recovery), so it cannot act as a logical subscriber. Instead, the subscriber is a **second cluster on the standby node, on port 5433**. The physical replication keeps running untouched.

| Role | Node | Cluster | Port |
|------|------|---------|------|
| Publisher | primary node | `16/main` (existing) | 5432 |
| Physical standby (unchanged) | standby node | `16/main` (existing) | 5432 |
| Subscriber | standby node | `16/sub` (new) | 5433 |

Placeholders used below: `<primary_private_ip>`, `<subscriber_private_ip>` (= the standby node's private IP).

## Physical vs logical (quick reference)

| | Physical | Logical |
|---|---|---|
| Replicates | Whole cluster (WAL blocks) | Selected tables (row changes) |
| Replica | Read-only | Writable, can have its own objects |
| DDL | Replicated automatically | **Not** replicated, apply manually on both sides |
| `wal_level` | `replica` | `logical` (on the publisher) |

## Phase 1: Configure the PUBLISHER (primary node)

1. Edit `/etc/postgresql/16/main/postgresql.conf`:
   ```
   wal_level = logical
   ```
   Defaults for `max_replication_slots` (10) and `max_wal_senders` (10) leave enough room for the physical slot, the physical standby and one logical subscription.

2. Allow the subscriber in `/etc/postgresql/16/main/pg_hba.conf`. Logical replication connects to a **specific database**, so use the database name here, not the `replication` keyword:
   ```
   host  lr_demo  logrepl  <subscriber_private_ip>/32  scram-sha-256
   ```

3. Restart (a restart is required for `wal_level`). The physical standby reconnects automatically.
   ```bash
   sudo systemctl restart postgresql@16-main
   sudo -u postgres psql -c "SHOW wal_level;"      # expect: logical
   ```

4. Create the role, database, table and sample data:
   ```bash
   sudo -u postgres psql
   ```
   ```sql
   CREATE ROLE logrepl WITH LOGIN REPLICATION PASSWORD '<strong_pwd>';
   CREATE DATABASE lr_demo;
   \c lr_demo
   CREATE TABLE orders (id int PRIMARY KEY, item text, qty int);
   INSERT INTO orders VALUES (1, 'laptop', 1), (2, 'mouse', 2);
   GRANT SELECT ON orders TO logrepl;
   ```
   The role needs the `REPLICATION` attribute and `SELECT` on the published tables (for the initial copy). A primary key is needed so `UPDATE` and `DELETE` can be replicated.

5. Create the publication (still connected to `lr_demo`):
   ```sql
   CREATE PUBLICATION lr_pub FOR TABLE orders;
   SELECT * FROM pg_publication_tables;
   ```

## Phase 2: Configure the SUBSCRIBER (standby node)

6. Test connectivity and credentials to the publisher first (run as `ubuntu`):
   ```bash
   psql "host=<primary_private_ip> port=5432 dbname=lr_demo user=logrepl" -c "SELECT 1;"
   ```
   Fix any failure here before creating the subscription.

7. Create and start the second cluster:
   ```bash
   sudo pg_createcluster 16 sub --port=5433 --start
   pg_lsclusters
   ```
   Expect `16 main 5432` (online, recovery) and `16 sub 5433` (online).

8. Create the database and the **same table definition** (DDL is not replicated):
   ```bash
   sudo -u postgres psql -p 5433
   ```
   ```sql
   CREATE DATABASE lr_demo;
   \c lr_demo
   CREATE TABLE orders (id int PRIMARY KEY, item text, qty int);
   ```

9. Create the subscription (connected to `lr_demo` on port 5433, as `postgres`):
   ```sql
   CREATE SUBSCRIPTION lr_sub
     CONNECTION 'host=<primary_private_ip> port=5432 dbname=lr_demo user=logrepl password=<strong_pwd>'
     PUBLICATION lr_pub;
   ```
   This creates the logical slot `lr_sub` on the publisher and starts the initial table copy.

## Phase 3: Verify

10. Subscriber (port 5433): the two initial rows should be present.
    ```sql
    SELECT * FROM orders;
    ```
11. Publisher: make changes.
    ```sql
    INSERT INTO orders VALUES (3, 'keyboard', 1);
    UPDATE orders SET qty = 5 WHERE id = 2;
    DELETE FROM orders WHERE id = 1;
    ```
12. Subscriber: confirm the changes arrived within a second or so.
    ```sql
    SELECT * FROM orders ORDER BY id;
    ```

More validation queries are in [TROUBLESHOOTING_AND_VALIDATION.md](TROUBLESHOOTING_AND_VALIDATION.md).

## Notes
- DDL (new columns, new tables) is not replicated. Apply it on the subscriber first, then the publisher.
- The subscriber is writable. Local writes to replicated tables can cause conflicts.
- Logical slots are not copied to the physical standby, so a logical slot is lost if the standby is ever promoted (PG 16).
- An inactive slot makes the publisher retain WAL and can fill the disk. Drop the subscription when you finish testing.
- Never commit real passwords or IPs. Keep the placeholders.