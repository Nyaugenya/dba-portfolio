# PostgreSQL Backup & Restore Drill

## What I did
As part of preparing for a junior DBA role, I set up a local PostgreSQL
database (`bankdemo`) with a simple `accounts` table simulating a bank
ledger. I took a full backup using pgAdmin's backup tool (custom format),
then simulated a data-loss incident by deleting a row, and restored from
the backup to recover it.

## The error
My first restore attempt failed because I was restoring into a database
that already had the `accounts` table and data in it. pg_restore tried to
recreate objects that already existed:

​
pg_restore: error: could not execute query: ERROR: relation "accounts" already exists
pg_restore: error: COPY failed for table "accounts": ERROR: duplicate key value violates unique constraint "accounts_pkey"
DETAIL: Key (id)=(2) already exists.
pg_restore: error: could not execute query: ERROR: multiple primary keys for table "accounts" are not allowed
pg_restore: error: utility failed with exit code: 1
​

## Root cause
`pg_restore` recreates objects from scratch by default — it doesn't
overwrite or merge with existing tables. Since I only deleted one row
(not the whole table) before restoring, the table, sequence, and primary
key constraint were all still present and collided with the restore.

## The fix
I manually cleared the target before restoring:

​sql
DROP TABLE IF EXISTS accounts CASCADE;
DROP SEQUENCE IF EXISTS accounts_id_seq CASCADE;


Then re-ran the restore into the now-empty database. It completed
successfully, and `SELECT * FROM accounts;` confirmed all original rows
were recovered.

## What I'd do differently in production
- Use `pg_restore --clean --if-exists` to handle this automatically in a
  scripted/repeatable restore process, instead of manually dropping objects.
- Prefer restoring into a fresh, empty database (or a separate verification
  environment) rather than restoring on top of a live one, to avoid
  partial-state conflicts like this.
- Always verify data after restore with a row-count or spot-check query
  before considering the recovery complete.
