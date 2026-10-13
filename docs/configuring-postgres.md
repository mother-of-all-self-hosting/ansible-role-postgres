<!--
SPDX-FileCopyrightText: 2019-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Hardy Erlinger
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020, 2021 Aaron Raimist
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2021 Marc Leuser
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Kim Brose
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up Postgres

This is an [Ansible](https://www.ansible.com/) role which installs [PostgreSQL](https://www.postgresql.org) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

PostgreSQL is a powerful, open-source object-relational database system with over 35 years of active development that has earned it a strong reputation for reliability, feature robustness, and performance.

See the project's [documentation](https://www.postgresql.org/docs/) to learn what PostgreSQL does and why it might be useful to you.

## Adjusting the playbook configuration

To enable Postgres with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# postgres                                                             #
#                                                                      #
########################################################################

postgres_enabled: true

# Put a strong password below, generated with `pwgen -s 64 1` or in another way
postgres_connection_password: ""

########################################################################
#                                                                      #
# /postgres                                                            #
#                                                                      #
########################################################################
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, the Postgres instance becomes available.

Table of contents:

- [Getting a database terminal](#getting-a-database-terminal), for when you wish to execute SQL queries

### Getting a database terminal

> [!WARNING]
> **Modifying the database directly (especially as services are running) is dangerous and may lead to irreversible database corruption.** When in doubt, consider [making a backup](#backing-up-postgresql). You might also want to use a web user interface software like [Adminer](https://www.adminer.org/) and [pgAdmin](https://www.pgadmin.org/).

You can use the CLI installed to the directory specified with `postgres_bin_path` (default: `/postgres/bin/`) for getting interactive terminal access to the Postgres server.

To see the available databases, run `SHOW DATABASES`.

To change to another existing database (for example `miniflux`), run `USE miniflux`.

You can then proceed to write queries. Example: `SELECT COUNT(*) FROM users;`

### Configuring Unix socket support

By default the role is configured to bind-mount the Postgres Unix socket directory to the host at `postgres_run_path`. It can be disabled by setting `postgres_container_unix_socket_enabled` to `false`.

It is also possible to control if helper CLI scripts should prefer Unix socket connections (when available) instead of TCP connection with the `postgres_cli_use_unix_socket_enabled` variable. This does not affect server behavior.

## Maintenance

### Backing up a database

The script to dump the database (`dump-all`) is installed to the directory specified with `postgres_bin_path`. It can dump the database to a path of your choosing.

### Upgrading Postgres

This role is configured to stick to the major and minor version your existing Postgres installation is on (e.g. `10.10` or `10.11`) until you manually upgrade it, as newer Postgres versions cannot start with data generated by older Postgres versions.

This role supports upgrading your existing Postgres installation with the following command:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=upgrade-postgres
```

Running the command migrates all databases, users, etc. on the Postgres server.

>[!NOTE]
> Patch level upgrades (e.g. `10.10.1` -> `10.10.3`) can be conducted without running it.

**Running the command backs up the old Postgres data directory** automatically. When being backing up, the directory is by default renamed to `/postgres/data-auto-upgrade-backup` (or how you/your playbook specifies the path, e.g. `/mash/postgres/data-auto-upgrade-backup`). The auto-upgrade-backup directory stays around forever, until you **manually decide to delete it**.

There are several options which can be set to the command. See below for details.

### Flag: `postgres_auto_upgrade_backup_data_path`

To have the old directory renamed to a different path, pass a flag to the command above, like this: `--extra-vars="postgres_auto_upgrade_backup_data_path=/another/disk/postgres-before-upgrade"`

### Flag: `postgres_dump_dir`

As part of the upgrade, the database is dumped to `/tmp`, an upgraded and empty Postgres server is started, and then the dump is restored into the new server. To use a different directory for the dump, pass a flag to the command above, like this: `--extra-vars="postgres_dump_dir=/directory/to/dump/here"`

### Flag: `postgres_dump_name`

To save disk space in `/tmp`, the dump file is gzipped on the fly at the expense of CPU usage. If you have plenty of space in `/tmp` and would rather avoid gzipping, you can explicitly pass a dump filename which doesn't end in `.gz`. Example: `--extra-vars="postgres_dump_name=postgres-dump.sql"`

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu postgres` (or how you/your playbook named the service, e.g. `mash-postgres`).
