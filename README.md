# Testing manual switchover on MariaDB using replication-manager

## MariaDB configurations

On the master, you need to assure:

 - log bin
 - address binded to 0.0.0.0

On the slave:

 - relay log
 - ignore system databases (mysql, information_schema..)

## Initial provisioning of the cluster

0. Grant replication permissions on master and obtain the GTID

```
$ docker-compose exec master mysql -uroot -ppass
MariaDB [(none)]> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
Query OK, 0 rows affected (0.01 sec)

MariaDB [(none)]> SHOW MASTER STATUS;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000005 |      493 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.00 sec)

MariaDB [(none)]> SELECT BINLOG_GTID_POS('mariadb-bin.000005', 493);
+--------------------------------------------+
| BINLOG_GTID_POS('mariadb-bin.000005', 493) |
+--------------------------------------------+
| 0-10-7160                                  |
+--------------------------------------------+
1 row in set (0.00 sec)
```

1. Set the replication channel on the slave

```
$ docker-compose exec slave mysql -uroot -ppass
MariaDB [(none)]> SET GLOBAL gtid_slave_pos = '0-10-7160';
MariaDB [(none)]> CHANGE MASTER 'europe' TO MASTER_HOST='master', MASTER_USER='replication', MASTER_PASSWORD='replicationpass', MASTER_USE_GTID=slave_pos;
Query OK, 0 rows affected (0.058 sec)

MariaDB [(none)]> START SLAVE 'europe';
Query OK, 0 rows affected (0.035 sec)

MariaDB [(none)]> GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';    # when we do the switchover and the slave becomes the master, the user needs also grants here
```

2. Do some writing on the master, to allow the current_pos of the slave to be the same than the slave_pos; otherwise the switchover will not work

## Use the replication manager to perform a switchover

```
$ docker-compose exec manager sh
/ # replication-manager-cli switchover --cluster=Default
| Group: Default |  Mode: Manual 
                 Id            Host   Port          Status   Failures   Using GTID         Current GTID           Slave GTID             Replication Health  Delay  RO
db2333073917932517382          master   3306          Master          0           No            0-10-7161                                                          0 OFF
db17127326350201208329           slave   3306           Slave          0    Slave_Pos            0-10-7161            0-10-7161                                     0  ON
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Master switch on slave:3306 complete 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Switching other slaves to the new master 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Doing MariaDB GTID switch of the old master 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Switching old master as a slave 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Inject fake transaction on new master slave:3306  
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Resetting slave on new master and set read/write mode on 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Stopping slave threads on new master 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - Candidate was in sync=false 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - master_log_pos=628 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - master_log_file=mariadb-bin.000005 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - Save replication status before electing 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Reading all relay logs on slave:3306 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Waiting for candidate master to apply relay log 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Rejecting updates on master:3306 (old master) 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Terminating all threads on master:3306 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Slave slave:3306 has been elected as a new master 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Electing a new master 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Flushing tables on master master:3306 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Checking long running updates on master 10 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - -------------------------- 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - Starting master switchover 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] INFO  - -------------------------- 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - Lookup server slave:3306 if maxscale binlog server: master:3306 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - Lookup server master:3306 if maxscale binlog server: master:3306 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - Lookup server slave:3306 if maxscale binlog server: master:3306 
INFO[2019-06-05T12:28:44Z]  2019-06-05 12:28:44 [Default] DEBUG - Lookup server master:3306 if maxscale binlog server: master:3306 
| Group: Default |  Mode: Manual 
                 Id            Host   Port          Status   Failures   Using GTID         Current GTID           Slave GTID             Replication Health  Delay  RO
db2333073917932517382          master   3306           Slave          0           No            0-10-7161                                                          0  ON
db17127326350201208329           slave   3306          Master          0           No            0-11-7236            0-10-7161                                     0 OFF
```
