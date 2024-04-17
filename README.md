# Postgresql 15逻辑复制实验工程

pg10开始增加了逻辑复制功能，是基于表级别的选择性复制，例如可以复制主库的一部分表到备库，这是一种粒度更细的复制，逻辑复制主要使用场景为：
* 根据业务需求，将一个数据库中的一部分表同步到另一个数据库
* 满足报表库取数需求，从多个数据库采集报表数据
* 实现 PostgreSQL 跨大版本数据同步
* 实现 PostgreSQL 大版本升级

# 设置节点 On Host
1. 启动主从两个节点
```
docker-compose up -d
```
实验发布节点为node1, 订阅节点为node2

2. 修改 data/node1 和 data/node2下的 postgresql.conf，增加以下两行，这是逻辑复制的关键，需要更细粒度的WAL日志
```
docker-compose down

echo "
wal_level = logical
max_replication_slots = 10
" >> data/node1/postgresql.conf

echo "
wal_level = logical
max_replication_slots = 10
" >> data/node2/postgresql.conf

```

3. 修改保存后重新启动两个节点就可以实验了。
```
docker-compose up -d 
```

# 创建复制的发布和订阅通道
## On Node1
创建复制的发布通道（类似消息中间件的Topic）
```
docker exec -it postgres-node1 psql -U postgres  
```
在Node1上执行以下SQL：
```
CREATE PUBLICATION backup_pub;

select pg_create_logical_replication_slot('backup_slot', 'pgoutput');
```

## On Node2
```
docker exec -it postgres-node2 psql -U postgres
```
在Node2上执行以下SQL：
```
CREATE SUBSCRIPTION backup_sub CONNECTION 'host=postgres_master port=5432 user=postgres password=password1 dbname=postgres'
PUBLICATION backup_pub
WITH (copy_data = true, create_slot=false, slot_name=backup_slot);
```
注意host要填slave能访问的网络地址，这里用的是service name（docker-compose规范）

# 发布节点（主库）设置具体要复制的表，订阅节点（从库）启动复制
## On Node1
```
CREATE TABLE test (id INT PRIMARY KEY);

ALTER PUBLICATION backup_pub ADD TABLE test;
```

## On Node2
```
CREATE TABLE test (id INT PRIMARY KEY);

ALTER SUBSCRIPTION backup_sub REFRESH PUBLICATION;
```

# 测试表增删数据，验证复制成功
## On Node1
```
insert into test values (1);
insert into test values (3);
delete from test where id = 1;
```

## On Node2
```
\x
select * from test;

postgres=# select * from test;
 id 
----
  3
(1 row)

select * from pg_stat_replication;

```

## 查看同步状态
```
SELECT * FROM pg_replication_slots;
SELECT * FROM pg_stat_subscription;
SELECT * FROM pg_stat_replication;
```

复制延迟
```
select   pid, client_addr, state, sync_state,  
         pg_wal_lsn_diff(sent_lsn, write_lsn) as write_lag,  
         pg_wal_lsn_diff(sent_lsn, flush_lsn) as flush_lag,  
         pg_wal_lsn_diff(sent_lsn, replay_lsn) as replay_lag
from pg_stat_replication;
```

## 变更或取消同步
```
alter database postgres
set timezone to 'Asia/Shanghai';

drop subscription backup_sub;

CREATE SUBSCRIPTION backup_sub
CONNECTION 'host=postgres_master port=5432 user=postgres password=password dbname=postgres'
PUBLICATION backup_pub
WITH (copy_data = true, create_slot=true, slot_name=backup_slot);

-- ALTER SUBSCRIPTION backup_sub enable;

ALTER PUBLICATION backup_pub DROP TABLE test;

ALTER SUBSCRIPTION backup_sub REFRESH PUBLICATION;
```

## 高级应用


See also:
1. http://www.postgres.cn/docs/14/logical-replication.html
2. https://supabase.com/docs/guides/database/postgres/setup-replication-external
3. https://www.cnblogs.com/jl1771/p/17855927.html
4. https://www.modb.pro/db/428793
5. https://cloud.tencent.com/developer/article/1671149
6. https://xknow.net/logical-replication-in-postgresql/
7. https://github.com/eulerto/wal2json