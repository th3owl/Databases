# postgresql.conf
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
## Modifying postgresql.conf file location
- We can modify the location of postgres.conf from its default vendor specific location
- Note: We need to shutdown POstgreSQL Server to do this
- Sequence of steps would be:
  ```
  pg_ctl -D $PGDATA stop -mf
  sudo mkfir -p /pconfig
  sudo chown postgres:postgres /pconfig
  sudo chmod -R 600 /pconfig
  mv $PGDATA/postgresql.conf /pconfig/
  pg_ctl -D $PGDATA -o '--config-file=/pconfog/postgresql.conf' start
  psql -c "show config_file"
  ```
  ## postgresql.auto.conf
  - Location default to PGDATA
  - Location CAN'T be modified
  - Any changes made using `ALTER SYSTEM` command are captures in this file
  - This is the LAST file read by teh server. Meaning, say we have a parameter in postgresql.conf and postgresql.auto.conf, preference is given to teh latter. 
  ### Example
```text
  Before Change:

[postgres@postgres-server ~]$ grep -i work_mem /var/lib/pgsql/16/data/postgresql.conf
#work_mem = 4MB                         # min 64kB

[postgres@postgres-server ~]$ grep -i work_mem /var/lib/pgsql/16/data/postgresql.auto.conf
grep: /var/lib/pgsql/16/data/postgresql.auto.conf: No such file or directory

demo_app=# show work_mem;
 work_mem 
----------
 4MB

After Change:

demo_app=# ALTER SYSTEM SET work_mem TO '8MB';
ALTER SYSTEM
demo_app=# show work_mem;
 work_mem 
----------
 4MB
(1 row)

[postgres@postgres-server ~]$ grep -i work_mem /var/lib/pgsql/16/data/postgresql.conf
#work_mem = 4MB                         # min 64kB

[postgres@postgres-server ~]$ grep -i work_mem /var/lib/pgsql/16/data/postgresql.auto.conf
work_mem = '8MB'
```
- In this case we need to reload teh config to update the value
```
demo_app=#  select context, name, setting, pending_restart from pg_settings where name = 'work_mem';
 context |   name   | setting | pending_restart 
---------+----------+---------+-----------------
 user    | work_mem | 8192    | f

demo_app=# select pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

demo_app=# show work_mem;
 work_mem 
----------
 8MB
(1 row)
```
- Some parameter require a restart of Postgres itself. Reloading teh config wont help in such cases
### Understanding `context` in `pg_settings`
- In pg_settings, context tells you when a parameter change takes effect.
```
Context				Meaning
internal			Fixed internally; cannot be changed by users
postmaster			Requires a complete PostgreSQL server restart
sighup				Takes effect after configuration reload
backend				Applies when a new backend/session starts
superuser-backend	New session required; usually only a superuser can change it
user				Can be changed by any user for their own session
superuser			Can be changed by a superuser for their own session
superuser-backend	Superuser setting that applies to newly created sessions
```
#### Example
```
demo_app=# SELECT
    name,
    setting,
    context,
    pending_restart
FROM pg_settings
WHERE name IN (
    'archive_mode',
    'archive_command',
    'work_mem',
    'shared_buffers',
    'max_connections',
    'log_min_duration_statement'
)
ORDER BY name;
            name            |  setting   |  context   | pending_restart 
----------------------------+------------+------------+-----------------
 archive_command            | (disabled) | sighup     | f
 archive_mode               | off        | postmaster | f
 log_min_duration_statement | -1         | superuser  | f
 max_connections            | 100        | postmaster | f
 shared_buffers             | 16384      | postmaster | f
 work_mem                   | 8192       | user       | f
(6 rows)
```

