---
layout:     post
title:      基于CentOS 7 安装Percona XtraDB Cluster(PXC) 5.7
subtitle:   MySQL集群部署
date:       2018-05-25
author:     haibarai
header-img: img/post-bg-senlinxiaowu.jpg
catalog: true
tags:
    - centos
    - Percona XtraDB
    - mysql
---


##基于CentOS 7 安装Percona XtraDB Cluster(PXC) 5.7（转自Echo_Sxl）
Percona XtraDB Cluster(简称PXC)是很多企业基于MySQL实现集群方案的不二选择。PXC它支持服务高可用,数据同步复制(并发复制)，几乎无延迟；多个可同时读写节点，可实现写扩展等等
本文是基于CentOS 7 PXC 5.7版一个更为标准的安装，可供大家参考。


### 第一步：查看当前os环境
```python
[root@ryxx-dbfs-01 ~]# more /etc/redhat-release
CentOS Linux release 7.5.1804 (Core)
```
### 第二步：配置运行环境
```python
[root@ryxx-dbfs-01 ~]# vim /etc/hosts
127.0.0.1 localhost.localdomain localhost 
192.168.81.142 ryxx-dbfs-01.example.com ryxx-dbfs-01
192.168.81.146 ryxx-dbfs-02.example.com ryxx-dbfs-02
192.168.81.147 ryxx-dbfs-03.example.com ryxx-dbfs-03
```
### 第二步：配置运行环境
####  说明
* `192.168.81.142`，主节点ip地址
* `ryxx-dbfs-01`主机名
* `ryxx-dbfs-01.example.com` more /etc/hosts 默认就有当前主机的域名
* 后面就是第二个节点的ip 域名 主机名，依次添加
### 第三步 修改selinux配置文件
>防止启动节点的时候sst报错 
```python
[root@ryxx-dbfs-01 ~]# vim /etc/selinux/config
SELINUX=disabled
```
###  第四步 分发配置文件到其余节点
```python
[root@ryxx-dbfs-01 ~]# scp /etc/hosts 192.168.81.146:/etc/hosts 
[root@ryxx-dbfs-01 ~]# scp /etc/hosts 192.168.81.147:/etc/hosts 
[root@ryxx-dbfs-01 ~]# scp /etc/selinux/config  192.168.81.146:/etc/hosts 
[root@ryxx-dbfs-01 ~]# scp /etc/selinux/config 192.168.81.147:/etc/hosts 
```
####  说明
* `scp`，发送配置文件到其他节点
* `yes` 可能提示你输入yes
* `节点密码` 会提示你要你输入节点的密码
###  第五步 分别3节点配置防火墙，如下示例
- [x] 注意仅列出一个主节点设置，其他节点记得依次操作
```python
[root@ryxx-dbfs-01 ~]# firewall-cmd --add-port=3306/tcp --permanent
[root@ryxx-dbfs-01 ~]# firewall-cmd --add-port=4567/tcp --permanent
[root@ryxx-dbfs-01 ~]# firewall-cmd --add-port=4568/tcp --permanent
[root@ryxx-dbfs-01 ~]# firewall-cmd --add-port=4444/tcp --permanent
[root@ryxx-dbfs-01 ~]# firewall-cmd --reload
```
###  第五步 配置时间同步服务(可以不操作，看自己需求)
```python
[root@ryxx-dbfs-01 ~]# crontab -e
*/10 * * * * ntpdate ntp3.aliyun.com
```
###  第五步 分别重启3节点
```python
[root@ryxx-dbfs-01 ~]# reboot 
[root@ryxx-dbfs-02 ~]# reboot 
[root@ryxx-dbfs-03 ~]# reboot 
```

###  第六步 安装Percona XtraDB Cluster 5.7
- [x] 注意仅列出一个主节点设置，其他节点记得依次操作
```python
[root@ryxx-dbfs-01 ~]# rpm -Uvh https://www.percona.com/downloads/percona-release/redhat/latest/percona-release-0.1-4.noarch.rpm
 [root@ryxx-dbfs-01 ~]# yum install Percona-XtraDB-Cluster-57 -y

yum安装时会提示UDFs功能，根据需要可以在mysql启动后执行以下语句
Percona XtraDB Cluster is distributed with several useful UDFs from Percona Toolkit.
Run the following commands to create these functions:
mysql -e "CREATE FUNCTION fnv1a_64 RETURNS INTEGER SONAME 'libfnv1a_udf.so'"
mysql -e "CREATE FUNCTION fnv_64 RETURNS INTEGER SONAME 'libfnv_udf.so'"
mysql -e "CREATE FUNCTION murmur_hash RETURNS INTEGER SONAME 'libmurmur_udf.so'"
```
###  第七步 配置mysql及集群配置文件
>主要包括mysqld.cnf，mysqld_safe.cnf，wsrep.cnf 
在当前的这个版本中，my.cnf为主配置文件，其余的配置文件放在/etc/percona-xtradb-cluster.conf.d目录 
在这几个配置文件中，大家根据自己的需要定制，本演示仅作基本修改

- [ ] 1、备份配置文件
```python
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/mysqld.cnf{,.org}
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/mysqld_safe.cnf{,.org}
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/wsrep.cnf{,.org}
```
- [ ] 2、修改配置文件
```python
[root@ryxx-dbfs-01 ~]# vim /etc/percona-xtradb-cluster.conf.d/mysqld.cnf
[root@ryxx-dbfs-01 ~]# more /etc/percona-xtradb-cluster.conf.d/mysqld.cnf
# Template my.cnf for PXC
# Edit to your requirements.
[client]
socket=/var/lib/mysql/mysql.sock

[mysqld]
server-id=142    #这个参数3个节点要使用不同的id
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
log-bin
log_slave_updates
expire_logs_days=7

# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0

log_bin_trust_function_creators=on
character_set_server = utf8  ##Author : SongxiaoLang
collation_server = utf8_bin  ##Blog : http://www.songxiaolang.com

[root@ryxx-dbfs-01 ~]# vim /etc/percona-xtradb-cluster.conf.d/wsrep.cnf
[root@ryxx-dbfs-01 ~]# grep -vE "^#|^$" /etc/percona-xtradb-cluster.conf.d/wsrep.cnf
[mysqld]
wsrep_provider=/usr/lib64/galera3/libgalera_smm.so
wsrep_cluster_address=gcomm://192.168.81.146,192.168.81.147,192.168.81.142 ##这个就是所有节点的ip，建议将主节点ip放在最后一个，可能有未知错误。
binlog_format=ROW
default_storage_engine=InnoDB
wsrep_slave_threads= 8
wsrep_log_conflicts
innodb_autoinc_lock_mode=2
wsrep_node_address=192.168.81.142 ##这个参数要改成相应的IP
wsrep_cluster_name=pxc-cluster   ##集群名字，可以不做修改
wsrep_node_name=ryxx-dbfs-01              ##这个参数要改成相应的节点名称
pxc_strict_mode=PERMISSIVE
wsrep_sst_method=xtrabackup-v2  ##sst连接方式，次节点启动报错建议修改为rsync
wsrep_sst_auth=sstuser:s3cretPass  ##同步数据用户 密码记得不要是是弱口令
```

- [ ] 3、修改剩余节点配置文件

```python
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/mysqld.cnf{,.146}
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/mysqld.cnf{,.147}
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/wsrep.cnf{,.146}
[root@ryxx-dbfs-01 ~]# cp /etc/percona-xtradb-cluster.conf.d/wsrep.cnf{,.147}
[root@ryxx-dbfs-01 ~]# vim /etc/percona-xtradb-cluster.conf.d/mysqld.cnf.146  #修改对应的server_id
[root@ryxx-dbfs-01 ~]# vim/etc/percona-xtradb-cluster.conf.d/mysqld.cnf.147  #修改对应的server_id

以下操作，分别修改wsrep_node_name，wsrep_node_address至相应的节点名称以及IP地址
[root@ryxx-dbfs-01 ~]# vim /etc/percona-xtradb-cluster.conf.d/wsrep.cnf.146 
[root@ryxx-dbfs-01 ~]# vim  /etc/percona-xtradb-cluster.conf.d/wsrep.cnf.147
```
- [ ] 4、修改剩余节点配置文件，并且采用scp 发送，具体提示跟第四步scp 一样
```python
[root@ryxx-dbfs-01 ~]# cd /etc/percona-xtradb-cluster.conf.d/
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# diff mysqld.cnf mysqld.cnf.146
7c7
< server-id=142
---
> server-id=146
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# diff mysqld.cnf mysqld.cnf.147
7c7
< server-id=142
---
> server-id=147
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# diff wsrep.cnf wsrep.cnf.146
28c28
< wsrep_node_address=192.168.81.142
---
> wsrep_node_address=192.168.81.146
32c32
< wsrep_node_name=node142
---
> wsrep_node_name=node146
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# diff wsrep.cnf wsrep.cnf.147
28c28
< wsrep_node_address=192.168.81.142
---
> wsrep_node_address=192.168.81.147
32c32
< wsrep_node_name=node142
---
> wsrep_node_name=node147

[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# scp mysqld.cnf.146 node146:/etc/percona-xtradb-cluster.conf.d/mysqld.cnf
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# scp wsrep.cnf.146 node146:/etc/percona-xtradb-cluster.conf.d/wsrep.cnf
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# scp mysqld.cnf.147 node147:/etc/percona-xtradb-cluster.conf.d/mysqld.cnf
[root@ryxx-dbfs-01 percona-xtradb-cluster.conf.d]# scp wsrep.cnf.147 node147:/etc/percona-xtradb-cluster.conf.d/wsrep.cnf
```

###  第七步 启动PXC集群
- [ ] 1、启动第一个节点
```python
[root@ryxx-dbfs-01 ~]# systemctl start mysql@bootstrap.service
[root@ryxx-dbfs-01 ~]# grep "temporary password" /var/log/mysqld.log
2017-12-28T08:57:24.231185Z 1 [Note] A temporary password is generated for root@localhost: wj!v<z/2)ctZ

[root@ryxx-dbfs-01 ~]# mysql -uroot -p
Enter password:
mysql> alter user 'root'@'localhost' identified by '123456';
mysql> create user 'sstuser'@'localhost' identified by 's3cretpass';
mysql> grant reload, lock tables, replication client, process on *.* to 'sstuser'@'localhost';
```
- [ ] 2、启动剩余节点
```python
[root@node146 ~]# systemctl start mysql
[root@node147 ~]# systemctl start mysql  
注，如果你使用的是CentOS7.2.1511，会发现mysql可以正常启动，但是未加入集群的情况
需要升级openssl，建议全部升级后再启动集群，这问题在CentOS 7.4.1708不存在即openssl版本较新
```
> * 如果报错，请直接看最后解决办法

###  第八步 集群验证
```python
在146上完成如下操作
[root@ryxx-dbfs-02 ~]# mysql -uroot -p
mysql> show variables like 'version';
+---------------+--------------+
| Variable_name | Value |
+---------------+--------------+
| version | 5.7.19-17-57 |
+---------------+--------------+
1 row in set (0.00 sec)

mysql> create database pxcdb;
mysql> use pxcdb;
mysql> create table t1(id tinyint,ename varchar(20));
mysql> insert into t1 values(1,'SongxiaoLang');

在147上进行验证

[root@ryxx-dbfs-02 ~]# mysql -uroot -p
mysql> show databases;
+--------------------+
| Database       |
+--------------------+
| information_schema |
| mysql |
| performance_schema |
| pxcdb |
| sys |
+--------------------+
5 rows in set (0.00 sec)

mysql> show variables like 'server_id';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| server_id    | 147 |
+---------------+-------+
1 row in set (0.01 sec)

mysql> select * from pxcdb.t1;
+------+---------+
| id | ename |
+------+---------+
| 1 | Leshami |
+------+---------+
1 row in set (0.00 sec)

--查看集群状态
mysql> show status like '%wsrep_clust%';
+--------------------------+--------------------------------------+
| Variable_name | Value |
+--------------------------+--------------------------------------+
| wsrep_cluster_conf_id | 13 |  ## Author : SongxiaoLang
| wsrep_cluster_size | 3 |         ## Blog : http://www.songxiaolang.com
| wsrep_cluster_state_uuid | aeb87793-ebb2-11e7-b33e-eeaf4988bbe4 |
| wsrep_cluster_status | Primary |   
+--------------------------+--------------------------------------+
4 rows in set (0.00 sec)

mysql> show status like 'wsrep_connected';
+-----------------+-------+
| Variable_name | Value |
+-----------------+-------+
| wsrep_connected | ON |
+-----------------+-------+
1 row in set (0.00 sec)
```


## 错误信息归类
> 查看错误方法  grep "ERROR" /var/log/mysqld.log
                systemctl start mysql@bootstrap.service
                journalctl -xe
                这三个命令
- [ ] 1.openssl版本过低导致的错误
```python
017-12-28T09:23:19.605353Z 0 [ERROR] WSREP: wsrep_load(): dlopen(): /usr/lib64/galera3/libgalera_smm.so: 
symbol SSL_COMP_free_compression_methods, version libssl.so.10 not defined in file libssl.so.10 with link time reference
2017-12-28T09:23:19.605379Z 0 [ERROR] WSREP: Failed to load wsrep_provider (/usr/lib64/galera3/libgalera_smm.so).
 Error: Invalid argument (code: 22). Reverting to no provider.
2017-12-28T09:23:19.605386Z 0 [Note] WSREP: Setting wsrep_ready to false

[root@ryxx-dbfs-02 ~]# rpm -qa|grep openssl
openssl-1.0.1e-42.el7.9.x86_64
openssl-libs-1.0.1e-42.el7.9.x86_64
[root@ryxx-dbfs-02 ~]# yum update openssl -y
```
- [ ] 2.启动主节点报错
```python
通过三个查看错误的方法说当mysql的错误日志文件不存在的时候，会产生这个无效用户的错误
解决方法
[root@ryxx-dbfs-01]# touch /var/log/mysqld.log
[root@ryxx-dbfs-01]# chown mysql:mysql /var/log/mysqld.log 
通过三个查看错误的方法说当mysql已经存在的时候，会产生这个无效用户的错误
[root@ryxx-dbfs-01]# cd /var/lib/
[root@ryxx-dbfs-01]# mv mysql mysql.bak
```
- [ ] 3.第二个节点，点三个节点报错
```python
sst 与xtrabackup-v2 错误
[root@ryxx-dbfs-02]vim /etc/percona-xtradb-cluster.conf.d/wsrep.cnf
不妨先将wsrep_sst_method的方式设置为rsync或者mysqldump，看能否成功
```
