# Postgresql 15逻辑复制实验工程

# 设置节点 On Host
1. 启动两个节点
```
docker-compose up -d
```
实验发布节点为node1, 订阅节点为node2

2. 修改 data/node1 和 data/node2下的 postgresql.conf，增加以下两行，这是逻辑复制的关键，需要WAL日志更细粒度
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

# 创建发布和订阅通道
## On Node1
创建复制的发布通道（类似消息中间件的Topic）
```
docker exec -it postgres-node1 psql -U postgres  

CREATE PUBLICATION backup_pub;

select pg_create_logical_replication_slot('backup_slot', 'pgoutput');
```

## On Node2
```
docker exec -it postgres-node2 psql -U postgres

CREATE SUBSCRIPTION backup_sub CONNECTION 'host=postgres_master port=5432 user=postgres password=password1 dbname=postgres'
PUBLICATION backup_pub
WITH (copy_data = true, create_slot=false, slot_name=backup_slot);
```
注意host要填slave能访问的网络地址，这里用的是service name（docker-compose规范）

# 创建发布和订阅通道
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

# Testing
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

See also:
1. https://supabase.com/docs/guides/database/postgres/setup-replication-external
2. https://www.cnblogs.com/jl1771/p/17855927.html
3. https://www.modb.pro/db/428793