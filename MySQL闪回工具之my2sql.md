# MySQL闪回工具之my2sql



my2sql是一个用Go语言编写的MySQL binlog解析工具，能够生成原始SQL、回滚SQL等，并提供DML统计信息。在误删数据时，它能帮助紧急回滚。使用时，binlog格式需为row且binlog_row_image=full。my2sql具备丰富的功能，包括回滚、DML统计等，并且解析速度较快。




my2sql
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

应用案例
误删整张表数据，需要紧急回滚





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

mysql> #数据
mysql> select * from student;
+------+--------+---------+---------------------+---------------------------------+
| id   | number | name    | add_time            | content                         |
+------+--------+---------+---------------------+---------------------------------+
|   12 |     12 | che     | 2020-07-06 19:39:17 | NULL                            |
| 1217 |     12 | hanraan | 2020-07-06 19:39:17 | NULL                            |
| 1218 |     12 | NULL    | 2020-07-06 19:39:17 | NULL                            |
| 1219 |     12 | hanran  | 2020-07-06 19:39:17 | NULL                            |
| 1221 |     13 | hanran  | 2020-07-06 19:40:10 | NULL                            |
| 1222 |     14 | hanran  | 2020-07-06 19:42:17 | NULL                            |
| 1223 |     15 | hanran  | 2020-07-06 21:55:48 | {"author": "liuhan"}            |
| 1224 |     16 | hanran  | 2020-07-06 21:55:48 | {"author": "liuhan"}            |
| 1226 |     17 | hanran  | 2020-07-06 21:55:48 | {"age": 13, "author": "liuhan"} |
| 1227 |     18 | hanran  | 2020-07-06 21:55:48 | {"age": 13, "author": "liuhan"} |
| 1229 |     20 | chenxi  | 2020-07-11 16:20:50 | NULL                            |
| 1231 |     21 | chenxi  | 2020-07-12 10:12:45 | NULL                            |
| 1232 |    134 | asdf    | 2020-07-12 11:08:41 | NULL                            |
| 1233 |     26 | ranran  | 2020-07-15 19:06:03 | NULL                            |
+------+--------+---------+---------------------+---------------------------------+

mysql> #为了方便演示，重新生成一个binlog
mysql> flush logs;

mysql> #删除表数据
mysql> delete from student;
Query OK, 14 rows affected (0.05 sec)

mysql> select * from student;
Empty set (0.00 sec)



**查看binlog文件**



mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+

| File | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
| ---- | -------- | ------------ | ---------------- | ----------------- |
|      |          |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000045 | 837  |      |      |      |
| ---------------- | ---- | ---- | ---- | ---- |
|                  |      |      |      |      |
+------------------+----------+--------------+------------------+-------------------+



根据操作时间解析binlog，生成回滚日志

./my2sql  -user root -password 123456  -port 3306 \
-host 127.0.0.1 -databases testdb  -tables student \
-work-type rollback   -start-file mysql-bin.000045 \
-start-datetime "2020-07-18 11:40:00" --stop-datetime "2020-07-18 12:00:00" \
-output-dir tmpdir/

**查看生成的回滚SQL**



cd tmpdir/
ls
biglong_trx.txt		binlog_status.txt	rollback.45.sql

#查看DML信息
cat binlog_status.txt
binlog            starttime           stoptime            startpos   stoppos    inserts  updates  deletes  database        table
mysql-bin.000045  2020-07-18_11:49:53 2020-07-18_11:49:53 293        806        0        0        14       testdb          student
#上面信息可以看到解析的binlog DML情况

#查看回滚SQL
cat rollback.45.sql
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1233,26,'ranran','2020-07-15 19:06:03',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1232,134,'asdf','2020-07-12 11:08:41',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1231,21,'chenxi','2020-07-12 10:12:45',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1229,20,'chenxi','2020-07-11 16:20:50',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1227,18,'hanran','2020-07-06 21:55:48','{\"age\":13,\"author\":\"liuhan\"}');
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1226,17,'hanran','2020-07-06 21:55:48','{\"age\":13,\"author\":\"liuhan\"}');
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1224,16,'hanran','2020-07-06 21:55:48','{\"author\":\"liuhan\"}');
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1223,15,'hanran','2020-07-06 21:55:48','{\"author\":\"liuhan\"}');
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1222,14,'hanran','2020-07-06 19:42:17',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1221,13,'hanran','2020-07-06 19:40:10',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1219,12,'hanran','2020-07-06 19:39:17',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1218,12,null,'2020-07-06 19:39:17',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (1217,12,'hanraan','2020-07-06 19:39:17',null);
INSERT INTO `testdb`.`student` (`id`,`number`,`name`,`add_time`,`content`) VALUES (12,12,'che','2020-07-06 19:39:17',null);

#应用回滚SQL恢复数据
mysql -u root -p123456 -P3306 -h127.0.0.1  testdb < tmpdir/rollback.45.sql
mysql> select * from student;
+------+--------+---------+---------------------+---------------------------------+

| id   | number | name | add_time | content |
| ---- | ------ | ---- | -------- | ------- |
|      |        |      |          |         |
+------+--------+---------+---------------------+---------------------------------+
| 12   | 12   | che  | 2020-07-06 19:39:17 | NULL |
| ---- | ---- | ---- | ------------------- | ---- |
|      |      |      |                     |      |
| 1217 | 12   | hanraan | 2020-07-06 19:39:17 | NULL |
| ---- | ---- | ------- | ------------------- | ---- |
|      |      |         |                     |      |
| 1218 | 12   | NULL | 2020-07-06 19:39:17 | NULL |
| ---- | ---- | ---- | ------------------- | ---- |
|      |      |      |                     |      |
| 1219 | 12   | hanran | 2020-07-06 19:39:17 | NULL |
| ---- | ---- | ------ | ------------------- | ---- |
|      |      |        |                     |      |
| 1221 | 13   | hanran | 2020-07-06 19:40:10 | NULL |
| ---- | ---- | ------ | ------------------- | ---- |
|      |      |        |                     |      |
| 1222 | 14   | hanran | 2020-07-06 19:42:17 | NULL |
| ---- | ---- | ------ | ------------------- | ---- |
|      |      |        |                     |      |
| 1223 | 15   | hanran | 2020-07-06 21:55:48 | {"author": "liuhan"} |
| ---- | ---- | ------ | ------------------- | -------------------- |
|      |      |        |                     |                      |
| 1224 | 16   | hanran | 2020-07-06 21:55:48 | {"author": "liuhan"} |
| ---- | ---- | ------ | ------------------- | -------------------- |
|      |      |        |                     |                      |
| 1226 | 17   | hanran | 2020-07-06 21:55:48 | {"age": 13, "author": "liuhan"} |
| ---- | ---- | ------ | ------------------- | ------------------------------- |
|      |      |        |                     |                                 |
| 1227 | 18   | hanran | 2020-07-06 21:55:48 | {"age": 13, "author": "liuhan"} |
| ---- | ---- | ------ | ------------------- | ------------------------------- |
|      |      |        |                     |                                 |
| 1229 | 20   | chenxi | 2020-07-11 16:20:50 | NULL |
| ---- | ---- | ------ | ------------------- | ---- |
|      |      |        |                     |      |
| 1231 | 21   | chenxi | 2020-07-12 10:12:45 | NULL |
| ---- | ---- | ------ | ------------------- | ---- |
|      |      |        |                     |      |
| 1232 | 134  | asdf | 2020-07-12 11:08:41 | NULL |
| ---- | ---- | ---- | ------------------- | ---- |
|      |      |      |                     |      |
| 1233 | 26   | ranran | 2020-07-15 19:06:03 | NULL |
| ---- | ---- | ------ | ------------------- | ---- |
|      |      |        |                     |      |
+------+--------+---------+---------------------+---------------------------------+
14 rows in set (0.00 sec)





**根据POS点解析binlog，生成回滚日志**
也可以根据binlog的pos点解析，这里不在展示
执行命令：



./my2sql  -user root -password 123456  -port 3306 \
-host 127.0.0.1 -databases testdb  -tables student \
-work-type rollback   -start-file mysql-bin.000045 \
-start-pos  4 -stop-file  mysql-bin.000045 -stop-pos  837 \
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
————————————————
