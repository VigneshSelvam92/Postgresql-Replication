# PostgreSQL 16 Streaming Replication on Ubuntu (AWS EC2)

High-level steps to install PostgreSQL 16 and set up physical streaming replication between a primary and a standby.

**Assumption:** two Ubuntu 22.04/24.04 EC2 instances in the same VPC, communicating over private IPs.

## Phase 1: AWS prerequisites
1. Launch 2 Ubuntu EC2 instances in the same VPC/subnet.
2. Security group: allow TCP 5432 between the two instances, plus SSH from your IP.
3. Note both private IPs.

## Phase 2: Install PostgreSQL 16 (on BOTH nodes)
4. `sudo apt update`
5. `sudo apt install -y postgresql-common`
6. `sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh`
7. `sudo apt install -y postgresql-16`
8. Verify: `sudo systemctl status postgresql@16-main` and `psql --version`

## Phase 3: Configure the PRIMARY
9. Create the replication role (as `postgres`):
   `CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '<strong_pwd>';`
10. Edit `/etc/postgresql/16/main/postgresql.conf`:
    - `listen_addresses = '*'` (or the primary's private IP)
    - Confirm `wal_level = replica`, `max_wal_senders >= 3`, `max_replication_slots >= 1`, `hot_standby = on` (PG16 defaults)
11. Edit `/etc/postgresql/16/main/pg_hba.conf` and add:
    `host  replication  replicator  <standby_private_ip>/32  scram-sha-256`
12. Restart: `sudo systemctl restart postgresql@16-main`
13. Optional: `SELECT pg_create_physical_replication_slot('standby1_slot');`

## Phase 4: Build the STANDBY
14. `sudo systemctl stop postgresql@16-main`
15. `sudo -u postgres rm -rf /var/lib/postgresql/16/main/*`
16. As `postgres`:
    `pg_basebackup -h <primary_private_ip> -U replicator -D /var/lib/postgresql/16/main -X stream -R -P -S standby1_slot`
    (If you skipped step 13, add `-C` to create the slot here.)
17. `sudo systemctl start postgresql@16-main`

## Phase 5: Verify and test
18. Primary: `SELECT client_addr, state, sync_state FROM pg_stat_replication;` (expect `streaming`)
19. Standby: `SELECT pg_is_in_recovery();` (expect `t`)
20. Create a table and insert rows on the primary, then confirm they appear on the standby.
21. Confirm the standby is read-only: writes should fail.
22. Check lag on the standby: `SELECT now() - pg_last_xact_replay_timestamp();`

## Common gotchas
- Config files live in `/etc/postgresql/16/main/`, not in the data directory.
- Run `pg_basebackup` as `postgres`, with an empty data directory.
- Use private IPs in `pg_hba.conf` and `primary_conninfo`.
- Auth failures usually mean a `pg_hba.conf` mismatch or a missing reload/restart.
