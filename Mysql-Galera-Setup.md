3 Nodes Minimum , with Hardware Specification 
1GHZ single core
512MB RAM
100MBPS network connectivity


PS : 1 node can cause whole cluster to be slow 
1. Confiugre enough swap space on your system.

Disable Firewalld / SE Linux
Configure network connectiivity 
Configure hostname for all 3 nodes
Use hostname in cluster config instead of ips
Use hosts file if you don't have local dns setup.

Hostnames: 
```
mysql-node01 192.168.1.151

mysql-node02 192.168.1.152

mysql-node03 192.168.1.153

Check below one on all 3 nodes
```
# sestatus
```
SELinux status:                 disabled
[root@mysql-master03 ~]# systemctl status firewalld
Unit firewalld.service could not be found.
[root@mysql-master03 ~]#
```

# Install MariaDB
Reference :  https://mariadb.com/docs/server/server-management/install-and-upgrade-mariadb/installing-mariadb/binary-packages/rpm/yum 

Add repo file on all 3 nodes

```
cat /etc/yum.repos.d/MariaDB.repo
[mariadb]
name = MariaDB
baseurl = https://rpm.mariadb.org/10.6/rhel/$releasever/$basearch
gpgkey= https://rpm.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1

```

## Once the repo is added install the package 
Run the below commands individuall on all 3 nodes

``` 
yum install MariaDB-server MariaDB-client MariaDB-common -y 

rpm -qa | grep -i maria

systemctl enable mariadb

systemctl start mariadb 
```

Configure Maria DB : 
on 1st Node (any node) start Maria Db and run the command
```
mysql_secure_installation # Note:  (For old centos 7 and below old Maria DB)

# For latest MariaDB : 
mariadb-secure-installation

```


once the secure installation is done on first node then create galera cluster config:
```

 #cat /etc/my.cnf.d/server.cnf
#
# These groups are read by MariaDB server.
# Use it for options that only the server (but not clients) should see
#
# See the examples of server my.cnf files in /usr/share/mysql/
#

# this is read by the standalone daemon and embedded servers
[server]

# this is only for the mysqld standalone daemon
[mysqld]

#
# * Galera-related settings
#
[galera]
# Mandatory settings
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so
wsrep_cluster_address="gcomm://mysql-master01,mysql-master02,mysql-master03"
binlog_format=row
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
#
# Allow server to accept connections on all interfaces.
#
bind-address=0.0.0.0
wsrep_cluster_name="myown-cluster"
wsrep_sst_method=rsync
#wsrep_node_address= "10.12.0.134"
wsrep_node_name="mysql-master01"
#
# Optional setting
#wsrep_slave_threads=1
#innodb_flush_log_at_trx_commit=0

# this is only for embedded server
[embedded]

# This group is only read by MariaDB servers, not by MySQL.
# If you use the same .cnf file for MySQL and MariaDB,
# you can put MariaDB-only options here
[mariadb]

# This group is only read by MariaDB-10.6 servers.
# If you use the same .cnf file for MariaDB of different versions,
# use this group for options that older servers don't understand
[mariadb-10.6]


```

Make sure you check the library file exist in the system : 
`ls -l /usr/lib64/galera-4/libgalera_smm.so` 

Once the file is generated copy same file on all other servers

```
scp /etc/my.cnf.d/server.cnf mysql-master02:/etc/my.cnf.d/server.cnf
scp /etc/my.cnf.d/server.cnf mysql-master03:/etc/my.cnf.d/server.cnf
```

Make sure to update the node name / hostname in config file copied on other server

Stop the mysql / mariadb service on 1st node and run the below service to start the cluster 

For latest version of rockylinux and MariaDB (systemd and bootstrapping)

```
galera_new_cluster
```

Once the cluster is started you can check the cluster status :
```

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 1     |
+--------------------+-------+
1 row in set (0.001 sec)

```
Restart the mysql/maridb service on other nodes that will get added to cluster.

```

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show status like 'wsrep_cluster_size';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 3     |
+--------------------+-------+
1 row in set (0.001 sec)



```

Checking the replication : 
Create a DB and table on one node and test the replication on other nodes


```

MariaDB [(none)]> create database test100;
Query OK, 1 row affected (0.017 sec)

MariaDB [(none)]> user test100;
ERROR 1064 (42000): You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near 'user test100' at line 1
MariaDB [(none)]> use test100;
Database changed
MariaDB [test100]> CREATE TABLE t1(a INT) ENGINE=InnoDB;
Query OK, 0 rows affected (0.044 sec)

MariaDB [test100]> insert into t1(a) values(50);
Query OK, 1 row affected (0.003 sec)

MariaDB [test100]> insert into t1(a) values(60);
Query OK, 1 row affected (0.005 sec)

MariaDB [test100]> insert into t1(a) values(54);
Query OK, 1 row affected (0.010 sec)

MariaDB [test100]>




```

Testing the cluster Bring down one node and add some records  (shutdown and reboot scenarios)
After that bring down another node and add some records. Also check the node pool status
After additon of the records bring the nodes one by one and check the data status in all nodes.


```
# cat /var/lib/mysql/grastate.dat
# GALERA saved state
version: 2.1
uuid:    e0f2a4cb-92f6-11f0-964b-8682f4cceaec
seqno:   -1
safe_to_bootstrap: 0
[root@mysql-master02 ~]#

```


Add New node to cluster : 

Bring up the new node with ip and all other config need to be copied to new server once that is done restart the mysqld on each node.
#more steps to be added
