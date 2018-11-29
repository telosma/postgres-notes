
## PSQL

Magic words:
```bash
psql -U postgres
```
Some interesting flags (to see all, use `-h`):
- `-E`: will describe the underlaying queries of the `\` commands (cool for learning!)
- `-l`: psql will list all databases and then exit (useful if the user you connect with doesn't has a default database, like at AWS RDS)

Most `\d` commands support additional param of `__schema__.name__` and accept wildcards like `*.*`

- `\q`: Quit/Exit
- `\c __database__`: Connect to a database
- `\d __table__`: Show table definition including triggers
- `\l`: List databases
- `\dy`: List events
- `\df`: List functions
- `\di`: List indexes
- `\dn`: List schemas
- `\dt *.*`: List tables from all schemas (if `*.*` is omitted will only show SEARCH_PATH ones)
- `\dv`: List views
- `\df+ __function__` : Show function SQL code.
- `\x`: Pretty-format query results instead of the not-so-useful ASCII tables
- `\copy (SELECT * FROM __table_name__) TO 'file_path_and_name.csv' WITH CSV`: Export a table as CSV

User Related:
- `\du`: List users
- `\du __username__`: List a username if present.
- `create role __test1__`: Create a role with an existing username.
- `create role __test2__ noinherit login password __passsword__;`: Create a role with username and password.
- `set role __test__;`: Change role for current session to `__test__`.
- `grant __test2__ to __test1__;`: Allow `__test1__` to set its role as `__test2__`.

## Configuration

- Service management commands:
```
sudo service postgresql stop
sudo service postgresql start
sudo service postgresql restart
```

- Changing verbosity & querying Postgres log:
  <br/>

  - **Ubuntu**: /etc/postgresql/9.3/main/postgresql.conf
  - **Centos**: /var/lib/pgsql/9.3/data/postgresql.conf
  - **Manual check**: connect to psql and run `SHOW config_file`
  ```
    postgres=# SHOW config_file;
  ```

1) First edit the config file, set a decent verbosity, save and restart postgres:
```
sudo vim /etc/postgresql/9.3/main/postgresql.conf

# Uncomment/Change inside:
log_min_messages = debug5
log_min_error_statement = debug5
log_min_duration_statement = -1

sudo service postgresql restart
```
  2) Now you will get tons of details of every statement, error, and even background tasks like VACUUMs
```
tail -f /var/log/postgresql/postgresql-9.3-main.log
```
  3) How to add user who executed a PG statement to log (editing `postgresql.conf`):
```
log_line_prefix = '%t %u %d %a '
```

## Create command

There are many `CREATE` choices, like `CREATE DATABASE __database_name__`, `CREATE TABLE __table_name__` ... Parameters differ but can be checked [at the official documentation](https://www.postgresql.org/search/?u=%2Fdocs%2F9.1%2F&q=CREATE).


## Handy queries
- `SELECT * FROM pg_proc WHERE proname='__procedurename__'`: List procedure/function
- `SELECT * FROM pg_views WHERE viewname='__viewname__';`: List view (including the definition)
- `SELECT pg_size_pretty(pg_total_relation_size('__table_name__'));`: Show DB table space in use
- `SELECT pg_size_pretty(pg_database_size('__database_name__'));`: Show DB space in use
- `show statement_timeout;`: Show current user's statement timeout
- `SELECT * FROM pg_indexes WHERE tablename='__table_name__' AND schemaname='__schema_name__';`: Show table indexes
- Get all indexes from all tables of a schema:
```sql
SELECT
   t.relname AS table_name,
   i.relname AS index_name,
   a.attname AS column_name
FROM
   pg_class t,
   pg_class i,
   pg_index ix,
   pg_attribute a,
    pg_namespace n
WHERE
   t.oid = ix.indrelid
   AND i.oid = ix.indexrelid
   AND a.attrelid = t.oid
   AND a.attnum = ANY(ix.indkey)
   AND t.relnamespace = n.oid
    AND n.nspname = 'kartones'
ORDER BY
   t.relname,
   i.relname
```
- Execution data:
  - Queries being executed at a certain DB:
```sql
SELECT datname, application_name, pid, backend_start, query_start, state_change, state, query
  FROM pg_stat_activity
  WHERE datname='__database_name__';
```
  - Get all queries from all dbs waiting for data (might be hung):
```sql
SELECT * FROM pg_stat_activity WHERE waiting='t'
```
  - Currently running queries with process pid:
```sql
SELECT pg_stat_get_backend_pid(s.backendid) AS procpid,
  pg_stat_get_backend_activity(s.backendid) AS current_query
FROM (SELECT pg_stat_get_backend_idset() AS backendid) AS s;
```

Casting:
- `CAST (column AS type)` or `column::type`
- `'__table_name__'::regclass::oid`: Get oid having a table name

Query analysis:
- `EXPLAIN __query__`: see the query plan for the given query
- `EXPLAIN ANALYZE __query__`: see and execute the query plan for the given query
- `ANALYZE [__table__]`: collect statistics  

## Tools
- [pg-top](http://ptop.projects.pgfoundry.org/): `top` for PG. `sudo apt-get install ptop` + `pg_top`
- [Unix-like reverse search in psql](https://dba.stackexchange.com/questions/63453/is-there-a-psql-equivalent-of-bashs-reverse-search-history):
```bash
$ echo "bind "^R" em-inc-search-prev" > $HOME/.editrc
$ source $HOME/.editrc
```
- [PostgreSQL Exercises](https://pgexercises.com/): An awesome resource to learn to learn SQL, teaching you with simple examples in a great visual way. **Highly recommended**.
- [A Performance Cheat Sheet for PostgreSQL](https://severalnines.com/blog/performance-cheat-sheet-postgresql): Great explanations of `EXPLAIN`, `EXPLAIN ANALYZE`, `VACUUM`, configuration parameters and more. Quite interesting if you need to tune-up a postgres setup.

## Tips
- Access database without type password

  Create a `.pgpass` file in the home directory of the account that psql / pg_dump will run as. Content of `.pgpass` like template below:
    ```
    hostname:port:database:username:password
    ```

  Add read/write permissions for only current user
    ```
    $ chmod 0600 ~/.pgpass
    ```

  Now you can access without typing password by
    ```
    $ psql -U username -h hostname database
    ```

## Change data directory
My postgres version is 8.4 so maybe some config paths / commands in later version are different.
By default postgres store data in `/var/lib/pgsql/data`
**Check current config**
First need to check current config parameters like where config files was stored and where is data directory.
- login to postgresql user
  ```
  $ psql -U postgres_user
  ```

- show current data directory
  ```
  $ show data_directory
  ```
  result maybe `/var/lib/pgsql/data`

- show current config_file
  ```
  $ show config_file
  ```
**stop current postgres service** to make sure data consistency
  ```
  $ /etc/inid.d/postgresql stop
  ```

- Copy current data directory to new data directory
    - We need to use `rsync` for this action
    More info about [rsync](https://www.tecmint.com/rsync-local-remote-file-synchronization-commands/)
    - Use **archive mode** with option `-a` to keep the file mod and `-v` for verbose to check all process log.
    Because all files in postgres data are owned by *postgres user* so keep ownership to help us avoid some issue relate to *permission*.

    **Note**: when use rsync your folder name have not contain *trailing slash* ("/"), because rsync with take it as sub folder.
    assume that we need to move data directory to `/data` instead of `/var/lib/`
    ```
    $ sudo rsync -av /var/lib/pgsql /data
    ```
    - When sync data is finished, we should backup old data folder
    ```
    $ sudo mv /var/lib/pgsql/data /var/lib/pgsql/data.bak
    ```
**Tell postgresql know about new data folder**
  - By default data folder was defined by `data_directory` in `/etc/postgresql/data/postgresql.conf`. But we need to follow `config_file` value which we get before.
  In this case config_file at `/var/lib/pgsql/data/postgresql.conf` but it has just moved to '/data'. So we will edit conf at `/usr/share/pgsql/postgresql.conf`, if it was not existed make a copy from example file. Change *data_directory* in config file like below
  ```
  . . .
  data_directory = '/data/pgsql/data'
  . . .

  ```

**Restart lại service postgresql**
```
$ sudo /etc/inid.d/postgresql restart
```

**Notes**
If *postgres.conf* file was located in old data folder and has been moved before so we need to update which was changed at `/etc/inid.d/postgresql`
- Backup `/etc/inid.d/postgresql`
```
$ sudo cp /etc/inid.d/postgresql /etc/inid.d/postgresql.bak
```
- Change some config in `/etc/inid.d/postgresql`
```
# Set defaults for configuration variables
#OLD
PGENGINE=/usr/bin
PGPORT=5432
PGDATA=/var/lib/pgsql/data
PGLOG=/var/lib/pgsql/pgstartup.log
```
```
# Set defaults for configuration variables
#NEW
PGENGINE=/usr/bin
PGPORT=5432
PGDATA=/data/pgsql/data
PGLOG=/data/pgsql/pgstartup.log
```
When change config done, restart postgresql service

**Check postgresql status**
Check postgresql service status and database
Check current config_file and data_directory
If everything OK we can remove backup data before.
