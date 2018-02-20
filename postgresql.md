# About

PostgreSQL notes


## See all active queries

```
SELECT datname, username, pid, client_addr, query_start, query FROM pg_stat_activity;
```


## Kill a query

```
SELECT pg_terminate_backend(query id);
```


## Multi column indexes

https://use-the-index-luke.com/sql/where-clause/the-equals-operator/concatenated-keys


## Show all values in an enum

```
SELECT ENUM_RANGE(ENUM_FIRST(null::name_of_enum_type), null::name_of_enum_type);
```


## Show data directory

```
SHOW data_directory;
```


## Log queries

https://stackoverflow.com/a/24705265

On Mac OS X:

Go to the data directory and edit the `postgresql.conf` file. Add these lines (or edit):

```
log_directory = 'pg_log'  # directory where log files are written
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'  # log file name pattern
log_statement = 'all'  # none, ddl, mod, all
logging_collector = on  # Enable capturing of stderr and csvlog
```
