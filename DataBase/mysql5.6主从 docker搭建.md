#拉取mysql5.6镜像


#启动从节点，挂载配置文件和数据文件
docker run --name master -p 3308:3306 -e MYSQL_ROOT_PASSWORD=root -d -v /opt/docker/mysql/master/conf:/etc/mysql/  -v /opt/docker/mysql/master/data:/var/lib/mysql mysql:5.6

#启动从节点，挂载配置文件和数据文件
docker run --name slave1 -p 3309:3306 -e MYSQL_ROOT_PASSWORD=root -d -v /opt/docker/mysql/slave/conf:/etc/mysql/conf.d  -v /opt/docker/mysql/slave/data:/var/lib/mysql  mysql:5.6

#修改/opt/docker/mysql/主从/conf/my.cnf，在 [mysqld] 节点最后加上后保存，主从的server-id非零唯一
log-bin=mysql-bin
server-id=1

#在主库配置用于连接从节点用户
GRANT REPLICATION SLAVE ON *.* TO 'root'@'%' identified by 'root';
show grants for 'root'@'%';

#在从库配置主库信息
CHANGE MASTER TO
MASTER_HOST='192.168.100.31',
MASTER_PORT=3308,
MASTER_USER='root',
MASTER_PASSWORD='root', 
master_log_file = 'mysqld-bin.000001', 
master_log_pos=3260; 

START SLAVE;

# master_log_file 和 master_log_pos 根据查询主库赋值

#查看从库状态		
show slave status;

#重置配置信息		
stop slave;
reset slave all;
		
set GLOBAL SQL_SLAVE_SKIP_COUNTER=1;
start slave;

#查看日志
docker logs slave1 -f
docker logs master -f

#参考文档
https://blog.csdn.net/bbwangj/article/details/80452571?utm_source=blogxgwz2
