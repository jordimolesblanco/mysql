# Why would you do this at all?

Good question! I wouldn't do this just for fun.

The goal is to migrate from an old version of MySQL to the newest available (or at least to a newer one).
- First you copy the data to a newer version of MySQL/MariaDB/Percona.
- Then you setup replication.
- Then you make clones of the new replica and do as much testing as you please. Don't run tests on the replica, since
doing so will break replication.
- Then you turn the new replica into master and point your application to the new server.
- Finally you enjoy your new MySQL with very little downtime.

Setting up MySQL replication is very well documented elsewhere. You can also find documentation about replication
between different MySQL versions as long as they are only 1 major version a part, for example: MySQL 5.5 to MySQL 5.6.
But at the time of writing this, I found very little information about attempting more challenging migrations.

There's a lot of discussion about how dangerous migrations are and that you should never attempt jumping more than 1
version of MySQL at a time.
Unfortunately, in IT you don't always get the green light to do a proper maintenance window for as long as you need
to do the "right thing", so this is just a tested and viable option to work around this problem.

So, this example will show you how I achieved the migration from MySQL 5.5 (2010) to MariaDB 10.2 (2017).
You will hear that things like this must never be attempted and that you might loose all your data.
Well... **_enter this area at your own peril_**, I will not be made responsible for any thing you break or
any data you loose.

I will however say a few things about 'here be dragons' messages:

- After all, this is only a read-replica. If the tests with the new version fail, no one is going to force you to
use it in production. Trying and failing is part of the process. You might as well migrate moving through all the
MySQL version there are if you encounter problem or you have more time.
- Let's say after setting up the replica, you do some testing and everything works fine, but then something goes wrong.
What are backups or DRs for then? 
I've done this a few times and usually the controlled risk of an unscheduled downtime is much
more appealing than stopping everything for a set number of hours, warn (and give explanations to) all users/clients,
etc.
With proper backups and/or DR systems, you wouldn't have to worry about any data loss at all.
- MySQL, Percona, MariaDB will strongly suggest you should never try this, warning of unknown consequences and that
replication between SQL engines that are more than 1 major version away is not tested. Well, they are right saying that
they haven't tested all the versions and configurations, and it would probably be a waste of time for them, but it
doesn't mean it can't work.
- **_Last but not least_**, I would never recommend doing this for more than necessary (hours or days). I wouldn't
go on Holidays for 2 weeks the day after I went live with the new version.
This needs proper monitoring and quick decision making if necessary.

# Steps to setup replication

## Initial steps depending on data source

#### Do this if you have an old replica already, otherwise go to [I don't have any slave](#do-this-if-you-have-no-slaves)

Great news! Most likely, you'll be able to copy the data from your existent replica without any downtime on the master.

Follow these simple steps:

1. Create a new box with MySQL version you want.
2. Stop the old replica and copy the data in /var/lib/mysql to the new box. You can probably afford stopping the
replica for a while until all the data is copied over, but if it's not the case see
[Copy Data To Slave](#copy-data-to-slave) section.

#### Do this if you have no slaves

If you don't have any slave server yet, you will have to bring MySQL down and change your my.cnf to enable
replication. However, don't take the opportunity to change other values in the old server,
leave everything else as it is. Probably your system is working ok(ish) as it is and if you are going to get rid of the
old MySQL anyway... why risk it?
Enabling replication in your master server should not break anything.

Follow these simple steps:

1. Prepare a new MySQL/MariaDB/Percona server. In my case it was attempted with MariaDB10.2, but other versions
newer than MySQL 5.5 should probably follow the same steps I'm indicating in this document.
Versions >=5.6 contain several changes that affect replication.
2. If not present already, add these three values under the [mysqld] section in my.cnf in your master server:
  - server-id = 1 (or whatever the value you want, it's only an id but it helps to know that 1 is master)
  - log_bin = /var/lib/mysql/mysql-bin.log (the path can be anything, but usually we keep the default)
  - binlog-format = MIXED (you can really choose any format available, but MIXED is recommended. For this particular
   configuration MySQL 5.5 -> MariaDB 10.2 other formats didn't work.
   [More info here](https://dev.mysql.com/doc/refman/5.7/en/replication-options-binary-log.html#sysvar_binlog_format)
3. Restart your master server. It should take less than 1 minute to do so if the server is not busy. You'd better
pick a valley period for you to run this, just in case it takes longer than expected.
4. Copy the data from master to the new slave. You might consider using **_mysqldump_** if the combined size of all
your databases is only a few GBs, but in any other situation, you'll have to use other tools. See
[Copy Data To Slave](#copy-data-to-slave) section.

## Next Steps

1. Make sure the my.cnf file in the new replica box contains the options that will allow it to make replication work.
(See [Slave Configuration Part 1](#slave-configuration-part-1) section)
2. Start the new MySQL.
    - If MySQL doesn't start properly, grep for mysql errors in the syslog log file (or similar file for your OS).
3. If MySQL starts, before you do anything else,run the mysql_upgrade command. This will "transform" the "old"
databases and tables into a format that the new engine will be 100% compatible with. Usually it will change the own
MySQL database (mysql) and its structure, but it might also change other tables. MySQL might be able to work and
serve data without running this, but replication will not work.
This I found very strange. Initially, MySQL was starting and it was able to serve data (even without the upgrade),
but replication was failing with rather uninformative error messages. I decided not to run the upgrade to make it
easier for MySQL to replicate since I thought it would make more sense to leave the data structure untouched for the
replication between such different versions. Then I ran the upgrade and replication started working.
4. Go into MySQL and start replication. (See [Slave Configuration Part 2](#slave-configuration-part-2) section)
5. If replication won't start, check the [Replication not starting](#replication-not-starting) section.
5. Monitor replication for a few hours. It may start without issues but fail after a few minutes/hours.
6. Done.

# Copy Data To Slave

There are many options to copy your data in a safe way (that is making a consistent copy that will not change during
the copy process), but I will only comment on two of them I've used extensively.

1. LVM. This is the ideal in any modern Linux-based OS. If you are managing a server where the MySQL partition is using
LVM, then you don't need other tools. You simply snapshot the partition, mount it and copy the data over.
2. If you are currently using LVM, I would recommend using HCP (R1Soft). The end result will be the same: You can
snapshot the partition and then mount the snapshot and copy the data.

Their documentation is not great, but you can find out more about it here:
[R1Soft Hot Copy](#http://wiki.r1soft.com/display/LTR1D/Hot+Copy)

Regardless of the tool you use, just follow these steps **_only in master server_**:

- Create a user for replication:
```bash
mysql> GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%' IDENTIFIED BY 'yourpassword';
mysql> FLUSH PRIVILEGES;
```
:bangbang: This opens the user to any host, modify according to your needs.
- Set MySQL in read-only for a few seconds:
```bash
mysql> FLUSH TABLES WITH READ LOCK;
```
- Check the status of the Master
```bash
mysql> show master status\G
*************************** 1. row ***************************
            File: mysql-bin.84894
        Position: 9545728
    Binlog_Do_DB: 
Binlog_Ignore_DB: 
```
:bangbang: Write down the **_File_** and **_Position_** values.
- Open another ssh session in the server or use SYSTEM command (exiting MySQL to run the copy will remove the read lock)
- Now run the command to make a snapshot of the data.
For example, with hcp command it would be something like this:
```bash
mysql> FLUSH TABLES WITH READ LOCK;
mysql> system hcp -o /dev/sda1
```
:bangbang: (obviously, change this to whatever /var/lib/mysql is mounted on)
This will snapshot the mysql partition and mount it somewhere in /var/idera_hotcopy/*
- Now release the lock on the tables.
```bash
mysql> UNLOCK TABLES;
```
- Finally, copy the data from the mounted snapshot to /var/lib/mysql in the slave. It is important that you copy all
the files in the mysql folder, including the mysql-bin.* files. Replication won't be able to start without these files.
- PS. Remember to clean up all copies/snapshots you create in the Master box when replication is up and running.

# Slave Configuration Part 1

See my-slave.cnf file for an example configuration that worked for me.

These are a few notes on relevant variables to make replication work.

- skip-slave-start
    - I always use this option. By default, when you start MySQL, replication also starts with it. If you set
    skip-slave-start in the config file, it will halt it.
    For example, if you your server failed and rebooted or if you just copied the data from the master, you will
    probably want to halt replication and trigger it manually to check everything is running ok.
- server-id
    - Every member of the replication has to have a unique id. The value itself doesn't matter, but I usually give
    masters a low value and slaves a high value so that I can identify them easily and also build some automation.
- log_bin_index and log_bin
    - When you copy the mysql data from an older version, the binary files will probably be something like mysql-bin.*
    but in newer/different version, it might be different. For example, mariadb config file by default will create
    mariadb-bin.* files. So, when copying from old MySQL version, change the these 2 values so that replication
    uses the files that you just copied.
- binlog_checksum=NONE
    - This option was introduced in MySQL 5.6, so if your master is 5.5.x or older, you will need to disable it.
    This helps with binary logs integrity sent from master to the slaves. It's a wonderful improvement compared
    to the previous system, but even with the latest versions, you can still setup replication without it, so disabling
    it in your special and temporary setup shouldn't cause any problem.
- binlog-format = MIXED
    - I didn't spend a lot of time debugging the reason as to why MIXED needed to be set, but in my case other values
    would not work. Asked in the forums, but since this setup is not supported, I didn't get any traction on this.
- sql_mode = NO_ENGINE_SUBSTITUTION
    - Replication may work initially without this parameter, but will probably break often afterwards. The reason is
    that in the latest versions of the MySQL engine, UPDATE and INSERT statements will go through checks that were not
    present before and they might some times fail. This is known as "STRICT" mode.
    [More info about SQL modes here](https://dev.mysql.com/doc/refman/5.7/en/sql-mode.html)

# Slave Configuration Part 2

Regardless of how you sent the data to the new replica, you will have to set replication from scratch.

If you've copied the data from another slave, you also copied the configuration to connect to the master node.
However, being such different versions, I've found that replication simply won't work in most cases unless you reset it.

So, follow these simple steps to configure replication:

- If you copied the data from another replica, you'll have to fetch 2 values:
**_Relay_Master_Log_File_** and **_Exec_Master_Log_Pos_**.
Simply go into MySQL and run the "show slave status" command.
```bash
mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 10.10.10.1
                  Master_User: replication_user
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.84894
          Read_Master_Log_Pos: 11821764
               Relay_Log_File: mysqld-relay-bin.000040
                Relay_Log_Pos: 5717386
        Relay_Master_Log_File: mysql-bin.84894
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB: 
          Replicate_Ignore_DB: 
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 9545728
              Relay_Log_Space: 11822511
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
                   Using_Gtid: No
                  Gtid_IO_Pos: 
      Replicate_Do_Domain_Ids: 
  Replicate_Ignore_Domain_Ids: 
                Parallel_Mode: conservative
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
```
- From now on, in any case, follow all the following steps:
    - Stop/reset/clear slave configuration:
    ```bash
    mysql> stop slave;
    mysql> reset slave;
    mysql> reset slave all;
    ```
    - Configure slave:
    ```bash
    mysql> change master to master_host='10.10.10.1', master_user='replication_user', master_password='yourpasswordhere', master_log_file='mysql-bin.84894', master_log_pos=9545728, MASTER_USE_GTID=no;
    ```
    :bangbang: Here are the important things you should know about this command:

        - master_host is the master MySQL server.
        - master_user is the user you created for replication.
        - master_password is the password you entered for the replication user.
        - master_log_file is...
            - _Relay_Master_Log_File_ if you copied data from another slave.
            - _File_ if you copied data from master.
        - master_log_pos is...
            - _Exec_Master_Log_Pos_ if you copied data from another slave.
            - _Position_ if you copied data from master.
        - MASTER_USE_GTID=no is mandatory to use when MySQL versions are very different. In 5.6, MySQL introduced GTID
        system for replication. If using an older version as master, you will have to disable this or else it won't connect.
        This is another 'here be dragons' message I usually get. GTID is a much better replication system, there's no discussion
        about it, but even in new installations, you can choose not to use it, so not using it (as it's been the case since the
        beginning of MySQL shouldn't mean you are going to loose all your data).
    - Start slave:
    ```bash
    mysql> START SLAVE;
    ```
    - Finally, check slave is running and catching up with master:
    ```bash
    mysql> show slave status\G
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: 10.10.10.1
                      Master_User: replication_user
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin.84895
              Read_Master_Log_Pos: 11821764
                   Relay_Log_File: mysqld-relay-bin.000041
                    Relay_Log_Pos: 5717386
            Relay_Master_Log_File: mysql-bin.84895
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB: 
              Replicate_Ignore_DB: 
               Replicate_Do_Table: 
           Replicate_Ignore_Table: 
          Replicate_Wild_Do_Table: 
      Replicate_Wild_Ignore_Table: 
                       Last_Errno: 0
                       Last_Error: 
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 54435434
                  Relay_Log_Space: 775453
                  Until_Condition: None
                   Until_Log_File: 
                    Until_Log_Pos: 0
               Master_SSL_Allowed: No
               Master_SSL_CA_File: 
               Master_SSL_CA_Path: 
                  Master_SSL_Cert: 
                Master_SSL_Cipher: 
                   Master_SSL_Key: 
            Seconds_Behind_Master: 34555
    Master_SSL_Verify_Server_Cert: No
                    Last_IO_Errno: 0
                    Last_IO_Error: 
                   Last_SQL_Errno: 0
                   Last_SQL_Error: 
      Replicate_Ignore_Server_Ids: 
                 Master_Server_Id: 1
                   Master_SSL_Crl: 
               Master_SSL_Crlpath: 
                       Using_Gtid: No
                      Gtid_IO_Pos: 
          Replicate_Do_Domain_Ids: 
      Replicate_Ignore_Domain_Ids: 
                    Parallel_Mode: conservative
                        SQL_Delay: 0
              SQL_Remaining_Delay: NULL
          Slave_SQL_Running_State:
    ```
    Pay attention to these values:
        - Slave_IO_Running: It should show Yes.
        - Slave_SQL_Running: It should show Yes.
        - Seconds_Behind_Master: It should show a number >0 that keeps decreasing every time you run the "show slave status"
        command. This is the key to be sure replication is working. It means it managed to connect to master server
        and that is catching up. The number refers to the number of seconds the slave is behind.

# Replication not starting

- If you copied the data from and old replica that is working fine, the new one should work just the same, but before going crazy with this,
make sure you double check the configuration options in [Slave Configuration Part 1](#slave-configuration-part-1) part.
- Also check that the new replica server can reach the old master. Check how you created the replica user and also
any firewall that may prevent you from reaching 3306 on the master server.
- Try to start replication manually.
```bash
mysql> START SLAVE;
```
And then check the warnings and errors you are getting (if any).
```bash
mysql> show warnings;

mysql> show errors;
```
- You may see errors like the following in the log file:

```bash
[ERROR] Missing system table mysql.roles_mapping; please run mysql_upgrade to create it
```
:heavy_exclamation_mark: This simply means you forgot to run mysql_upgrade.

```bash
[ERROR] Slave I/O: Replication event checksum verification failed while reading from network, Internal MariaDB error code: 1743
```
:heavy_exclamation_mark: This simply means you forgot to set the variables binlog_checksum and binlog-format.
