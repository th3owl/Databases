# Create a dedicated owner role
```
postgres=# CREATE ROLE demo_owner LOGIN PASSWORD 'postgres';
CREATE ROLE
```
- We can either use CREATE USER or CREATE ROLE
- PostgreSQL user is a role with CONNECT privilege
- With CREATE ROLE, LOGIN role is not assigned automatically to the user
```
demo_reporting=# SELECT
    rolname,
    rolsuper,
    rolinherit,
    rolcreaterole,
    rolcreatedb,
    rolcanlogin,
    rolreplication,
    rolbypassrls,
    rolconnlimit,
    rolvaliduntil
FROM pg_roles
WHERE rolname = 'demo_owner';
  rolname   | rolsuper | rolinherit | rolcreaterole | rolcreatedb | rolcanlogin | rolreplication | rolbypassrls | rolconnlimit | rolvaliduntil 
------------+----------+------------+---------------+-------------+-------------+----------------+--------------+--------------+---------------
 demo_owner | f        | t          | f             | f           | t           | f              | f            |           -1 | 
```
```
postgres=# \du+ demo_owner
             List of roles
 Role name  | Attributes | Description 
------------+------------+-------------
 demo_owner |            | 
```
# Create a database with an owner and connection limit
- CREATEDB and DROPDB are the OS level commands used to create and drop a database
- CREATE DATABASE and DROP DATABASE are DB level commands used to create and drop a database
```
postgres=# CREATE DATABASE demo_reporting
    WITH
    OWNER = demo_owner
    TEMPLATE = template0
    ENCODING = 'UTF8'
    CONNECTION LIMIT = 5
    ALLOW_CONNECTIONS = true;
CREATE DATABASE

postgres=# \l+ demo_reporting
                                                                           List of databases
      Name      |   Owner    | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules | Access privileges |  Size   | Tablespace | Description 
----------------+------------+----------+-----------------+-------------+-------------+------------+-----------+-------------------+---------+------------+-------------
 demo_reporting | demo_owner | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |                   | 7345 kB | pg_default | 
(1 row)
```
# Altering database settings
```
postgres=# ALTER DATABASE demo_reporting
    SET timezone TO 'UTC';

ALTER DATABASE demo_reporting
    SET statement_timeout TO '30s';

ALTER DATABASE demo_reporting
    SET idle_in_transaction_session_timeout TO '60s';
ALTER DATABASE

postgres=# SELECT  datname,datallowconn,datconnlimit
FROM pg_database
WHERE datname = 'demo_reporting';
    datname     | datallowconn | datconnlimit 
----------------+--------------+--------------
 demo_reporting | t            |            5

demo_reporting=# SELECT
    datname,
    pg_get_userbyid(datdba) AS owner,
    datallowconn,
    datconnlimit
FROM pg_database
WHERE datname = 'demo_reporting';
    datname     |   owner    | datallowconn | datconnlimit 
----------------+------------+--------------+--------------
 demo_reporting | demo_owner | t            |            5
```
# Connect to the new database
```
postgres=# \c demo_reporting
You are now connected to database "demo_reporting" as user "postgres".
demo_reporting=# 
demo_reporting=# SELECT
    current_database(),
    current_user,
    current_setting('timezone'),
    current_setting('statement_timeout');
 current_database | current_user | current_setting | current_setting 
------------------+--------------+-----------------+-----------------
 demo_reporting   | postgres     | UTC             | 30s
(1 row)
```
# Create a schema owned by the demo_owner
- Logical entity to group related objects
- Not mandatory to create one
- Default is `public` schema
- When a schema is created, unlike Oracle, it wont create a user with teh same name. User creation is to be done separately
- `AUTHORIZATION` is used to assign the ownership of a schema
```
demo_reporting=# CREATE SCHEMA reporting AUTHORIZATION demo_owner;
CREATE SCHEMA

demo_reporting=# show search_path;
   search_path   
-----------------
 "$user", public
(1 row)
```
## Set the default schema for the database:
```
demo_reporting=# ALTER DATABASE demo_reporting SET search_path TO reporting, public;
ALTER DATABASE

demo_reporting=# show search_path;
   search_path   
-----------------
 "$user", public

Reconnect

demo_reporting=# \c demo_reporting
You are now connected to database "demo_reporting" as user "postgres".
demo_reporting=# show search_path;
    search_path    
-------------------
 reporting, public

demo_reporting=# select n.nspname AS "Name", pg_catalog.pg_get_userbyid(n.nspowner) AS "Owner" FROM pg_catalog.pg_namespace n WHERE n.nspname = 'reporting';
   Name    |   Owner    
-----------+------------
 reporting | demo_owner
(1 row)

demo_reporting=# \conninfo
You are connected to database "demo_reporting" as user "postgres" via socket in "/run/postgresql" at port "5432".
demo_reporting=# 

```
# Sample Data Creation
## Create demo tables
```
CREATE TABLE reporting.customers (
    customer_id BIGSERIAL PRIMARY KEY,
    customer_name VARCHAR(100) NOT NULL,
    email VARCHAR(150) UNIQUE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'active'
        CHECK (status IN ('active', 'inactive')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE reporting.orders (
    order_id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL
        REFERENCES reporting.customers(customer_id),
    order_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    amount NUMERIC(12,2) NOT NULL
        CHECK (amount >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN ('pending', 'completed', 'cancelled'))
);
```
## Insert sample data
```
INSERT INTO reporting.customers
    (customer_name, email)
VALUES
    ('Alice Johnson', 'alice@example.com'),
    ('Bob Smith', 'bob@example.com'),
    ('Carol Williams', 'carol@example.com');

INSERT INTO reporting.orders
    (customer_id, amount, status)
VALUES
    (1, 125.50, 'completed'),
    (1, 80.00, 'pending'),
    (2, 240.75, 'completed'),
    (3, 45.25, 'cancelled');
```
# Object and User Checks:
```
demo_reporting=# \conninfo
You are connected to database "demo_reporting" as user "postgres" via socket in "/run/postgresql" at port "5432".

demo_reporting=# \dn+
                                         List of schemas
   Name    |       Owner       |           Access privileges            |      Description       
-----------+-------------------+----------------------------------------+------------------------
 public    | pg_database_owner | pg_database_owner=UC/pg_database_owner+| standard public schema
           |                   | =U/pg_database_owner                   | 
 reporting | demo_owner        |                                        | 
(2 rows)

demo_reporting=# \dt+ reporting.*
                                         List of relations
  Schema   |   Name    | Type  |  Owner   | Persistence | Access method |    Size    | Description 
-----------+-----------+-------+----------+-------------+---------------+------------+-------------
 reporting | customers | table | postgres | permanent   | heap          | 8192 bytes | 
 reporting | orders    | table | postgres | permanent   | heap          | 8192 bytes | 
(2 rows)

demo_reporting=# \d+ reporting.customers
                                                                      Table "reporting.customers"
    Column     |           Type           | Collation | Nullable |                    Default                     | Storage  | Compression | Stats target | Description 
---------------+--------------------------+-----------+----------+------------------------------------------------+----------+-------------+--------------+-------------
 customer_id   | bigint                   |           | not null | nextval('customers_customer_id_seq'::regclass) | plain    |             |              | 
 customer_name | character varying(100)   |           | not null |                                                | extended |             |              | 
 email         | character varying(150)   |           | not null |                                                | extended |             |              | 
 status        | character varying(20)    |           | not null | 'active'::character varying                    | extended |             |              | 
 created_at    | timestamp with time zone |           | not null | now()                                          | plain    |             |              | 
Indexes:
    "customers_pkey" PRIMARY KEY, btree (customer_id)
    "customers_email_key" UNIQUE CONSTRAINT, btree (email)
Check constraints:
    "customers_status_check" CHECK (status::text = ANY (ARRAY['active'::character varying, 'inactive'::character varying]::text[]))
Referenced by:
    TABLE "orders" CONSTRAINT "orders_customer_id_fkey" FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
Access method: heap

demo_reporting=# \du demo_owner
      List of roles
 Role name  | Attributes 
------------+------------
 demo_owner | 
```
# Identifying the physical location of a database
```
postgres=# select oid, datname from pg_database;
  oid  |    datname     
-------+----------------
     5 | postgres
 16390 | demo_app
     1 | template1
     4 | template0
 16473 | demo_reporting
(5 rows)

postgres=# \q

[postgres@postgres-server ~]$ ls -ld $PGDATA/base/16390
drwx------. 2 postgres postgres 8192 Sep  8 04:18 /var/lib/pgsql/16/data/base/16390
```
# Identifying the physical location of an object - table
```
[demo_app=# select oid, relname from pg_class where relname = 'customers';
  oid  |  relname  
-------+-----------
 16449 | customers

 demo_app=# SELECT pg_relation_filepath('sales.customers');
 pg_relation_filepath 
----------------------
 base/16390/16449
```
# Dropping a user
- User with no associated objects cane be dropped directly using `drop user` command
- If the user owns any objects, we need to re-assign teh ownership
```
\c demo

REASSIGN OWNED BY sample TO postgres;
DROP OWNED BY sample;

\c postgres

REVOKE ALL PRIVILEGES ON DATABASE demo FROM sample;

SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE usename = 'sample';

DROP ROLE sample;

\du
```
REASSIGN OWNED: transfers objects owned by sample to postgres.
DROP OWNED: removes privileges and remaining database objects owned by sample.
DROP ROLE: removes the role.
