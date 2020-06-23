## PostgreSQL HA

Install and configure PostgreSQL cluster managed with repmgr. Add dependencies, extensions, databases and users.

#### Requirements

Ansible >=2.9

#### Installation

```bash
ansible-galaxy install fidanf.postgresql-ha
```

#### Example Playbook

```yaml
---
- hosts: pgcluster
  gather_facts: yes
  become: yes
  roles:
    - fidanf.postgresql-ha

```

#### Testing

TODO

#### Example variables

```yaml
# Basic settings
postgresql_version: 12
postgresql_encoding: "UTF-8"
postgresql_locale: "en_US.UTF-8"
postgresql_ctype: "en_US.UTF-8"

postgresql_admin_user: "postgres"
postgresql_default_auth_method: "peer"

postgresql_service_enabled: false # should the service be enabled, default is true

postgresql_cluster_name: "main"
postgresql_cluster_reset: false

# List of databases to be created (optional)
# Note: for more flexibility with extensions use the postgresql_database_extensions setting.
postgresql_databases:
  - name: foobar
    owner: baz          # optional; specify the owner of the database
    hstore: yes         # flag to install the hstore extension on this database (yes/no)
    uuid_ossp: yes      # flag to install the uuid-ossp extension on this database (yes/no)
    citext: yes         # flag to install the citext extension on this database (yes/no)
    encoding: "UTF-8"   # override global {{ postgresql_encoding }} variable per database
    lc_collate: "en_GB.UTF-8"   # override global {{ postgresql_locale }} variable per database
    lc_ctype: "en_GB.UTF-8"     # override global {{ postgresql_ctype }} variable per database

# List of database extensions to be created (optional)
postgresql_database_extensions:
  - db: foobar
    extensions:
      - hstore
      - citext

# List of users to be created (optional)
postgresql_users:
  - name: baz
    pass: pass
    encrypted: yes  # if password should be encrypted, postgresql >= 10 does only accepts encrypted passwords

# List of schemas to be created (optional)
postgresql_database_schemas:
  - database: foobar           # database name
    schema: acme               # schema name
    state: present

  - database: foobar           # database name
    schema: acme_baz           # schema name
    owner: baz                 # owner name
    state: present

# List of user privileges to be applied (optional)
postgresql_user_privileges:
  - name: baz                   # user name
    db: foobar                  # database
    priv: "ALL"                 # privilege string format: example: INSERT,UPDATE/table:SELECT/anothertable:ALL
    role_attr_flags: "CREATEDB" # role attribute flags

# Manage replication with repmgr (optional)
repmgr_target_group: pgcluster # ansible group name of the cluster
repmgr_master: pgsql01 # ansible hostname of the master node

postgresql_ext_install_repmgr: yes
postgresql_wal_level: "replica"
postgresql_max_wal_senders: 10
postgresql_max_replication_slots: 10
postgresql_wal_keep_segments: 100
postgresql_hot_standby: on
postgresql_archive_mode: on
postgresql_archive_command: "test ! -f /tmp/%f && cp %p /tmp/%f"
postgresql_shared_preload_libraries:
  - repmgr

postgresql_users:
  - name: "{{ repmgr_user }}"
    pass: "password"

postgresql_databases:
  - name: "{{ repmgr_database }}"
    owner: "{{ repmgr_user }}"
    encoding: "UTF-8"

postgresql_user_privileges:
  - name: "{{ repmgr_user }}"
    db: "{{ repmgr_database }}"
    priv: "ALL"
    role_attr_flags: "SUPERUSER,REPLICATION"

```

Every other configuration parameter can be found in default variables :
  - [defaults/main.yml](./defaults/main.yml)
  - [defaults/repmgr.yml](./defaults/main.yml)

## Verifying cluster functionality using Ansible ad-hoc command 

```bash
ansible pgcluster -b --become-user postgres -m shell -a "repmgr cluster show"
ansible pgcluster -b --become-user postgres -m shell -a "repmgr cluster crosscheck"
```

License
-------

MIT / BSD