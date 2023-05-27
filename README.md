# MongoDB

An Ansible role that installs, configures and manages MongoDB for EL 8.  
A role for Ubuntu can be [found here](https://github.com/csuka/ansible_role_mongodb_ubuntu).

**Please read this file carefully before deploying this Ansible role**

## Functionalities

* Applies recommended production notes, e.g. [numa](https://docs.mongodb.com/manual/administration/production-notes/#configuring-numa-on-linux) and [disables transparent hugepages](https://docs.mongodb.com/manual/tutorial/transparent-huge-pages/)
* Bootstrapping a cluster in a PSA architecture (primary, secondary, arbiter)
   * includes cluster verification
* Secures connection by encrypting traffic via a keyfile, auto generated
* Install either Community or the Enterprise edition
* Easily to configure, with a future proof configuration method
* Update playbook, supports patch releases
* Add user defined users
* Add user defined databases
* Backup with mongodump
* Logrotation, set from within mongo

## Requirements

* Brain
* On the controller node, ensure the [mongodb collection](https://docs.ansible.com/ansible/latest/collections/community/mongodb/index.html) is installed
* The hosts are able to connect to each other, with preferably hostnames, and the set port, default 27017
* Keep in mind there is enough disk space for data disk, and the backup if set

## Assertions

There are several assertions to ensure a minimal valid configuration. Of course, it does not cover every use case, but most of them.
Please follow this readme carefully. As a hint, for valid a configuration, see the variables files in the molecule folder.

## Versioning and edition

The version and edition can be set. By default, the official mongodb repository and gpg key are added.

```yaml
mongo_repo: true
mongo_version: 5.0
mongo_edition: org  # or enterprise
```

## Recommendations

This section refers to this [official production notes of MongoDB](https://docs.mongodb.com/manual/administration/production-checklist-operations/#linux).

This role includes several configuration recommendations, but not all. There is for example: "Turn off atime for the storage volume containing the database files." Such tasks are out of scope for this role.

There are tasks which this role does apply if set, these are:

  * Start MongoDB with numactl, set in systemd file
  * Disabling zone reclaimm
  * Disabling transparent hugepages
  * Configure tuned profile

See `tasks/thp.yml` and `tasks/numa.yml` for the actual changes to the system.

These configuration recommendations are applied by default.

```yaml
mongo_thp: true
mongo_numa: true
```

## Configuration variables

First, see `defaults/main.yml`.

The configuration file is placed at `/etc/mongod.conf`.

The values set in the Ansible configuration are set **exactly** to the configuration file on the host. Only the keys are pre-defined. Example:

```yaml
# Variable set in Ansible configuration
mongo_operationprofiling:
  my_Value:
    anotherValue: something
```

Will result in the configuration file on disk:

```yaml
operationProfiling:
  my_Value:
    anotherValue: something
```

If the key is set to an empty string, it will be commented out on the configuration file on disk.

The possible keys to set are:
```yaml
mongo_systemlog
mongo_storage
mongo_processmanagement
mongo_security
mongo_operationprofiling
mongo_replication
mongo_sharding
mongo_auditlog
mongo_snmp
```

There are pre-defined values, which are default for Mongo.
If for some reason, it is desired to set custom key/values, use:

```yaml
mongo_custom_cnf:
  my_key:
    my_value: true
```

## Authorization

By design there are 3 users created: `admin`, `backup` and `adminuser`.
The admin has role root, the backup user has role backup and adminuser has role userAdminAnyDatabase.

```yaml
mongo_admin_pass: 'change_me'
mongo_adminuser_name: adminuser
mongo_adminuser_pass: 'change_me'
```

## Databases and users

Taken from [the docs](https://docs.ansible.com/ansible/latest/collections/community/mongodb/mongodb_user_module.html), define your own set of users/databases.

See the docs for [the possible roles](https://docs.mongodb.com/manual/reference/built-in-roles/).

Users and databases are always configured via localhost, via user admin and database admin.
When a cluster configured, configuring database and users is executed from the primary host.

Set with:
```yaml
mongo_user:
# in it's most simple form
  - database: my_db
    name: my_user

# standard values
  - database: another_db
    name: another_user
    update_password: on_create
    password: "123ABC!PASSWORD_XYZ"
    roles: 'readWrite,dbAdmin,userAdmin'
    state: present

# all possible variables
  - database: my_db
    name: someone
    password: my_password
    update_password: on_create             # default always
    roles: 'readWrite,dbAdmin,userAdmin'   # default omitted
    state: absent                          # default present
    ssl: true                              # default omitted
    ssl_ca_certs: /path/to/ca_certs        # default omitted
    ssl_cert_reqs: CERT_REQUIRED           # default omitted
    ssl_certfile: /path/to/ssl_certfile    # default omitted
    ssl_crlfile: /path/to/ssl_crlfile      # default omitted
    ssl_keyfile: /path/to/ssl_keyfile      # default omitted
    ssl_pem_passphrase: 'something'        # default omitted
    auth_mechanism: PLAIN                  # default omitted
    connection_options: my_conn_options    # default omitted
    create_for_localhost_exception: value  # default omitted
```

To keep the role idempotent, you should set the value for `update_password` to `on_create`. Only when actually updating a password, set it to `always`, then switch back to `on_create`.


## Clustering

When there are multiple hosts in the play, Ansible assumes clustering is configured.

Clustering in a PSA (primary/secondary/arbiter) architecture is possible, or a primary/secondary/secondary setup.

For security reasons, the connection between the hosts is secured with a keyfile.

The [keyfile is automatically generated and deemed secure](https://docs.mongodb.com/manual/tutorial/enforce-keyfile-access-control-in-existing-replica-set/).

```yaml
mongo_security:
  keyFile: /etc/keyfile_mongo
```

A 2 host cluster is a fundamentally broken design, as it cannot maintain uptime without a quorum and the ability of a node to go down to aid recovery.
When there are exactly 2 hosts in the play, forming a cluster is not possible.
Mongo will not cluster, because there must be a valid number of replicaset members.

There are assertions in place to verify an uneven amount of hosts in the play.

Configure with:
```yaml
# set the role per host. in the host_vars
mongo_replication_role: primary # or secondary, or arbiter

# in group_vars/all.yml
mongo_replication:
  replSetName: something
```

There can be only 1 primary and 1 arbiter set.

You shouldn't change the design of the cluster once it has been deployed, e.g. change a secondary to an arbiter.

Ensure to [disable read concern majority](https://docs.mongodb.com/v4.0/reference/read-concern-majority/#disable-read-concern-majority) when version 4.4 or lower is installed.

```yaml
# not for version 5.0 or higher
# and only when the architecture is PSA (Primary, Secondary, Arbiter)
mongo_replication:
  replSetName: something
  enableMajorityReadConcern: false
```

### Arbiter

As per the [docs](https://docs.mongodb.com/manual/tutorial/add-replica-set-arbiter/#add-an-arbiter), avoid deploying more than one arbiter per replica set.

Ansible will take care of adding the arbiter properly to the cluster.

```yaml
mongo_replication_role: arbiter
```

## Backup

The back-up is configured to be set on either on a single host without replication, or on the first secondary host in the play. It is performed with [`mongodump`](https://docs.mongodb.com/manual/reference/program/mongodump/), set in cron.

The backup scripts are **only** placed on the first applicable secondary node:

```
- host1 [primary]   <-- backup scripts absent here
- host2 [secondary] <-- backup scripts placed here
- host3 [secondary] <-- backup scripts absent here
```

```yaml
mongo_backup:
  enabled: true
  dbs:
    - admin
    - config
    - local
  user: backup
  pass: change_me
  path: /var/lib/mongo_backups
  owner: mongod
  group: mongod
  mode: '0660'
  hour: 2
  minute: 5
  day: "*"
  retention: 46  # in hours
```

Ensure to change the password of the backup user, and allow the backup user to actually backup a given database.

On disk, the result will be:
```bash
[root@test-multi-03 mongo_backups]# pwd ; ls -larth
/var/lib/mongo_backups
total 4.0K
drwxr-xr-x. 36 root   root   4.0K Jan 20 12:33 ..
lrwxrwxrwx   1 root   root     46 Nov 20 12:37 latest -> /var/lib/mongo_backups/mongo_backup_2021-11-20
drw-rw----   3 mongod mongod   51 Nov 20 12:37 .
drwxr-xr-x   5 root   root     77 Nov 20 12:38 mongo_backup_2021-11-20
```

## Logrotation

Please [read the docs](https://docs.mongodb.com/manual/tutorial/rotate-log-files/). Ensure settings are configured properly.

## Updating

Before updating, ensure proper testing is in place. Begin with reading the latest changes in the official docs.

There is a separate update playbook, see `playbooks/update.yml`. Set the correct hostname in place and simply run the playbook.

Updating can easily be done for patch versions, e.g. from 5.0.1 to 5.0.2.
This role does not include major versioning upgrades due to breaking changes on each release and other obvious reasons. If this is desired, the logic is in place in this role to perform a major upgrade, you can easily built it yourself.

New variables can be set when updating mongo. To update:

```yaml
mongo_state: latest

# setting new variables is possible when updating mongo
mongo_net:
  new_variable: true
```

While updating, ensure applications don't write to mongo. Also, ensure a backup is created beforehand.

### Updating a replica set

If a replication set is active, the cluster should be maintained after the update. As taken [from the docs](https://docs.mongodb.com/manual/tutorial/upgrade-revision/), the following steps are executed:

```
 - verify cluster health, if ok, continue

 - shutdown mongo application on arbiter if present
 - update mongo on arbiter
 - place config on arbiter
 - start mongo on arbiter

 - wait until cluster health is ok

 - shutdown mongo application on a secondary
 - update mongo on secondary
 - place config on secondary
 - start mongo on secondary

 - wait until cluster health is ok

 - repeat for remaining secondaries

 - step down primary
 - update mongo on original primary
 - place config on original primary
 - start mongo on original primary

 - wait until cluster health is ok
```

### Updating a sharded environment

In development.

## Development

 * There is a reset playbook to remove all mongo files. This is useful for development purposes, see `tasks/reset.yml`. It is commented out by design

### Scaling

Still in development... I am not even sure whenever I'll this functionality, since this is currently not even possible with mongo 5.0.
It is not easy to configure scaling in mongo with Ansible, since the method is not straight forward.

So far, I saw that the steps should be:
- If arbiter is present in configuration and on system
    - remove arbiter from cluster
- Add new secondary or secondaries
- Add arbiter if configured

I have tried configuring this countless amount of times, but always failed due to a system error. I decided to not include scaling for now.

# Example playbook

```yaml
- hosts:
    - host_mongo_primary
    - host_mongo_secondary
    - host_mongo_arbiter
  roles:
    - mongodb
  any_error_true: true
  vars:
    mongo_restart_config: true
    mongo_restart_seconds: 8
    mongo_thp: true
    mongo_numa: true
    mongo_replication:
      replSetName: replicaset1
    mongo_security:
      authorization: enabled
      keyFile: /etc/keyfile_mongo
    mongo_admin_pass: something
    mongo_adminuser_pass: something
    mongo_net:
      bindIp: 0.0.0.0
      port: 27017
    mongo_systemlog:
      destination: file
      logAppend: true
      path: /opt/somewhere/mongod.log
    mongo_storage:
      dbPath: /opt/mongo/
      journal:
        enabled: true
    mongo_user:
      - database: burgers
        name: bob
        password: 12345
        state: present
        update_password: on_create    
  pre_tasks:
    # ensure this is done
    # - name: ensure hosts can connect to each other via hostnames
    #   template:
    #     src: hosts.j2
    #     dest: /etc/hosts
```
