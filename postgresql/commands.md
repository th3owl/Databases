# PostgreSQL `psql` Commands and Shortcuts

## Connect to PostgreSQL

```bash
psql -h localhost -p 5432 -U postgres -d demo_app
```

Example output:

```text
psql (16.14)
Type "help" for help.
```

## PostgreSQL Environment Variables

```bash
echo "$PGDATA"
export PGDATA=/var/lib/pgsql/16/data
echo "$PGDATA"
```

Set the variables permanently:

```bash
echo 'export PGDATA=/var/lib/pgsql/16/data' >> ~/.bash_profile
echo 'export PATH=/usr/pgsql-16/bin:$PATH' >> ~/.bash_profile
source ~/.bash_profile
```

Verify:

```bash
echo "$PATH"
which pg_ctl
pg_ctl --version
```

Check PostgreSQL status:

```bash
pg_ctl -D "$PGDATA" status
```

Example output:

```text
pg_ctl: server is running (PID: 851)
/usr/pgsql-16/bin/postgres "-D" "/var/lib/pgsql/16/data/"
```

## Connection Information

```sql
\conninfo
SELECT current_database();
SELECT current_user;
SELECT version();
```

Example output:

```text
 current_database
------------------
 demo_app
(1 row)

 current_user
--------------
 postgres
(1 row)

PostgreSQL 16.14 on aarch64-unknown-linux-gnu, compiled by gcc (GCC) 11.5.0 20240719 (Red Hat 11.5.0-14), 64-bit
```

## Databases

```sql
\l
\l+
\list
```

Example database list:

```text
 demo_app  | postgres | UTF8 | libc | en_US.UTF-8 | en_US.UTF-8
 postgres  | postgres | UTF8 | libc | en_US.UTF-8 | en_US.UTF-8
 template0 | postgres | UTF8 | libc | en_US.UTF-8 | en_US.UTF-8
 template1 | postgres | UTF8 | libc | en_US.UTF-8 | en_US.UTF-8
(4 rows)
```

Show the current database:

```sql
SELECT current_database();
```

Connect to another database:

```sql
\c demo_app
```

Create a database:

```sql
CREATE DATABASE demo_app;
```

Show database sizes:

```sql
SELECT
    datname,
    pg_size_pretty(pg_database_size(datname)) AS size
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
```

## Schemas

List schemas:

```sql
\dn
\dn+
```

Show the schema search path:

```sql
SHOW search_path;
```

Set the default schema search path:

```sql
SET search_path TO sales, public;
```

List schemas with SQL:

```sql
SELECT schema_name
FROM information_schema.schemata
ORDER BY schema_name;
```

Example output:

```text
 information_schema
 pg_catalog
 pg_toast
 public
 sales
(5 rows)
```

Create a schema:

```sql
CREATE SCHEMA sales;
```

## Tables

List tables in the current search path:

```sql
\dt
```

List tables in all schemas:

```sql
\dt *.*
```

List tables in the `sales` schema:

```sql
\dt sales.*
\dt+ sales.*
```

Example output:

```text
 Schema |   Name    | Type  |  Owner
--------+-----------+-------+----------
 sales  | customers | table | postgres
 sales  | orders    | table | postgres
(2 rows)
```

List tables using SQL:

```sql
SELECT
    table_schema,
    table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
ORDER BY table_schema, table_name;
```

List only non-system tables:

```sql
SELECT
    table_schema,
    table_name
FROM information_schema.tables
WHERE table_type = 'BASE TABLE'
  AND table_schema <> 'pg_catalog'
ORDER BY table_schema, table_name;
```

## Describe Tables

Describe one table:

```sql
\d sales.customers
```

Describe a table with additional details:

```sql
\d+ sales.customers
```

Describe all relations in a schema:

```sql
\d sales.*
```

Example output:

```text
Table "sales.customers"

    Column     |           Type           | Nullable | Default
---------------+--------------------------+----------+------------------------------------------------------
 customer_id   | bigint                   | not null | nextval('sales.customers_customer_id_seq'::regclass)
 customer_name | character varying(100)   | not null |
 email         | character varying(120)   | not null |
 city          | character varying(50)    |          |
 created_at    | timestamp with time zone |          | now()

Indexes:
    "customers_pkey" PRIMARY KEY, btree (customer_id)
    "customers_email_key" UNIQUE CONSTRAINT, btree (email)
```

## Columns

```sql
SELECT
    column_name,
    data_type,
    is_nullable,
    column_default
FROM information_schema.columns
WHERE table_schema = 'sales'
  AND table_name = 'customers'
ORDER BY ordinal_position;
```

## Query Data

```sql
SELECT * FROM sales.customers;
SELECT * FROM sales.customers LIMIT 5;
SELECT * FROM sales.orders;
SELECT * FROM sales.orders ORDER BY amount DESC;
SELECT COUNT(*) FROM sales.customers;
```

Filter rows:

```sql
SELECT *
FROM sales.customers
WHERE city = 'Austin';
```

Select specific columns:

```sql
SELECT customer_name, city
FROM sales.customers;
```

## Joins

```sql
SELECT
    c.customer_name,
    c.city,
    o.order_id,
    o.status,
    o.amount
FROM sales.customers AS c
JOIN sales.orders AS o
    ON o.customer_id = c.customer_id
ORDER BY o.order_id;
```

## Aggregation

```sql
SELECT
    status,
    COUNT(*) AS order_count,
    SUM(amount) AS total_amount,
    AVG(amount) AS average_amount
FROM sales.orders
GROUP BY status
ORDER BY status;
```

## Users and Privileges

List database users and roles:

```sql
\du
\du+
```

Show table privileges:

```sql
\dp
\dp sales.*
\z sales.*
```

Show the current user:

```sql
SELECT current_user;
SELECT session_user;
```

Reset the `postgres` password:

```sql
ALTER USER postgres WITH PASSWORD 'ChooseA_StrongPassword';
```

## Indexes, Views, and Sequences

```sql
\di
\di+ sales.*
\dv
\ds
\dm
```

List indexes using SQL:

```sql
SELECT
    schemaname,
    tablename,
    indexname,
    indexdef
FROM pg_indexes
WHERE schemaname = 'sales';
```

## Transactions

Commit a transaction:

```sql
BEGIN;

UPDATE sales.orders
SET status = 'completed'
WHERE order_id = 2;

COMMIT;
```

Cancel a transaction:

```sql
BEGIN;

DELETE FROM sales.orders
WHERE order_id = 2;

ROLLBACK;
```

## Display Options

```sql
\x
\x auto
\x off
```

Control the pager:

```sql
\pset pager off
\pset pager on
```

Show query execution time:

```sql
\timing on
\timing off
```

## Help Commands

```sql
\?
```

Show SQL help:

```sql
\h
\h SELECT
\h CREATE TABLE
\h ALTER TABLE
```

## Files and Shell Commands

Execute a SQL file:

```sql
\i script.sql
```

Open the current query in an editor:

```sql
\e
```

Run operating-system commands:

```sql
\! pwd
\! ls
\! clear
```

Save query output to a file:

```sql
\o /tmp/output.txt

SELECT * FROM sales.customers;

\o
```

## CSV Import and Export

Export table data:

```sql
\copy sales.customers
TO '/tmp/customers.csv'
CSV HEADER
```

Import CSV data:

```sql
\copy sales.customers(customer_name, email, city)
FROM '/tmp/customers.csv'
CSV HEADER
```

## Monitoring Queries

Show active sessions:

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    state,
    query
FROM pg_stat_activity;
```

Show estimated table row counts:

```sql
SELECT
    schemaname,
    relname AS table_name,
    n_live_tup AS estimated_rows
FROM pg_stat_user_tables
ORDER BY relname;
```

Show table sizes:

```sql
SELECT
    schemaname,
    relname AS table_name,
    pg_size_pretty(pg_total_relation_size(relid)) AS total_size
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC;
```

Show database version:

```sql
SELECT version();
```
Show config file details:

```sql
SELECT config_file;

[postgres@postgres-server ~]$ ps -ef | grep -i /postgres
postgres     793       1  0 00:54 ?        00:00:13 /usr/local/bin/postgres_exporter
postgres     851       1  0 00:54 ?        00:00:01 /usr/pgsql-16/bin/postgres -D /var/lib/pgsql/16/data/
```
Example output:

```text
              config_file               
----------------------------------------
 /var/lib/pgsql/16/data/postgresql.conf
```

Other config files:
```
[postgres@postgres-server ~]$ grep -iE "include|include_dir" /var/lib/pgsql/16/data/postgresql.conf
					# can include strftime() escapes
# CONFIG FILE INCLUDES
#include_dir = '...'			# include files ending in '.conf' from
#include_if_exists = '...'		# include file only if it exists
#include = '...'			# include file

demo_app=# select name, setting from pg_settings where setting like '%.conf%';
    name     |                setting                 
-------------+----------------------------------------
 config_file | /var/lib/pgsql/16/data/postgresql.conf
 hba_file    | /var/lib/pgsql/16/data/pg_hba.conf
 ident_file  | /var/lib/pgsql/16/data/pg_ident.conf

```
- If a parameter has been set to different values in different .conf files, it would be assing teh value mentioned in the last read file
## Exit and Keyboard Shortcuts

Exit `psql`:

```sql
\q
```
