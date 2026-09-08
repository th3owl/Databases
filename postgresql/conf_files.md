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
## Modifying postgres.conf file location
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
