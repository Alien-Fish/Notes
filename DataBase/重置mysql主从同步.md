
重置mysql主从同步（MySQL Reset Master-Slave Replication）

　　在mysql主从同步的过程中，可能会因为各种原因出现主库与从库不同步的情况，网上虽然有一些解决办法，但是有时很难彻底解决，重置主从服务器也许不是最快的办法，但却是最安全有效的。

　　下面将自己重置主从同步的步骤总结一下，以备不时之需。

　　master与slave均使用：centos6.0+mysql 5.1.61 ，假设有db1,db2两个数据库需要热备。

　　文中shell与mysql均使用root账号，在真实环境中，请根据情况更换。 

 

1.停止slave服务器的主从同步

为了防止主从数据不同步，需要先停止slave上的同步服务。

STOP SLAVE;

 

2.对master服务器的数据库加锁

为了避免在备份的时候对数据库进行更新操作，必须对数据库加锁。

FLUSH TABLES WITH READ LOCK;

如果是web服务器也可以关闭apache或nginx服务，效果也是一样的。

 

3.备份master上的数据

mysqldump -u root -p -databases db1 db2 > bak.sql

 

4.重置master服务

RESET MASTER;

这个是重置master的核心语法，看一下官方解释。

RESET MASTER removes all binary log files that are listed in the index file, leaving only a single, empty binary log file with a numeric suffix of .000001, whereas the numbering is not reset by PURGE BINARY LOGS.

RESET MASTER is not intended to be used while any replication slaves are running. The behavior of RESET MASTER when used while slaves are running is undefined (and thus unsupported), whereas PURGE BINARY LOGS may be safely used while replication slaves are running.

大概的意思是RESET MASTER将删除所有的二进制日志，创建一个.000001的空日志。RESET MASTER并不会影响SLAVE服务器上的工作状态，所以盲目的执行这个命令会导致slave找不到master的binlog，造成同步失败。

但是我们就是要重置同步，所以必须执行它。

 

5.对master服务器的数据库解锁

UNLOCK TABLES;

如果你停止了apache或nginx，请开启它们

 

6.将master上的备份文件拷贝到slave服务器上

大可不必用WinScp先下载到本地再上传到slave上，可以直接使用scp命令在服务器间拷贝，速度更快。

scp -r root@XXX.XXX.XXX.XXX:/root/bak.sql ./

 

7.删除slave服务器上的旧数据

删除前，请先确认该备份的是否都备份了。

DROP DATABASE db1;
DROP DATABASE db2;

 

8.导入数据

SOURCE /root/bak.sql;

 

9.重置slave服务

RESET SLAVE;

还是看一下官方解释

RESET SLAVE makes the slave forget its replication position in the master's binary log. This statement is meant to be used for a clean start: It deletes the master.info and relay-log.info files, all the relay log files, and starts a new relay log file. To use RESET SLAVE, the slave replication threads must be stopped (use STOP SLAVE if necessary).

大概意思是，RESET SLAVE将清除slave上的同步位置，删除所有旧的同步日志，使用新的日志重新开始，这正是我们想要的。需要注意的是，必须先停止slave服务（STOP SLAVE），我们已经在第一步停止了它。

 

10.开启slave服务

START SLAVE;

 

大功告成，SHOW SLAVE STATUS\G 检查同步状态，一切正常。
