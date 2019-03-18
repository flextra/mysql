# mysql
mysql master-slave by docker
--5.6

1. run master Container
docker run -p 4406:3306 --name master -v /my/own/datadir:/var/lib/mysql -v youdockerpath/master.cnf:/etc/mysql/conf.d/master.cnf -e MYSQL_ROOT_PASSWORD=yourpasswd -d mysql:5.6.42

2. run slave Container
docker run -p 4407:3306 --name slave -v /my/own/datadir:/var/lib/mysql -v youdockerpath/slave.cnf:/etc/mysql/conf.d/slave.cnf -e MYSQL_ROOT_PASSWORD=yourpasswd -d mysql:5.6.42

3. into the master container
docker exec -it master bash

4. login the mysql
show master status;
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000004 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

create user slaver;

GRANT ALL PRIVILEGES ON *.* TO slaver@'%' IDENTIFIED BY 'yourpasswd';

5. show all container ip
docker inspect --format='{{.Name}} - {{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aq)
docker inspect -f='{{.Name}} {{.NetworkSettings.IPAddress}} {{.HostConfig.PortBindings}}' $(docker ps -aq)

6. login the slave mysql
docker exec -it slave bash

change master to
master_host='172.17.0.8', master_port=3306,
master_user='slaver',
master_password='yourpasswd',
master_log_file='mysql-bin.000004',
master_log_pos=120;

start slave;
show slave status\G

mysql> show slave status\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 172.17.0.8
                  Master_User: slaver
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
          Read_Master_Log_Pos: 406
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 569
        Relay_Master_Log_File: mysql-bin.000004
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
          Exec_Master_Log_Pos: 406
              Relay_Log_Space: 743
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
                  Master_UUID: 8fa36015-43a9-11e9-9c12-0242ac110008
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: 
                Auto_Position: 0
1 row in set (0.00 sec)

7. install ProxySQL
cat <<EOF | tee /etc/yum.repos.d/proxysql.repo
[proxysql_repo]
name= ProxySQL YUM repository
baseurl=https://repo.proxysql.com/ProxySQL/proxysql-2.0.x/centos/\$releasever
gpgcheck=1
gpgkey=https://repo.proxysql.com/ProxySQL/repo_pub_key
EOF

yum list proxysql
yum install proxysql
systemctl start proxysql
chkconfig proxysql on

8. login ProxySQL 
mysql -uadmin -padmin -h127.0.0.1 -P6032 --prompt='Admin>'
insert into mysql_servers(hostgroup_id,hostname,port,weight,comment) values(1,'172.17.0.8',3306,1,'Write Group');
insert into mysql_servers(hostgroup_id,hostname,port,weight,comment) values(2,'172.17.0.6',3306,1,'Read Group');
Admin>select * from mysql_servers;
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| hostgroup_id | hostname   | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment     |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
| 1            | 172.17.0.8 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              | Write Group |
| 2            | 172.17.0.6 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              | Read Group  |
+--------------+------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+-------------+
2 rows in set (0.00 sec)

