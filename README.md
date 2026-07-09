第 1 篇：MySQL + Hive Metastore（理解元数据）
目标
跑通 Hive Metastore，并理解 Hive 的“表”到底存在哪里。

Hive 表的数据在哪里？
Hive Metastore 存什么？
MySQL 为什么是 Hive 的后端？
CREATE TABLE 背后发生了什么？

Hive 本质：
SQL
 |
 v
Hive Metastore
 |
 +----------------+
 |                |
元数据            数据文件
(MySQL)          (HDFS/S3)

环境架构
                 Beeline
                    |
                    |
              HiveServer2
                    |
                    |
            Hive Metastore
                    |
                    |
                 MySQL

创建项目

bigdata-lab/

├── docker-compose.yml

└── hive/

    ├── hive-site.xml

    └── mysql-connector-j.jar

启动 MySQL
docker compose up -d mysql
docker logs hive-mysql

初始化 Metastore
docker exec -it hive-metastore bash
schematool -dbType mysql -initSchema

查看 MySQL 中生成的元数据表
docker exec -it hive-mysql mysql -uroot -proot
use metastore;

show tables;


启动 HiveServer2
docker compose up -d hiveserver2
docker logs hive-server2


使用 Beeline
docker exec -it hive-server2 bash
beeline
!connect jdbc:hive2://localhost:10000


创建第一张 Hive 表
create database demo;
use demo;

create table user(
    id int,
    name string
);
show tables;

回到 MySQL 看变化
use metastore;

select * from DBS;
select * from TBLS;


这一篇需要理解的核心
元数据
+
数据文件
| 内容   | 存储      |
| ---- | ------- |
| 表名   | MySQL   |
| 字段   | MySQL   |
| 分区   | MySQL   |
| 文件路径 | MySQL   |
| 真实数据 | HDFS/S3 |






