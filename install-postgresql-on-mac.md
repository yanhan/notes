# About

How to install PostgreSQL and create a database in it on Mac OS X


## Terminology

In PostgreSQL, a `role` is equivalent to a user in MySQL.


## Install PostgreSQL

1. Install brew. Instructions at https://brew.sh

2. Install PostgreSQL

```
brew install postgresql
```

3. Run PostgreSQL and create a role for your user:

```
postgres -D /usr/local/var/postgres
createdb `whoami`
```

This willcreate a role whose username is your UNIX username and allow you to use the `psql` command which logs you into the database as your own user. Before doing that, there are no roles in the db server.

Instructions for this step are from: http://exponential.io/blog/2015/02/21/install-postgresql-on-mac-os-x-via-brew/

4. Create `postgres` role

```
psql
> CREATE ROLE postgres SUPERUSER;
> ALTER DATABASE postgres OWNER TO postgres;
```


## Start PoatgreSQL

```
postgres -D /usr/local/var/postgres
```


## Create a new database

**NOTE:** We are assuming the database is called `one_db` and the role that should own it be `my_user`. Please replace these accordingly with the actual db and role you want to create.

1. Create the database `one_db`

```
create_db one_db
```

2. Create a role that can access the `one_db`. Please note down this username and password as we will be needing them later.

```
createuser -l -P my_user
> Enter password for new role:
> Enter it again:
```

3. Transfer ownership of the `one_db` to this new user:

```
psql
> ALTER DATABASE one_db OWNER TO my_user;
> \c one_db
> GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO my_user;
```

4. (Optional) Enforce password login for PostgreSQL

Modify `/usr/local/var/postgres/pg_hba.conf` so that the method is set to something other than `trust` for the relevant rows. My configuration looks something like:

```
# TYPE  DATABASE         USER             ADDRESS                  METHOD
local   all              postgres                                  trust
local   all              all                                       md5
host    all              all              127.0.0.1/32             md5
host    all              all              ::1/128                  md5
```

So this new user `my_user` will need a password to access the db server.

**NOTE:** If you made any changes to `/usr/local/var/postgres/pg_hba.conf`, remember to restart PostgreSQL.


## References

- http://exponential.io/blog/2015/02/21/install-postgresql-on-mac-os-x-via-brew/
