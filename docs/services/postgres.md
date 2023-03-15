# Postgres

[PostgreSQL](https://www.postgresql.org) is a powerful, open source object-relational database system with over 35 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

Many of the services installed by this playbook require a Postgres database.

Enabling the Postgres database service will automatically wire all other services to use it.


## Configuration

To enable this service, add the following configuration to your `vars.yml` file and re-run the [installation](../installing.md) process:

```yaml
########################################################################
#                                                                      #
# devture-postgres                                                     #
#                                                                      #
########################################################################

devture_postgres_enabled: true

# Put a strong password below, generated with `pwgen -s 64 1` or in another way
devture_postgres_connection_password: ''

########################################################################
#                                                                      #
# /devture-postgres                                                    #
#                                                                      #
########################################################################
```

## Importing

### Importing an existing Postgres database from another installation (optional)

Follow this section if you'd like to import your database from a previous installation.

### Prerequisites

The playbook supports importing Postgres dump files in **text** (e.g. `pg_dump > dump.sql`) or **gzipped** formats (e.g. `pg_dump | gzip -c > dump.sql.gz`).

Importing multiple databases (as dumped by `pg_dumpall`) is also supported.

Before doing the actual import, **you need to upload your Postgres dump file to the server** (any path is okay).


### Importing

To import, run this command (make sure to replace `<server-path-to-postgres-dump.sql>` with a file path on your server):

```sh
ansible-playbook -i inventory/hosts setup.yml \
--extra-vars='server_path_postgres_dump=<server-path-to-postgres-dump.sql> postgres_default_import_database=main' \
--tags=import-postgres
```

**Notes**:

- `<server-path-to-postgres-dump.sql>` must be a file path to a Postgres dump file on the server (not on your local machine!)
- `postgres_default_import_database` defaults to `main`, which is useful for importing multiple databases (for dumps made with `pg_dumpall`). If you're importing a single database (e.g. `miniflux`), consider changing `postgres_default_import_database` accordingly


## Maintenance

This section shows you how to perform various maintenance tasks related to the Postgres database server used by various components of this playbook.

Table of contents:

- [Getting a database terminal](#getting-a-database-terminal), for when you wish to execute SQL queries

- [Vacuuming PostgreSQL](#vacuuming-postgresql), for when you wish to run a Postgres [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html) (optimizing disk space)

- [Backing up PostgreSQL](#backing-up-postgresql), for when you wish to make a backup

- [Upgrading PostgreSQL](#upgrading-postgresql), for upgrading to new major versions of PostgreSQL. Such **manual upgrades are sometimes required**.

- [Tuning PostgreSQL](#tuning-postgresql) to make it run faster

### Getting a database terminal

You can use the `/mash/postgres/bin/cli` tool to get interactive terminal access ([psql](https://www.postgresql.org/docs/15/app-psql.html)) to the PostgreSQL server.

By default, this tool puts you in the `main` database, which contains nothing.

To see the available databases, run `\list` (or just `\l`).

To change to another database (for example `miniflux`), run `\connect miniflux` (or just `\c miniflux`).

You can then proceed to write queries. Example: `SELECT COUNT(*) FROM users;`

**Be careful**. Modifying the database directly (especially as services are running) is dangerous and may lead to irreversible database corruption.
When in doubt, consider [making a backup](#backing-up-postgresql).


### Vacuuming PostgreSQL

Deleting lots data from Postgres does not make it release disk space, until you perform a `VACUUM` operation.

To perform a `FULL` Postgres [VACUUM](https://www.postgresql.org/docs/current/sql-vacuum.html), run the playbook with `--tags=run-postgres-vacuum`.

Example:

```bash
ansible-playbook -i inventory/hosts setup.yml --tags=run-postgres-vacuum,start
```

**Note**: this will automatically stop Synapse temporarily and restart it later. You'll also need plenty of available disk space in your Postgres data directory (usually `/mash/postgres/data`).


### Backing up PostgreSQL

To automatically make Postgres database backups on a fixed schedule, see [Setting up postgres backup](configuring-playbook-postgres-backup.md).

To make a one off back up of the current PostgreSQL database, make sure it's running and then execute a command like this on the server:

```bash
/usr/bin/docker exec \
--env-file=/mash/postgres/env-postgres-psql \
mash-postgres \
/usr/local/bin/pg_dumpall -h mash-postgres \
| gzip -c \
> /mash/postgres.sql.gz
```

Restoring a backup made this way can be done by [importing it](#importing).


### Upgrading PostgreSQL

Once this playbook installs Postgres for you, it attempts to preserve the Postgres version it starts with.
This is because newer Postgres versions cannot start with data generated by older Postgres versions.

Upgrades must be performed manually.

This playbook can upgrade your existing Postgres setup with the following command:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=upgrade-postgres
```

**The old Postgres data directory is backed up** automatically, by renaming it to `/mash/postgres/data-auto-upgrade-backup`.
To rename to a different path, pass some extra flags to the command above, like this: `--extra-vars="postgres_auto_upgrade_backup_data_path=/another/disk/mash-postgres-before-upgrade"`

The auto-upgrade-backup directory stays around forever, until you **manually decide to delete it**.

As part of the upgrade, the database is dumped to `/tmp`, an upgraded and empty Postgres server is started, and then the dump is restored into the new server.
To use a different directory for the dump, pass some extra flags to the command above, like this: `--extra-vars="postgres_dump_dir=/directory/to/dump/here"`

To save disk space in `/tmp`, the dump file is gzipped on the fly at the expense of CPU usage.
If you have plenty of space in `/tmp` and would rather avoid gzipping, you can explicitly pass a dump filename which doesn't end in `.gz`.
Example: `--extra-vars="postgres_dump_name=mash-postgres-dump.sql"`

**All databases, roles, etc. on the Postgres server are migrated**.


### Tuning PostgreSQL

PostgreSQL can be tuned to make it run faster. This is done by passing extra arguments to Postgres with the `devture_postgres_process_extra_arguments` variable. You should use a website like https://pgtune.leopard.in.ua/ or information from https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server to determine what Postgres settings you should change.

**Note**: the configuration generator at https://pgtune.leopard.in.ua/ adds spaces around the `=` sign, which is invalid. You'll need to remove it manually (`max_connections = 300` -> `max_connections=300`)

#### Here are some examples:

These are not recommended values and they may not work well for you. This is just to give you an idea of some of the options that can be set. If you are an experienced PostgreSQL admin feel free to update this documentation with better examples.

Here is an example config for a small 2 core server with 4GB of RAM and SSD storage:
```
devture_postgres_process_extra_arguments: [
  "-c shared_buffers=128MB",
  "-c effective_cache_size=2304MB",
  "-c effective_io_concurrency=100",
  "-c random_page_cost=2.0",
  "-c min_wal_size=500MB",
]
```

Here is an example config for a 4 core server with 8GB of RAM on a Virtual Private Server (VPS); the paramters have been configured using https://pgtune.leopard.in.ua with the following setup: PostgreSQL version 12, OS Type: Linux, DB Type: Mixed type of application, Data Storage: SSD storage:
```
devture_postgres_process_extra_arguments: [
  "-c max_connections=100",
  "-c shared_buffers=2GB",
  "-c effective_cache_size=6GB",
  "-c maintenance_work_mem=512MB",
  "-c checkpoint_completion_target=0.9",
  "-c wal_buffers=16MB",
  "-c default_statistics_target=100",
  "-c random_page_cost=1.1",
  "-c effective_io_concurrency=200",
  "-c work_mem=5242kB",
  "-c min_wal_size=1GB",
  "-c max_wal_size=4GB",
  "-c max_worker_processes=4",
  "-c max_parallel_workers_per_gather=2",
  "-c max_parallel_workers=4",
  "-c max_parallel_maintenance_workers=2",
]
```

Here is an example config for a large 6 core server with 24GB of RAM:
```
devture_postgres_process_extra_arguments: [
  "-c max_connections=40",
  "-c shared_buffers=1536MB",
  "-c checkpoint_completion_target=0.7",
  "-c wal_buffers=16MB",
  "-c default_statistics_target=100",
  "-c random_page_cost=1.1",
  "-c effective_io_concurrency=100",
  "-c work_mem=2621kB",
  "-c min_wal_size=1GB",
  "-c max_wal_size=4GB",
  "-c max_worker_processes=6",
  "-c max_parallel_workers_per_gather=3",
  "-c max_parallel_workers=6",
  "-c max_parallel_maintenance_workers=3",
]
```