## Introduction
Barman (Backup and Recovery Manager) is an open-source administration tool for disaster recovery of PostgreSQL servers written in Python. It allows your organisation to perform remote backups of multiple servers in business critical environments to reduce risk and help DBAs during the recovery phase.
Barman’s most wanted features include backup catalogs, incremental backup, retention policies, remote recovery, archiving and compression of WAL files and backups.

In this article we're going to deploy this architecture:

<img src="https://user-images.githubusercontent.com/38520491/117682391-3f639b00-b1c8-11eb-9197-1c6867c54252.png" eight="60%" width="60%">

We have 2 VMs. one of them running postgre container and the other is barman VM.

![b1](https://user-images.githubusercontent.com/38520491/117684332-23f98f80-b1ca-11eb-9773-d41f9a7bbd37.png)

***
## Configuration 

Connect to the Barman VM via SSH and install `barman` package:

`apt install barman` or `yum install barman`

Proceed to create a password for `barman` user. You also will be able to don't do this and copy paste `postgre` user SSH public key to `authorized_key` of `Barman` user in Barman VM. 

`passwd barman`

The default (global) `barman` config file is `/etc/barman.conf`. In this file there is a variable called `barman_home`. The default path is `/var/lib/barman`. If you want to store backups and WALs somewhere else, feel free to change it. In our scenario it's `/backup/barman`.

Don't forget about creating directory and it's ownership:

`cd /backup && mkdir barman && chown barman: barman`

After changing this variable, change the default home directory of `barman` in `passwd` file with `usermod` command:

`usermod -d /backup/barman barman`  

Now open `barman` global configuration and change the `barman_home` variable and set `retention_policy` variable to 4. It means keep 4 full backup.

`vim /etc/barman.conf`

```
barman_home = /backup/barman
retention_policy = REDUNDANCY 4
```
Now let's proceed to the postgre VM. You can install `Barman` with `systemd` or with `docker`. If you're going to use postgre with `systemd`, you should install `barman-cli` RedHat or Debian package in your Linux OS. Otherwise if you willing to use `docker`, you should customize a `postgre docker image` and install `barman-cli` in the costume `Dockerfile`. Here we already have a running postgre container, so the easy way is to install `barman-cli` inside the container. This task easily done with a `docker exec` command:

Connect to the postgre container:
 
`docker exec -it postgres bash`

Install requirements packages: 

`apt update && apt install barman-cli openssh-client -y`

Usually we run postgre with a super user called postgre. It has the full privileges on the database. So we change our user to this to configure the postgre.   

`su - postgres`

Because we need to stream WALs from postgre VM to `barman` VM, We need a create a connection between them. So we need to create a `SSH key pair` for postgre user to be able to connect to `Barman` VM. 

Generate the key pair:

`ssh-keygen`

Copy he public key to `Barman` VM:

`ssh-copy-id barman@<Barman_IP> -p <Port_number>`

We need to make sure that the `Barman` can connect to the Postgre as a superuser. You can create a specific superuser in Postgre, named barman, as follows: (It will prompt for password, Remember the password we need it later.)

`createuser -s -P barman`

Because we plan to use WAL streaming or streaming backup, We need to setup a streaming connection. We recommend creating a specific user in Postgre, named `streaming_barman`, as follows: (It will prompt for password, Remember the password we need it later.)

`createuser -P --replication streaming_barman`

Now we should postgre itself for our desired state. Open the main postgre config file `postgre.conf` and add this configurations: 

```
wal_level = 'replica'
max_wal_senders = 10
archive_mode = on
archive_command = 'barman-wal-archive -U barman <Barman_IP> <Server_name> %p'
```
* **wal_level**: determines how much information is written to the WAL. The default value is **replica**, which writes enough data to support WAL archiving and replication, including running read-only queries on a standby server. **minimal** removes all logging except the information required to recover from a crash or immediate shutdown. Finally, **logical** adds information necessary to support logical decoding. Each level includes the information logged at all lower levels. This parameter can only be set at server start.+
* **max_wal_senders**: Specifies the maximum number of concurrent connections from standby servers or streaming base backup clients, or the maximum number of simultaneously running WAL sender processes. The default is zero, meaning replication is disabled. WAL sender processes can not towards the total number of connections. This parameter can only be set at server start. **wal_level** must be set to **archive** or higher to allow connections from standby servers.

* **archive_mode**: When **archive_mode** is enabled, completed WAL segments are sent to archive storage by setting **archive_command**. In addition to **off**, to disable, there are two modes: **on**, and **always**. During normal operation, there is no difference between the two modes, but when set to **always** the WAL archiver is enabled also during archive recovery or standby mode. In **always** mode, all files restored from the archive or streamed with streaming replication will be archived (again).

Next add this line in `pg_hba.conf` (HBA stands for host-based authentication.):

```
host    replication     streaming_barman <Barman_IP>/32       trust
```
Next connect to `Barman` and create a new file like this `/etc/barman.d/pg.conf` and put this lines:

```
[pg] # It's a server name
conninfo = host=<Postgre_IP> user=barman dbname=postgre
streaming_conninfo = host=<Postgre_IP> user=streaming_barman dbname=postgre
backup_method = postgres
streaming_archiver = on
archiver = on
slot_name = barman
create_slot = auto
max_wal_size = 32MB
```
In the home directory of `barman` create a file called `.pgpass`. It should be owned by barman user and group and `0600` permission. Add this lines to it:
```
<Posstgre_IP>:5432:*:barman:<Password>
<Posstgre_IP>:5432:*:streaming_barman:<Password>
```

Now we can restart the postgre container or service and see the results.

In `barman` vm run `barman check <server_name>`. It shows diagnostic information about SERVER_NAME, including: Ssh connection check, Postgre version, configuration and backup directories, archiving  process, streaming process, replication slots, etc.

It may takes a few seconds to everything be ok, Run the command several times with a few seconds interval to see the changes.

It should be like this at the end:
```
barman@Barman:~$ barman check pg
Server pg:
        PostgreSQL: OK
        is_superuser: OK
        PostgreSQL streaming: OK
        wal_level: OK
        replication slot: OK
        directories: OK
        retention policy settings: OK
        backup maximum age: OK (no last_backup_maximum_age provided)
        compression settings: OK
        failed backups: OK (there are 0 failed backups)
        minimum redundancy requirements: OK 
        pg_basebackup: OK
        pg_basebackup compatible: OK
        pg_basebackup supports tablespaces mapping: OK
        systemid coherence: OK
        pg_receivexlog: OK
        pg_receivexlog compatible: OK
        receive-wal running: OK
        archive_mode: OK
        archive_command: OK
        continuous archiving: OK
        archiver errors: OK
```

If something went wrong check the `barman` log file. the default path is `/var/log/barman/barman.log`.

`Barman` will create a directory with name of `server_name` that you specified in the server config file (pg.conf in this example).
If you `cd` to this directory and `ls` (list) the directory, you will see some directories.
```
root@Barman:/backup/barman/pg# ls
base  errors  identity.json  incoming  streaming  wals
```
The `wals` directory holds the WAls. and the `base` directory holds the full backups. To run a full backup run this command: 

`barman backup <server_name>`

It takes a while to complete.

Take your time to explore `man barman` to see different options of this command.

For a daily backup you can create a cronjob with `barman` user like this:

`0 4 * * * barman backup <server_name> >> ~/full_backup.log`

It will take a full backup every night at 4 AM and will keep last 4 full backups.

***
