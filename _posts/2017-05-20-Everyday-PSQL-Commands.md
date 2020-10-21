---
title: Everyday PSQL Commands
search: true
tags: 
  - database
  - PostgreSQL
toc: true
toc_label: "My Table of Contents"
toc_icon: "cog"
classes: wide
---

PostgreSQL Cheat Sheet.

## Contents

- psql query Parameters
- Common connect and query
- Query Postgres db version
- Pg dump and restore

## Why do you need schems

- Schemas allow you to organize database objects e.g., tables into logical groups to make them more manageable.
- Schemas enable multiple users to use one database without interfering with each other.

## PSQL Query Parameters

- `-d`, `–dbname=DBNAME` database name
- `-h`, `–host=HOSTNAME` database server hostname or ip
- `-t`, a table
- `-p`, `–port=PORT` database server port number (default: 5432)
- `-U`, `–username=NAME` connect as specified database user
- `-W`, `–password` force password prompt
- `–role=ROLENAME` do SET ROLE

## Common commands and queries

### Connect to PostgreSQL database

```bash
psql -U user_name template1;

psql -h localhost -p 5432 -U user_name template1;
```

Switch to another db

```sql
\c template0  -- without username
\c template0 user_name  -- with username
```

### List and descriptions

1. List available databases `\l`
2. List available tables `\dt`

    ```sql
    -- find that table name contains 'table_nam', case insensitive
    template1=> \dt *table_nam*
    ```

3. Describe a table `\d table_name`
4. List available schema `\dn`
5. List available functions `\df`. Use `\df+` or `\ef` to view or edit a founction.
6. List available views `\dv`
7. List users and their roles `\du`
8. List sequences in current schema: `\ds`, or

    ```sql
    SELECT c.relname FROM pg_class c WHERE c.relkind = 'S';

    select sequence_schema, sequence_name from information_schema.sequences;
    ```

9. List Command history `\s`
    save commandline history to a file named history.txt

    ```sql
    \s moss.cmd.txt
    ```

10. Execute psql commands from a file `\i`
11. Turn on and off query execution time `\timing`

    ```sql
    template1=> \timing
    Timing is on.
    template1=> select now();
                  now
    -------------------------------
     2046-10-15 17:55:52.075797+13
    (1 row)

    Time: 20.232 ms
    template1=> \timing
    Timing is off.
    ```

12. Edit command in your own editor `\e`. It will open vim.
13. Create or View/Edit a function in the editor
    - `\ef` It generates an editable function template.

        ```sql
        CREATE FUNCTION ( )
         RETURNS
         LANGUAGE
         -- common options:  IMMUTABLE  STABLE  STRICT  SECURITY DEFINER
        AS $function$

        $function$
        ```

    - `\ef myFuncName` It opens a vim to view **existing** function.
    - After editing a function, you shall execute the updated function

        ```sql
        \ef function_name()
            [edit function and save]
        \g
        ```

14. Quit psql `\q`

### Query current schmea, or search_path

```sql
SHOW search_path;

SELECT current_schema();
```

### Query current psql version

```sql
template1=> SELECT version();
-- PostgreSQL 9.6.14 on x86_64-pc-linux-gnu (Debian 9.6.14-1.pgdg90+1), compiled by gcc (Debian 6.3.0-18+deb9u1) 6.3.0 20170516, 64-bit

template1=> SELECT current_setting('server_version_num')
-- 90614 (9.6.14)

template1=> SHOW server_version_num;
--  90614 (9.6.14)
```

### How can I query if a column exists in a table using an SQL statement

```sql
SELECT column_name
FROM information_schema.columns
WHERE table_name='your_table' and column_name='your_column';

select count(1)
from events
where time between (now() - '1 week'::interval) and (now() - '2 weeks'::interval);
```

### How to query records with timestamp

```sql
SELECT *
FROM table
WHERE update_date >= '2013-05-03'::date
AND update_date < ('2013-05-03'::date + '1 day'::interval);
```

### How to count the number of occurrences of a given substring

```sql
SELECT *
FROM table
WHERE (length(hey_field) - length(replace(hey_field, 'needle_field', '')) = occurrence);
```

### How to rename a database

Before renaming a database, you have to make sure there is no active connections to the database.
If there are active connections and you need to rename it ASAP, you need to talk to the connection owners.
You also need to modify the connection string related to the old database and change it to the new one.

```psql
-- query
SELECT  * FROM pg_stat_activity WHERE datname = 'db_name';

-- terminate pid
SELECT pg_terminate_backend (pid) FROM pg_stat_activity WHERE datname = 'db_name';

-- name
ALTER DATABASE db RENAME TO new_db_name;
```

## PostgreSQL REINDEX (since 9.3)

```sql
REINDEX [ ( VERBOSE ) ] { INDEX | TABLE | SCHEMA | DATABASE | SYSTEM } name;

REINDEX INDEX index_name;
REINDEX TABLE table_name;
REINDEX SCHEMA schema_name;
REINDEX DATABASE database_name;
REINDEX SYSTEM database_name;
```

The 'VERBOSE' keyword is optional. When included, the statement displays a progress report when each index is reindexed.

## Backup and Restore

```sql
pg_dump -d <databasename> -h <hostname> -p <port> -n <schemaname> -f <location of the dump file>
```

`pg_restore` is a utility for restoring a PostgreSQL database from an archive created by pg_dump in one of the **non-plain-text formats**.

It will issue the commands necessary to reconstruct the database to the state it was in **at the time it was saved**.

The dump and restore processes should be in one transaction. This mode can be specified by passing the `-1` or `--single-transaction` command-line options to `psql`.

### For one table

```bash
root@postgresql-local:/$ pg_dump -U user_name -d template -n public -t users > template_public_users.psql

root@postgresql-local:/$ psql -U user_name -d org < org_public_users.psql -- restore
```

### For one Schema

```bash
pg_dump -U user_name -d template1 -n schema_name > schema_name.dmp
```

### For one Database

```bash
pg_dump -U user_name dbname > dbname.dmp
```

Since a database is too big, you may want to compress a large db

```bash
pg_dump -U postgres -d database_name | gzip > database_name.gz
gunzip -c database_name.gz | psql -U postgres database_name
```

### Export and import schema and table definition

1. schema definition without data

    ```bash
    pg_dump -U postgres -s -d myDatabase -n my_schema > my_schema_dump.txt
    ```

2. We need the customers definition because
    1. We need to reuse definition while creating new customers table
    2. we want to make sure new customers table remains the same definition

    ```bash
    pg_dump -U postgres -s -d myDatabase -n my_schema -t my_schema.customers > customers_dump.txt
    ```

## PSQL Stored Procedures

### DO $$ Block

1. Excute sql

    ```sql
    Do $$
        Begin
           execute format('drop database %I ', 'bak_20180404_2014_12');
        End
    $$ ;
    ```

2. Excute based on conditions

    ```sql
    DO
        $do$
        BEGIN
            IF EXISTS (SELECT FROM orders) THEN
               DELETE FROM orders;
            ELSE
               INSERT INTO orders VALUES (1,2,3);
            END IF;
        END
    $do$
    ```

    - Need a `;` at the end of each statement, except for the final `END`
    - need `END IF;` at the end of the `IF` statement.

## PSQL Logs

```bash
\$ cd /var/log/postgres/9.2
\$ ls -alht

-rw-------. 1 postgres postgres 2.4M Jul  3 08:36 postgresql-Thu.log
-rw-------. 1 postgres postgres 7.7M Jul  2 09:59 postgresql-Wed.log
-rw-------. 1 postgres postgres 2.5M Jul  1 09:59 postgresql-Tue.log
-rw-------. 1 postgres postgres 4.9M Jun 30 09:59 postgresql-Mon.log
-rw-------. 1 postgres postgres 1.3M Jun 29 09:59 postgresql-Sun.log
-rw-------. 1 postgres postgres 968K Jun 28 09:59 postgresql-Sat.log
-rw-------. 1 postgres postgres 2.5M Jun 27 09:59 postgresql-Fri.log

\$ sudo less postgresql-Sun.log | grep haha
```

## Reference

1. [PSQL 24.1. SQL Dump](https://www.postgresql.org/docs/9.1/backup-dump.html)
2. [17 Practical psql Commands That You Don’t Want To Miss](http://www.postgresqltutorial.com/psql-commands/)
3. [5 Tips to Backup and Restore Database in PostgreSQL](https://tecadmin.net/backup-and-restore-database-in-postgresql/)
4. [postgresql-if-statement](https://stackoverflow.com/questions/11299037/postgresql-if-statement/)
5. [PostgreSQL Schema](https://www.postgresqltutorial.com/postgresql-schema/)
