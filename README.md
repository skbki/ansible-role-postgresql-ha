## PostgreSQL HA

[![Build Status](https://travis-ci.com/fidanf/ansible-role-postgresql-ha.svg?branch=master)](https://travis-ci.com/fidanf/ansible-role-postgresql-ha)

Install and configure a PostgreSQL high-availability cluster managed with repmgr. Add dependencies, extensions, databases and users. Works for standalone installations as well.

Tested with :
  - Debian 10.x :heavy_check_mark:
  - Ubuntu 18.04.x :heavy_check_mark:

---

- [PostgreSQL HA](#postgresql-ha)
  - [Requirements](#requirements)
  - [Installation](#installation)
  - [Example Inventory](#example-inventory)
  - [Role variables](#role-variables)
  - [Example Playbook](#example-playbook)
- [Usefull commands to run after your first installation](#usefull-commands-to-run-after-your-first-installation)
  - [Verifying cluster functionality](#verifying-cluster-functionality)
  - [Show cluster status](#show-cluster-status)
  - [List nodes and their attributes](#list-nodes-and-their-attributes)
- [Register former primary as a standby node after automatic failover](#register-former-primary-as-a-standby-node-after-automatic-failover)
- [License](#license)

### Requirements

Python >=3.6
The role is compatible with Ansible >=2.10 but hasn't yet be tested with Ansible 3.x.

See [./requirements.txt](./requirements.txt) for detailled dependencies used to develop the role.

**Recommended, for each postgresql host**
- Python3 in PATH
- Pip3 in PATH
- [psycopg2-binary](https://pypi.org/project/psycopg2-binary/) (itself requires libpq-dev apt-package)

You can take a look at [prepare.yml](molecule/default/prepare.yml) to check out an example setup for Python 3.

### Installation

```bash
ansible-galaxy install fidanf.postgresql_ha
```

### Example Inventory

```ini
[pgcluster]
pgsql01     ansible_host=192.168.56.10  repmgr_node_id=1 repmgr_priority=3
pgsql02     ansible_host=192.168.56.11  repmgr_node_id=2 repmgr_priority=2
pgsql03     ansible_host=192.168.56.12  repmgr_node_id=3 repmgr_priority=1

```
### Role variables

Role default variables are split amongst two files :
  - [001-postgresql.yml](./defaults/main/001-postgresql.yml)
  - [002-repmgr.yml](./defaults/main/002-repmgr.yml)

In order to exactly figure out the purpose and valid values for each of these variables, do not hesitate to inspect all the Jinja templates in the [templates](./templates) directory. Original default configuration files are also included (`.orig`).

### Example Playbook

```yaml
---
- hosts: pgcluster
  gather_facts: yes
  become: yes
  roles:
    - name: fidanf.postgresql-ha
      vars:
        # Required configuration items
        repmgr_target_group: pgcluster
        repmgr_master: pgsql01
        repmgr_failover: automatic
        repmgr_promote_command: /usr/bin/repmgr standby promote -f /etc/repmgr.conf --log-to-file
        repmgr_follow_command: /usr/bin/repmgr standby follow -f /etc/repmgr.conf --log-to-file --upstream-node-id=%n
        repmgr_monitoring_history: "yes"
        repmgr_connection_check_type: query
        repmgr_log_level: DEBUG
        repmgr_reconnect_attempts: 2
        repmgr_reconnect_interval: 10
        # Basic settings
        postgresql_version: 11
        postgresql_cluster_name: main
        postgresql_cluster_reset: false # TODO: Needs to be tested for repmgr
        postgresql_listen_addresses: "*"
        postgresql_port: 5432
        postgresql_wal_level: "replica"
        postgresql_max_wal_senders: 10
        postgresql_max_replication_slots: 10
        postgresql_wal_keep_segments: 100
        postgresql_hot_standby: on
        postgresql_archive_mode: on
        postgresql_archive_command: "test ! -f /tmp/%f && cp %p /tmp/%f"
        postgresql_ext_install_repmgr: yes
        postgresql_shared_preload_libraries:
          - repmgr
        # postgresql logging 
        postgresql_log_checkpoints: on
        postgresql_log_connections: on
        postgresql_log_disconnections: on
        postgresql_log_temp_files: 0
        # pg_hba.conf
        postgresql_pg_hba_custom:
          - { type: "host", database: "all", user: "all", address: "192.168.56.0/24", method: "md5" }
          - { type: "host", database: "replication", user: "{{ repmgr_user }}", address: "192.168.56.0/24", method: "trust" }  
          - { type: "host", database: "replication", user: "{{ repmgr_user }}", address: "127.0.0.1/32", method: "trust" }  
          - { type: "host", database: "{{ repmgr_database }}", user: "{{ repmgr_user }}", address: "127.0.0.1/32", method: "trust" }  
          - { type: "host", database: "{{ repmgr_database }}", user: "{{ repmgr_user }}", address: "192.168.56.0/32", method: "trust" }  
        # Databases
        postgresql_databases:
          - name: "{{ repmgr_database }}"
            owner: "{{ repmgr_user }}"
            encoding: "UTF-8"
          - name: testdb
            owner: admin
            encoding: "UTF-8"
        # Users
        postgresql_users:
          - name: "{{ repmgr_user }}"
            pass: "{{ repmgr_password }}"
          - name: admin
            pass: secret # postgresql >=10 does not accept unencrypted passwords
            encrypted: yes
        # Roles
        postgresql_user_privileges:
          - name: "{{ repmgr_user }}"
            db: "{{ repmgr_database }}"
            priv: "ALL"
            role_attr_flags: "SUPERUSER,REPLICATION"
          - name: admin
            db: testdb
            role_attr_flags: "SUPERUSER"

```

## Usefull commands to run after your first installation

### Verifying cluster functionality

```bash
ansible pgcluster -b --become-user postgres -m shell -a "repmgr cluster crosscheck"
```

### Show cluster status

```bash
ansible pgcluster -b --become-user postgres -m shell -a "repmgr cluster show"
```

### List nodes and their attributes

```bash
ansible pgcluster -b --become-user postgres -m shell -a "repmgr node status"
```

## Register former primary as a standby node after automatic failover

```
postgres@pgsql01:~$ pg_ctlcluster 11 main stop
postgres@pgsql01:~$ repmgr standby clone --force -h pgsql02 -U repmgr -d repmgr
postgres@pgsql01:~$ pg_ctlcluster 11 main start
postgres@pgsql01:~$ repmgr standby register --force
```

Or you may use the repmgr node rejoin with [pg_rewind](https://repmgr.org/docs/current/repmgr-node-rejoin.html#REPMGR-NODE-REJOIN-PG-REWIND) 

```bash
repmgr node rejoin -d repmgr -U repmgr -h pgsql02 --verbose --force-rewind=/usr/lib/postgresql/11/bin/pg_rewind
```

## License

MIT / BSD
