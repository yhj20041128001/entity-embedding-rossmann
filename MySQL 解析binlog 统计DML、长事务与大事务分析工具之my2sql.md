# MySQL 解析binlog 统计DML、长事务与大事务分析工具之my2sql

y2sql
简介
go版MySQL binlog解析工具，通过解析MySQL binlog ，可以生成原始SQL、回滚SQL、去除主键的INSERT SQL等，也可以生成DML统计信息。https://github.com/liuhr/my2sql

安装
编译

git clone https://github.com/liuhr/my2sql.git
cd my2sql/
go build .
1
2
3
也可以直接下载Linux版编译好的可执行文件
https://github.com/liuhr/my2sql/blob/master/releases/my2sql



##### 应用案例

**生成DML统计信息，找到热点表，统计大事务信息**
zabbix是常用的监控系统，其底层使用的是mysql做为数据存储，这里以zabbix库为例，查看一段时间DML信息，以及事务信息



./my2sql  -user root -password 123456  -port 3306 \
-databases testdb  -tables student \
-big-trx-row-limit 500 -long-trx-seconds 300 \
-work-type stats   -start-file mysql-bin.000045 \
-start-datetime "2020-07-18 11:40:00" --stop-datetime "2020-07-18 12:00:00" \
-output-dir tmpdir/



**查看生成的分析结果**

cd tmpdir
ls
biglong_trx.txt  binlog_status.txt

#DML统计信息
head -n 10 binlog_status.txt
binlog            starttime           stoptime            startpos   stoppos    inserts  updates  deletes  database        table               
mysql-bin.025924  2020-07-16_13:44:49 2020-07-16_13:45:18 373        30418263   192777   0        0        zabbix          history             
mysql-bin.025924  2020-07-16_13:44:49 2020-07-16_13:45:18 6312       30431731   0        80986    0        zabbix          item_discovery      
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:45:18 378419     30321953   0        122      0        zabbix          hosts               
mysql-bin.025924  2020-07-16_13:44:49 2020-07-16_13:45:18 539        30428268   437159   0        0        zabbix          history_uint        
mysql-bin.025924  2020-07-16_13:44:49 2020-07-16_13:45:18 254109     30431069   69       0        0        zabbix          events              
mysql-bin.025924  2020-07-16_13:44:49 2020-07-16_13:45:18 254563     30431311   45       22       0        zabbix          problem             
mysql-bin.025924  2020-07-16_13:44:49 2020-07-16_13:45:18 255024     30430836   0        62       0        zabbix          item_rtdata         
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:45:18 729854     30003688   22       0        0        zabbix          event_recovery      
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:45:18 763767     30161178   0        20       0        zabbix          triggers       

#事务统计信息
head -n 10 biglong_trx.txt
binlog            starttime           stoptime            startpos   stoppos    rows     duration   tables
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 297896     322782     981      0          [zabbix.history(inserts=206, updates=0, deletes=0) zabbix.history_uint(inserts=775, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 347990     372526     967      0          [zabbix.history(inserts=206, updates=0, deletes=0) zabbix.history_uint(inserts=761, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 379255     403816     968      0          [zabbix.history(inserts=139, updates=0, deletes=0) zabbix.history_uint(inserts=829, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 432562     457923     1000     0          [zabbix.history_uint(inserts=842, updates=0, deletes=0) zabbix.history(inserts=158, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 554079     579440     1000     0          [zabbix.history(inserts=119, updates=0, deletes=0) zabbix.history_uint(inserts=881, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 579505     604851     998      0          [zabbix.history(inserts=334, updates=0, deletes=0) zabbix.history_uint(inserts=664, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 604916     630277     1000     0          [zabbix.history(inserts=269, updates=0, deletes=0) zabbix.history_uint(inserts=731, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 647063     672424     1000     0          [zabbix.history(inserts=184, updates=0, deletes=0) zabbix.history_uint(inserts=816, updates=0, deletes=0)]
mysql-bin.025924  2020-07-16_13:44:50 2020-07-16_13:44:50 672489     697850     1000     0          [zabbix.history_uint(inserts=734, updates=0, deletes=0) zabbix.history(inserts=266, updates=0, deletes=0)]



总结
限制
使用回滚/闪回功能时，binlog格式必须为row,且binlog_row_image=full， DML统计以及大事务分析不受影响
只能回滚DML， 不能回滚DDL
支持指定-tl时区来解释binlog中time/datetime字段的内容。开始时间-start-datetime与结束时间-stop-datetime也会使用此指定的时区， 但注意此开始与结束时间针对的是binlog event header中保存的unix timestamp。结果中的额外的datetime时间信息都是binlog event header中的unix timestamp
此工具是伪装成从库拉取binlog，需要连接数据库的用户有SELECT, REPLICATION SLAVE, REPLICATION CLIENT权限
