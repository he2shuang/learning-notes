好的，YugabyteDB 是一个支持两种 API 的分布式 SQL 数据库：

1.  **YSQL**: 与 PostgreSQL 完全兼容的分布式 SQL API。
2.  **YCQL**: 与 Apache Cassandra 查询语言 (CQL) 兼容的半关系型 API。

因此，命令会根据你使用的 API 而有所不同。下面我将分别针对这两种 API 列出你需要的常用命令。通常，如果你在寻找传统的关系型数据库体验，你会使用 **YSQL**。

---

### YSQL (PostgreSQL 兼容 API)

首先，你需要使用 `ysqlsh` 命令行工具连接到数据库，它类似于 PostgreSQL 的 `psql`。

```bash
# 连接到 YugabyteDB
ysqlsh -h <your-host-ip> -p 5433 -U <username>
```

连接成功后，你可以在 `ysqlsh` 交互式环境中使用以下命令。

#### 1. 列出所有数据库

使用元命令 `\l` 或 `\list` 是最快捷的方式。

```sql
\l
```

或者，你也可以使用标准的 SQL 查询：

```sql
SELECT datname FROM pg_database;
```

#### 2. 列出当前数据库中所有的表

首先，确保你已经连接到了目标数据库。如果不在，可以使用 `\c` 命令切换。

```sql
\c my_database_name;
```

然后，使用元命令 `\dt` 来列出所有用户表。

```sql
\dt
```

`\dt` 的含义是 "display tables"。

*   如果想查看更详细的信息（包括系统表），可以使用 `\dt *.*`。
*   如果想查看表的描述、大小等更多信息，可以使用 `\dt+`。

或者，使用标准的 SQL 查询 `information_schema`：

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public' AND table_type = 'BASE TABLE';
```

#### 3. 清空指定表的所有数据但保留表结构

使用 `TRUNCATE` 命令，这是最高效的方式。

```sql
TRUNCATE TABLE your_table_name;
```

**TRUNCATE 的一些重要选项：**

*   `RESTART IDENTITY`: 如果表中有自增序列（比如 `SERIAL` 或 `IDENTITY` 列），这个选项会重置序列计数器。
    ```sql
    TRUNCATE TABLE your_table_name RESTART IDENTITY;
    ```
*   `CASCADE`: 如果有其他表通过外键引用了这个表，使用 `CASCADE` 会自动清空那些引用表。**请谨慎使用！**
    ```sql
    TRUNCATE TABLE your_table_name CASCADE;
    ```

#### 4. 其他 YSQL 常用命令

**数据库/连接管理**
*   `CREATE DATABASE db_name;`  -- 创建新数据库
*   `DROP DATABASE db_name;`    -- 删除数据库
*   `\c db_name;`               -- 切换到另一个数据库
*   `\conninfo;`                -- 显示当前连接信息
*   `\q`                        -- 退出 ysqlsh

**表结构管理**
*   `\d table_name;`            -- 显示表的结构（列、类型、索引、约束等）
*   `\di`                       -- 列出所有索引
*   `\dv`                       -- 列出所有视图
*   `\df`                       -- 列出所有函数

**用户和权限管理**
*   `CREATE ROLE role_name WITH LOGIN PASSWORD 'your_password';` -- 创建用户/角色
*   `GRANT ALL PRIVILEGES ON DATABASE db_name TO role_name;`    -- 授权
*   `GRANT SELECT, INSERT ON table_name TO role_name;`          -- 对表授权
*   `\du`                                                       -- 列出所有用户和角色

**数据操作 (CRUD)**
*   `SELECT * FROM table_name LIMIT 10;`   -- 查询数据（带限制，避免拉取大数据）
*   `INSERT INTO table_name (...) VALUES (...);`
*   `UPDATE table_name SET ... WHERE ...;`
*   `DELETE FROM table_name WHERE ...;`

**备份和恢复 (在 shell 中执行, 而不是 ysqlsh)**
*   `ysql_dump -h <host> -p 5433 -U <user> db_name > backup.sql` -- 备份数据库
*   `ysqlsh -h <host> -p 5433 -U <user> db_name < backup.sql`    -- 恢复数据库

---

### YCQL (Cassandra 兼容 API)

首先，使用 `cqlsh` 命令行工具连接。

```bash
# 连接到 YugabyteDB
cqlsh <your-host-ip> 9042
```

在 YCQL 中，数据库被称为 **Keyspace**。

#### 1. 列出所有 Keyspace (数据库)

```cql
DESCRIBE KEYSPACES;
```
或者简写为 `DESC keyspaces;`

#### 2. 列出当前 Keyspace 中所有的表

首先，你需要进入一个 Keyspace。

```cql
USE my_keyspace_name;
```

然后，列出该 Keyspace 中的所有表。

```cql
DESCRIBE TABLES;
```
或者简写为 `DESC tables;`

#### 3. 清空指定表的所有数据但保留表结构

使用 `TRUNCATE` 命令。

```cql
TRUNCATE TABLE your_table_name;
```
或者
```cql
TRUNCATE your_table_name;
```

#### 4. 其他 YCQL 常用命令

**Keyspace 管理**
*   `CREATE KEYSPACE ks_name WITH REPLICATION = {'class': 'SimpleStrategy', 'replication_factor' : 3};` -- 创建 Keyspace
*   `DROP KEYSPACE ks_name;` -- 删除 Keyspace
*   `USE ks_name;`           -- 切换 Keyspace

**表结构管理**
*   `DESCRIBE TABLE table_name;`  -- 显示表的结构
*   `CREATE TABLE table_name (... PRIMARY KEY(...));` -- 创建表
*   `ALTER TABLE table_name ADD column_name type;`   -- 修改表
*   `DROP TABLE table_name;`      -- 删除表

**数据操作 (CRUD)**
*   `SELECT * FROM table_name WHERE pk_column = ...;`  -- 查询（必须带上主键条件）
*   `INSERT INTO table_name (...) VALUES (...);`
*   `UPDATE table_name SET ... WHERE ...;`
*   `DELETE FROM table_name WHERE ...;`

### 总结

对于刚接触 YugabyteDB 的用户，**YSQL** 是最容易上手的，因为它和 PostgreSQL 的使用体验几乎完全一样。你可以直接使用你熟悉的 PostgreSQL 工具和命令。

**核心要点:**

| 操作 | YSQL (PostgreSQL) | YCQL (Cassandra) |
| :--- | :--- | :--- |
| **连接工具** | `ysqlsh` | `cqlsh` |
| **列出数据库** | `\l` | `DESCRIBE KEYSPACES;` |
| **切换数据库** | `\c db_name` | `USE keyspace_name;` |
| **列出表** | `\dt` | `DESCRIBE TABLES;` |
| **查看表结构** | `\d table_name` | `DESCRIBE TABLE table_name;` |
| **清空表** | `TRUNCATE TABLE table_name;` | `TRUNCATE TABLE table_name;` |

另外，别忘了 YugabyteDB 自带的 **Admin UI** (通常在 `http://<node-ip>:7000`)，它提供了一个图形化界面来查看集群状态、节点、表、正在运行的查询等信息，非常有用。