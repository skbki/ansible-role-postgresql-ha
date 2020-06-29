## PostgreSQL HA

[![Build Status](https://travis-ci.com/fidanf/ansible-role-postgresql-ha.svg?branch=master)](https://travis-ci.com/fidanf/ansible-role-postgresql-ha)

Install and configure a PostgreSQL high-availability cluster managed with repmgr. Add dependencies, extensions, databases and users. Works for standalone installations as well.

Tested with :
- Debian 10.x :heavy_check_mark:
- Ubuntu 18.04.x :heavy_check_mark:

#### Requirements

- Ansible >=2.9

**Recommended, for each postgresql host**
- Python3 in PATH
- Pip3 in PATH
- [psycopg2-binary](https://pypi.org/project/psycopg2-binary/) (itself requires libpq-dev apt-package)

You can take a look at [prepare.yml](molecule/default/prepare.yml) to check out an example setup for Python 3.

#### Installation

```bash
ansible-galaxy install fidanf.postgresql-ha
```

#### Example Inventory

```ini
[pgcluster]
pgsql01     ansible_host=192.168.56.10  repmgr_node_id=1
pgsql02     ansible_host=192.168.56.11  repmgr_node_id=2
pgsql03     ansible_host=192.168.56.12  repmgr_node_id=3
```
#### Role variables

Role default variables are split amongst two files :
  - [001-postgresql.yml](./defaults/main/001-postgresql.yml)
  - [002-repmgr.yml](./defaults/main/002-repmgr.yml)

In order to exactly figure out the purpose and valid values for each of these variables, do not hesitate to inspect all the Jinja templates in the [templates](./templates) directory. Original default configuration files are also included (`.orig`).

#### Example Playbook

```yaml
---
- hosts: pgcluster
  gather_facts: yes
  become: yes
  roles:
    - name: fidanf.postgresql-ha
      vars:
        # Required configuration items
        repmgr_target_group: "pgcluster"
        repmgr_target_group_hosts: "{{ groups[repmgr_target_group] }}"
        repmgr_master: "pgsql01"
        # Basic settings
        postgresql_version: 11
        postgresql_cluster_name: main
        postgresql_cluster_reset: false # TODO: Needs to be tested for repmgr
        postgresql_port: 5432
        postgresql_encoding: "UTF-8"
        postgresql_locale: "fr_FR.UTF-8"
        postgresql_ctype: "fr_FR.UTF-8"
        postgresql_admin_user: "postgres"
        postgresql_default_auth_method: "peer"
        postgresql_listen_addresses: "*"
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
          - name: testdb
            owner: admin
            encoding: "UTF-8"
          - name: "{{ repmgr_database }}"
            owner: "{{ repmgr_user }}"
            encoding: "UTF-8"
        # Users
        postgresql_users:
          - name: admin
            pass: secret # postgresql >=10 does not accept unencrypted passwords
            encrypted: yes
          - name: "{{ repmgr_user }}"
            pass: "{{ repmgr_password }}"
        # Roles
        postgresql_user_privileges:
          - name: admin
            db: testdb
            role_attr_flags: "SUPERUSER"
          - name: "{{ repmgr_user }}"
            db: "{{ repmgr_database }}"
            priv: "ALL"
            role_attr_flags: "SUPERUSER,REPLICATION"

```

## Verifying cluster functionality using Ansible ad-hoc command 

```bash
ansible pgcluster -b --become-user postgres -m shell -a "repmgr cluster show"
ansible pgcluster -b --become-user postgres -m shell -a "repmgr cluster crosscheck"
```

License
-------

MIT / BSD
