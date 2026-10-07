# PostgreSQL Backup, Recovery, and Streaming Replication Lab
## Windows (PostgreSQL 18) – Complete Walkthrough

This README documents the full assignment: logical backups, WAL archiving, point‑in‑time recovery, and streaming replication, adapted to a Windows environment using PostgreSQL 18.

---

## Environment

- OS: Windows  
- PostgreSQL version: 18  
- Binaries location: `C:\Program Files\PostgreSQL\18\bin`  
- Data directory: `C:\Users\AliomaneJr 2002\postgres_data`  
- Backups directory: `C:\Users\AliomaneJr 2002\backups`  
  - Logical dumps: `C:\Users\AliomaneJr 2002\backups\bootcamp.dump`  
  - WAL archive: `C:\Users\AliomaneJr 2002\backups\wal`  
  - Base backups: `C:\Users\AliomaneJr 2002\backups\base` (tar.gz)  
  - Standby data: `C:\Users\AliomaneJr 2002\standby`

All commands below are run from:

```cmd
cd "C:\Program Files\PostgreSQL\18\bin"
```

Superuser for this setup: `AliomaneJr 2002`.

---

## Step 1 – Logical Backup and Restore

### 1.1 Create backups folder

```cmd
mkdir "C:\Users\AliomaneJr 2002\backups"
```

### 1.2 Take a logical backup

Using `mydb` as the working database (acting as "bootcamp"):

```cmd
pg_dump -Fc -f "C:\Users\AliomaneJr 2002\backups\bootcamp.dump" -U "AliomaneJr 2002" -d mydb
```

- `-Fc` → custom format (suitable for `pg_restore`).

### 1.3 Inspect the dump

```cmd
pg_restore --list "C:\Users\AliomaneJr 2002\backups\bootcamp.dump"
```

Shows TOC entries (objects) contained in the backup.

### 1.4 Verify by restoring to a new database

```cmd
createdb -U "AliomaneJr 2002" bootcamp_check
pg_restore -U "AliomaneJr 2002" -d bootcamp_check "C:\Users\AliomaneJr 2002\backups\bootcamp.dump"
```

Check contents:

```cmd
psql -U "AliomaneJr 2002" -d bootcamp_check -c "\dt"
```

Logical backup and restore are verified.

---

## Step 2 – Enable WAL Archiving and Take a Base Backup

### 2.1 Create WAL archive folder

```cmd
mkdir "C:\Users\AliomaneJr 2002\backups\wal"
mkdir "C:\Users\AliomaneJr 2002\backups\base"
```

### 2.2 Configure `postgresql.conf`

Edit:

```text
C:\Users\AliomaneJr 2002\postgres_data\postgresql.conf
```

In the "Archiving" section, set:

```conf
wal_level = replica
archive_mode = on
archive_command = 'copy "%p" "C:\Users\AliomaneJr 2002\backups\wal\%f"'
```

Save and close.

### 2.3 Restart PostgreSQL

```cmd
pg_ctl -D "C:\Users\%USERNAME%\postgres_data" restart
```

Verify settings:

```cmd
psql -U "AliomaneJr 2002" -d postgres -c "SHOW archive_mode; SHOW wal_level; SHOW archive_command;"
```

Expected:

- `archive_mode = on`
- `wal_level = replica`
- `archive_command` shows the copy command with the correct path.

### 2.4 Take a physical base backup

```cmd
pg_basebackup -D "C:\Users\AliomaneJr 2002\backups\base" -Ft -z -Xs -P -U "AliomaneJr 2002"
```

- `-Ft -z` → compressed tar format  
- `-Xs` → stream WAL during backup  
- `-P` → show progress  

Resulting files:

- `base.tar.gz`
- `pg_wal.tar.gz`
- `backup_manifest`

in `C:\Users\AliomaneJr 2002\backups\base`.

---

## Step 3 – Simulate a Disaster and Perform Point‑in‑Time Recovery

### 3.1 Create test data

Connect to `mydb`:

```cmd
psql -U "AliomaneJr 2002" -d mydb
```

```sql
CREATE TABLE students (
    id serial PRIMARY KEY,
    name text,
    enrolled_at timestamptz DEFAULT now()
);

INSERT INTO students (name)
SELECT 'Student ' || i
FROM generate_series(1, 10) AS i;

SELECT * FROM students;
-- 10 rows
```

### 3.2 Record disaster time

```sql
SELECT now();
-- e.g. 2026-10-07 12:09:07.372673+03
```

This timestamp defines "just after" the disaster.

### 3.3 Simulate disaster (delete all rows)

```sql
DELETE FROM students;
SELECT count(*) FROM students;
-- 0 rows
```

Data is now lost in the live database.

### 3.4 Stop PostgreSQL and move data directory aside

Exit `psql` (`\q`), then:

```cmd
pg_ctl -D "C:\Users\%USERNAME%\postgres_data" stop

move "C:\Users\AliomaneJr 2002\postgres_data" "C:\Users\AliomaneJr 2002\postgres_data_old"
```

### 3.5 Restore base backup

Create a fresh data directory:

```cmd
mkdir "C:\Users\AliomaneJr 2002\postgres_data"
```

Extract `base.tar.gz` into it (using any archive tool, e.g. 7‑Zip, Git Bash, WSL):

- Source: `C:\Users\AliomaneJr 2002\backups\base\base.tar.gz`  
- Destination: `C:\Users\AliomaneJr 2002\postgres_data`

After extraction, the directory contains a full PostgreSQL cluster as of the base backup time.

### 3.6 Configure point‑in‑time recovery

Edit the restored cluster's `postgresql.conf`:

```text
C:\Users\AliomaneJr 2002\postgres_data\postgresql.conf
```

Add (or modify) these settings:

```conf
restore_command = 'copy "C:\\Users\\AliomaneJr 2002\\backups\\wal\\%f" "%p"'
recovery_target_time = '2026-10-07 12:09:00+03'
recovery_target_action = 'promote'
```

- `recovery_target_time` is set just **before** the `DELETE`.

Create a recovery signal file:

```cmd
type nul > "C:\Users\AliomaneJr 2002\postgres_data\recovery.signal"
```

### 3.7 Start PostgreSQL in recovery mode

```cmd
pg_ctl -D "C:\Users\AliomaneJr 2002\postgres_data" -l "C:\Users\AliomaneJr 2002\postgres_data\server.log" start
```

PostgreSQL:

- Starts in recovery mode (because `recovery.signal` exists).
- Applies WAL from `C:\Users\AliomaneJr 2002\backups\wal` up to `recovery_target_time`.
- Promotes to normal operation.

### 3.8 Verify recovery

```cmd
psql -U "AliomaneJr 2002" -d mydb -c "SELECT count(*) FROM students;"
```

Expected result:

```text
 count
-------
   10
```

The `students` table is restored to its state just before the accidental `DELETE`.

---

## Step 4 – Set Up a Streaming Standby

### 4.1 Create a replication role

On the primary:

```cmd
psql -U "AliomaneJr 2002" -d postgres
```

```sql
CREATE ROLE replicator
WITH REPLICATION LOGIN PASSWORD 'reppass';
```

### 4.2 Allow replication connections

Edit `pg_hba.conf` in the primary data directory:

```text
C:\Users\AliomaneJr 2002\postgres_data\pg_hba.conf
```

Add:

```conf
host replication replicator 127.0.0.1/32 md5
```

Reload configuration:

```cmd
pg_ctl -D "C:\Users\AliomaneJr 2002\postgres_data" reload
```

### 4.3 Build the standby using `pg_basebackup`

Create a separate data directory for the standby:

```cmd
mkdir "C:\Users\AliomaneJr 2002\standby"

pg_basebackup -h 127.0.0.1 -U replicator -D "C:\Users\AliomaneJr 2002\standby" -R -P
```

- `-h 127.0.0.1` → connect to primary on localhost  
- `-U replicator` → use replication role  
- `-R` → write `primary_conninfo` and create standby signal  
- `-P` → show progress  

This creates a physical copy of the primary ready to run as a standby.

### 4.4 Configure standby port

To avoid port conflict, run the standby on a different port (e.g. 5433).

Edit:

```text
C:\Users\AliomaneJr 2002\standby\postgresql.conf
```

Set:

```conf
port = 5433
```

(Optionally adjust `log_directory` or `data_directory` references if needed.)

### 4.5 Start the standby server

```cmd
pg_ctl -D "C:\Users\AliomaneJr 2002\standby" -l "C:\Users\AliomaneJr 2002\standby\server.log" start
```

The standby connects to the primary and begins streaming WAL.

---

## Step 5 – Monitor Replication Health

On the **primary** server:

```cmd
psql -U "AliomaneJr 2002" -d postgres
```

Run:

```sql
SELECT
    application_name,
    state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes
FROM pg_stat_replication;
```

Typical output:

| application_name | state     | lag_bytes |
|------------------|-----------|-----------|
| walreceiver      | streaming | 0         |

Interpretation:

- `state = streaming` → standby is actively receiving WAL.
- `lag_bytes` near 0 → minimal replication lag; standby is nearly in sync with primary.

This confirms a healthy streaming replication setup.

---

## Wrap‑Up

Through this lab you:

1. **Logical backup & restore**  
   - Used `pg_dump` and `pg_restore` to back up and verify a database.

2. **WAL archiving & base backup**  
   - Configured `wal_level`, `archive_mode`, and `archive_command`.  
   - Took a compressed tar base backup with `pg_basebackup`.

3. **Point‑in‑time recovery**  
   - Simulated data loss by deleting all rows from a table.  
   - Restored the cluster from the base backup and applied WAL up to a specific timestamp.  
   - Verified that the deleted data reappeared after recovery.

4. **Streaming standby**  
   - Created a replication role and updated `pg_hba.conf`.  
   - Built a standby server with `pg_basebackup -R`.  
   - Ran the standby on a separate port.

5. **Replication monitoring**  
   - Queried `pg_stat_replication` to observe replication state and lag.

This demonstrates a complete backup and high‑availability workflow for PostgreSQL on Windows: logical and physical backups, PITR, and streaming replication.
