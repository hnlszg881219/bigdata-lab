OLTP 数据
KV 存储

OLAP数据
数据仓库

单条数据的处理逻辑
1. 分层削峰：让"高频小写入"和"批量分析"彻底解耦
   ODS 层
       业务系统产生的原始数据（订单、点击、日志），走 Kafka 这种真正为高频小消息设计的系统承接，不直接怼进 Hive
   DWD/DWS 层
       定时（比如每 15 分钟、每小时、每天）批量把 Kafka 里攒的数据，通过 Spark/Flink 批处理任务，一次性算好、整批写入 Hive/Iceberg
   ADS层
3. Lambda / Kappa 架构：实时和离线分两条腿走路
   Lambda 架构:
           同一份数据，一条线走流处理（Flink 实时算,写入 Redis/HBase/ClickHouse 这种支持高频写的存储,服务实时查询需求）,另一条线走批处理（定时算入 Hive/Iceberg,服务离线分析）。缺点是两套逻辑要维护两遍，容易口径不一致。
   Kappa 架构
           干脆只留流处理这一条线，用 Flink 做实时计算,把结果直接写到能承受高频更新的存储（现在很流行直接写 Iceberg/Hudi，靠它们的增量写入能力吸收高频变更）
4. CDC（Change Data Capture）：把"单条更新"转换成"批量增量"
    生产库（MySQL/PostgreSQL） -> Debezium/Flink CDC 之类工具，捕获数据库的 binlog -> 批量写入Iceberg/Hudi 这类支持高效 upsert 的表格式里
5. 冷热分层：用不同存储承接不同时效性要求
   需要毫秒级单条查询/写入的场景（比如用户实时画像、风控），用 Redis/HBase/MySQL,压根不碰 Hive/Iceberg 这类为批处理优化的存储
   需要复杂聚合分析、跑几十亿行的场景，才落到 Hive/Spark/Iceberg 这类系统
   中间还有 ClickHouse/Doris 这类 OLAP 引擎，专门优化"准实时写入 + 秒级聚合查询"，填补两者之间的空白地带



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
!connect jdbc:hive2://localhost:10000/;auth=noSasl


创建第一张 Hive 表
create database demo;
use demo;

create table users(
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

本地模式有很多坑
1 /tmp/hive 权限（755 → 1777）——Hive 的 scratch 目录必须全局可写
2 Docker 网络名带下划线（bigdata_default → hive-net）——导致 metastore URI 反解析出的 hostname 不合法
3 beeline 客户端没走 auth=noSasl ——服务端配的是 NOSASL 认证模式，客户端连接串也得显式声明
能用hdfs就用它了






