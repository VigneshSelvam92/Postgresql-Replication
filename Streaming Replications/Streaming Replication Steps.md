# PostgreSQL 16 Streaming Replication on Ubuntu (AWS EC2)

Tested, working steps to install PostgreSQL 16 and configure physical streaming replication between a primary and a standby.

See [TROUBLESHOOTING_AND_VALIDATION.md](TROUBLESHOOTING_AND_VALIDATION.md) for the issues hit during the setup and the validation queries.

## Setup

| Role    | Example private IP |
|---------|--------------------|
| Primary | `<primary_private_ip>` |
| Standby | `<standby_private_ip>` |

- Two Ubuntu 22.04/24.04 EC2 instances in the same VPC, communicating over private IPs.
- PostgreSQL 16 from the official PGDG repo.
- Replication slot: `standby1_slot`, replication role: `replicator`.

## Phase 1: AWS prerequisites
1. Launch 2 Ubuntu EC2 instances in the same VPC/subnet.
2. Security group: allow TCP **5432** between the two instances (source = the SG itself or the peer's private IP), plus SSH from your IP.
3. Note both private IPs.

## Phase 2: Install PostgreSQL 16 (on BOTH nodes)
4. `sudo apt update`
5. `sudo apt install -y postgresql-common`
6. `sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh`
7. `sudo apt install -y postgresql-16`
8. Verify: `sudo systemctl status postgresql@16-main` and `psql --version`

> Run SQL as the `postgres` OS user: `sudo -u postgres psql`. The default `ubuntu` user has no matching database role (peer authentication).

## Phase 3: Configure the PRIMARY
9. Create the replication role:
   ```bash
   sudo -u postgres psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD '<strong_pwd>';"
   ```
10. Edit `/etc/postgresql/16/main/postgresql.conf`:
    - `listen_addresses = '*'` (or the primary's private IP). **Required**, the default is localhost only.
    - Confirm `wal_level = replica`, `max_wal_senders >= 3`, `max_replication_slots >= 1`, `hot_standby = on` (all PG16 defaults).
11. Edit `/etc/postgresql/16/main/pg_hba.conf` and add:
    ```
    host  replication  replicator  <standby_private_ip>/32  scram-sha-256
    ```
12. Restart (a restart is required for `listen_addresses`):
    ```bash
    sudo systemctl restart postgresql@16-main
    sudo ss -tlnp | grep 5432      # should show 0.0.0.0:5432
    ```
13. Optional: create the slot on the primary. If you skip this, use `-C` in step 16 instead.
    ```bash
    sudo -u postgres psql -c "SELECT pg_create_physical_replication_slot('standby1_slot');"
    ```

## Phase 4: Build the STANDBY
14. Stop the auto-created cluster and empty the data directory:
    ```bash
    sudo systemctl stop postgresql@16-main
    sudo -u postgres rm -rf /var/lib/postgresql/16/main/*
    ```
15. Check connectivity to the primary:
    ```bash
    pg_isready -h <primary_private_ip> -p 5432
    ```
16. Take the base backup as `postgres`:
    ```bash
    sudo -u postgres pg_basebackup -h <primary_private_ip> -U replicator \
      -D /var/lib/postgresql/16/main -X stream -R -P -S standby1_slot
    ```
    - If the slot was NOT created in step 13, add `-C` at the end.
    - If the slot WAS created in step 13, do NOT use `-C` (it fails with "slot already exists").
    - `-R` creates `standby.signal` and writes `primary_conninfo` into `postgresql.auto.conf`.
17. Start the standby (as `ubuntu` with sudo, the `postgres` user has no sudo rights):
    ```bash
    sudo systemctl start postgresql@16-main
    sudo systemctl status postgresql@16-main
    ```

## Phase 5: Verify
18. Primary:
    ```sql
    SELECT client_addr, state, sync_state FROM pg_stat_replication;
    ```
    Expect `state = streaming`.
19. Standby:
    ```sql
    SELECT pg_is_in_recovery();
    ```
    Expect `t`.
20. Create a table and insert rows on the primary, then confirm they appear on the standby.
21. Confirm the standby is read-only: writes should fail.

More validation queries are in [TROUBLESHOOTING_AND_VALIDATION.md](TROUBLESHOOTING_AND_VALIDATION.md).

## Notes
- Config files live in `/etc/postgresql/16/main/`, not in the data directory, and `pg_basebackup` does not copy them.
- Use private IPs in `pg_hba.conf` and `primary_conninfo`.
- Never commit real passwords or IPs. Keep the placeholders.
