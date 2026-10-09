# Troubleshooting and Validation

Issues hit while building PostgreSQL 16 streaming replication on Ubuntu EC2, plus the queries used to validate it.

## Part 1: Issues faced and fixes

### 1. `psql` fails or lacks permission as the `ubuntu` user
**Cause:** PostgreSQL uses peer authentication on the local socket, so the OS user must match a database role.
**Fix:**
```bash
sudo -i -u postgres        # full shell as postgres, then run psql
sudo -u postgres psql      # one-off
```
Type `exit` to return to `ubuntu`. The `postgres` OS user has no sudo rights, so run `systemctl` commands as `ubuntu`.

### 2. Terminal always shows `git:(main)` in the home directory
**Cause:** `git init` / the first commit was run in `~` instead of a project folder, so the home directory became a repo.
**Fix:** move the repo into the project folder.
```bash
git rev-parse --show-toplevel      # confirm the repo location
cd ~
mkdir Postgresql-Replication
mv README.md Postgresql-Replication/
mv .git Postgresql-Replication/
cd Postgresql-Replication
git status
```
Or, to discard the accidental repo: `rm -rf ~/.git` (this deletes its history permanently).

### 3. `pg_basebackup: connection refused`
```
pg_basebackup: error: connection to server at "<primary_ip>", port 5432 failed: Connection refused
```
**Cause:** the primary only listened on localhost (`listen_addresses` default). A security group problem would show as a timeout, not a refusal.
**Diagnose on the primary:**
```bash
sudo systemctl status postgresql@16-main
sudo ss -tlnp | grep 5432
sudo -u postgres psql -c "SHOW listen_addresses;"
```
**Fix:** set `listen_addresses = '*'` in `/etc/postgresql/16/main/postgresql.conf`, then restart (a reload is not enough):
```bash
sudo systemctl restart postgresql@16-main
```
**Test from the standby:** `pg_isready -h <primary_private_ip> -p 5432`

### 4. `-S -C` typo in `pg_basebackup`
**Cause:** `-S` takes the slot name as its argument, so `-S -C` used `-C` as the slot name and the slot was never named.
**Fix:**
```bash
pg_basebackup -h <primary_private_ip> -U replicator -D /var/lib/postgresql/16/main \
  -X stream -R -P -S standby1_slot -C
```
Drop `-C` if the slot already exists on the primary.

### 5. `Password:` prompt during `pg_basebackup`
**Cause:** expected behavior. The connection works and the `replicator` role needs its password.
**Fix:** type the password set in `CREATE ROLE replicator ...`. To avoid the prompt, use a `.pgpass` file for the `postgres` user on the standby:
```bash
echo "<primary_private_ip>:5432:replication:replicator:<password>" > ~/.pgpass
chmod 600 ~/.pgpass
```
If authentication fails, reset the password on the primary and check the `pg_hba.conf` entry:
```bash
sudo -u postgres psql -c "ALTER ROLE replicator WITH PASSWORD 'newpwd';"
```
```
host  replication  replicator  <standby_private_ip>/32  scram-sha-256
```

### 6. `psql: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed` on the standby
**Cause:** the standby cluster was still stopped after `pg_basebackup` (it was stopped before the backup).
**Fix:** start it as `ubuntu`, then connect.
```bash
exit                                          # leave the postgres shell
sudo systemctl start postgresql@16-main
sudo systemctl status postgresql@16-main
sudo -u postgres psql
```
If it fails to start, check `sudo tail -50 /var/log/postgresql/postgresql-16-main.log` and verify:
- Data directory owner and mode: `sudo chown -R postgres:postgres /var/lib/postgresql/16/main` and `sudo chmod 700 /var/lib/postgresql/16/main`
- `standby.signal` exists: `sudo ls /var/lib/postgresql/16/main/standby.signal`

## Part 2: Validation queries

Run each with `sudo -u postgres psql` on the relevant node.

### On the PRIMARY

Replication status (main check, expect `state = streaming`):
```sql
SELECT client_addr, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```

Replication lag in bytes:
```sql
SELECT client_addr,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

Replication slot status (expect `standby1_slot`, `active = t`):
```sql
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;
```

Confirm it is the primary (expect `f`):
```sql
SELECT pg_is_in_recovery();
```

### On the STANDBY

Confirm standby mode (expect `t`):
```sql
SELECT pg_is_in_recovery();
```

WAL receiver status (expect `status = streaming`, `sender_host = <primary_private_ip>`):
```sql
SELECT status, sender_host, sender_port, slot_name,
       received_lsn, latest_end_lsn
FROM pg_stat_wal_receiver;
```

Replay lag in time (on an idle primary this value keeps growing, which is not a problem):
```sql
SELECT now() - pg_last_xact_replay_timestamp() AS replay_lag;
```

Received vs. replayed WAL (should match or be very close):
```sql
SELECT pg_last_wal_receive_lsn() AS received,
       pg_last_wal_replay_lsn()  AS replayed;
```

### End-to-end test

Primary:
```sql
CREATE TABLE repl_test (id int, note text);
INSERT INTO repl_test VALUES (1, 'hello from primary');
```

Standby (the row should appear within about a second):
```sql
SELECT * FROM repl_test;
```

Read-only check on the standby (should fail with "cannot execute ... in a read-only transaction"):
```sql
INSERT INTO repl_test VALUES (2, 'should fail');
```
