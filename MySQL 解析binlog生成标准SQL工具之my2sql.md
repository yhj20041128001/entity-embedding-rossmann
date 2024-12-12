# MySQL 解析binlog生成标准SQL工具之my2sql



my2sql是一款用Golang编写的MySQL binlog解析工具，能够生成原始SQL、回滚SQL等，并提供DML统计信息。支持根据时间或POS点解析binlog，适用于Linux环境。尽管存在如只支持DML回滚等限制，但其丰富的功能和快速的解析速度（1.1G binlog约1分30秒）使其成为高效运维的利器。可在GitHub上找到项目并了解更多信息。



### my2sql

##### 简介

go版MySQL binlog解析工具，通过解析MySQL binlog ，可以生成原始SQL、回滚SQL、去除主键的[INSERT](https://so.csdn.net/so/search?q=INSERT&spm=1001.2101.3001.7020) SQL等，也可以生成DML统计信息。https://github.com/liuhr/my2sql

##### 安装

编译

git clone https://github.com/liuhr/my2sql.git
cd my2sql/
go build .



也可以直接下载Linux版编译好的可执行文件
https://github.com/liuhr/my2sql/blob/master/releases/my2sql

##### 应用案例

**解析mysql binlog生成标准SQL语句**



mysql> #数据库
mysql> show create database testdb\G
*************************** 1. row ***************************
       Database: testdb
Create Database: CREATE DATABASE `testdb` /*!40100 DEFAULT CHARACTER SET utf8mb4 */

mysql> #表结构
mysql> show create table student\G
*************************** 1. row ***************************
       Table: student
Create Table: CREATE TABLE `student` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `number` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `add_time` timestamp NULL DEFAULT CURRENT_TIMESTAMP COMMENT '添加的时间',
  `content` json DEFAULT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `idx_name` (`number`,`name`)
) ENGINE=InnoDB AUTO_INCREMENT=1234 DEFAULT CHARSET=utf8

#查询数据表结构
mysql> select * from student;
Empty set (0.00 sec)

#重新生成要给binlog
mysql> flush logs;
Query OK, 0 rows affected (0.05 sec)

#插入一些数据
mysql> INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1233,26,'ranran','2020-07-15 19:06:03',null);
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1232,134,'asdf','2020-07-12 11:08:41',null);
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1231,21,'chenxi','2020-07-12 10:12:45',null);
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1229,20,'chenxi','2020-07-11 16:20:50',null);
Query OK, 1 row affected (0.01 sec)

mysql> INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1227,18,'hanran','2020-07-06 21:55:48','{\"age\":13,\"author\":\"liuhan\"}');



**查看binlog文件**

mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+

| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| ---- | -------- | ------------ | ---------------- | ----------------- |
|      |          |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000047 | 1661 |      |      |      |
| ---------------- | ---- | ---- | ---- | ---- |
|                  |      |      |      |      |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)



**根据操作时间解析binlog，生成标准SQL**

./my2sql  -user root -password 123456  -port 3306 \
-host 127.0.0.1 -databases testdb  -tables student \
-work-type 2sql   -start-file mysql-bin.000047 \
-start-datetime "2020-07-18 12:35:00" --stop-datetime "2020-07-18 12:43:00" \
-output-dir tmpdir/



**查看生成的标准SQL**

cd tmpdir/
ls
biglong_trx.txt		binlog_status.txt	forward.47.sql

#查看DML信息
cat binlog_status.txt
binlog            starttime           stoptime            startpos   stoppos    inserts  updates  deletes  database        table
mysql-bin.000047  2020-07-18_12:41:19 2020-07-18_12:41:19 301        1630       5        0        0        testdb          student

cat forward.47.sql
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1233,26,'ranran','2020-07-15 19:06:03',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1232,134,'asdf','2020-07-12 11:08:41',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1231,21,'chenxi','2020-07-12 10:12:45',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1229,20,'chenxi','2020-07-11 16:20:50',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1227,18,'hanran','2020-07-06 21:55:48','{\"age\":13,\"author\":\"liuhan\"}');



**根据POS点解析binlog，生成标准SQL**
也可以根据binlog的pos点解析，这里不在展示
执行命令：



./my2sql  -user root -password 123456  -port 3306 \
-host 127.0.0.1 -databases testdb  -tables student \
-work-type rollback   -start-file mysql-bin.000047 \
-start-pos  4 -stop-file  mysql-bin.000047 -stop-pos  1661 \
-output-dir tmpdir/



总结
限制
使用回滚/闪回功能时，binlog格式必须为row,且binlog_row_image=full， DML统计以及大事务分析不受影响
只能回滚DML， 不能回滚DDL
支持指定-tl时区来解释binlog中time/datetime字段的内容。开始时间-start-datetime与结束时间-stop-datetime也会使用此指定的时区， 但注意此开始与结束时间针对的是binlog event header中保存的unix timestamp。结果中的额外的datetime时间信息都是binlog event header中的unix timestamp
此工具是伪装成从库拉取binlog，需要连接数据库的用户有SELECT, REPLICATION SLAVE, REPLICATION CLIENT权限
优点
功能丰富，不仅支持回滚操作，还有其他实用功能。请参考之前博文
基于golang实现，速度快，全量解析1.1Gbinlog只需要1分30秒左右，当前其他类似开源工具一般要几十分钟
github地址 https://github.com/liuhr/my2sql (好使的话别忘记给个星哦😄)


