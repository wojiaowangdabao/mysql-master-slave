# MySQL主从复制配置

这是一个基于Docker Compose的MySQL主从复制配置项目，使用MySQL 8.0版本，支持GTID复制模式和SSL连接。

## 环境要求

- Docker 19.03+ 
- Docker Compose 1.25+
- Windows/Linux/MacOS

## 配置说明

### 1. 核心配置文件

- `docker-compose.yml` - Docker Compose配置文件，定义了主从节点的服务配置
- `.env` - 环境变量配置文件，存储数据库密码等敏感信息
- `mysql-master.cnf` - 主节点MySQL配置文件
- `mysql-slave1.cnf` - 从节点1 MySQL配置文件
- `mysql-slave2.cnf` - 从节点2 MySQL配置文件

### 2. 端口配置

- **主节点**：3306
- **从节点1**：3307
- **从节点2**：3308

### 3. 资源限制

所有容器的资源限制如下：
- CPU限制：0.5核心
- 内存限制：512MB
- CPU预留：0.25核心
- 内存预留：256MB

### 4. 环境变量

在`.env`文件中配置以下变量：

```env
# 数据库密码配置
MYSQL_ROOT_PASSWORD=root
MYSQL_REPL_USER=repl
MYSQL_REPL_PASSWORD=repl123456
```

## 使用方法

### 1. 启动服务

在项目目录下执行以下命令：

```bash
docker-compose up -d
```

### 2. 验证主从复制

#### 检查主节点状态

```bash
docker exec -it mysql-master mysql -uroot -proot -e "SHOW MASTER STATUS;"
```

#### 检查从节点状态

```bash
docker exec -it mysql-slave1 mysql -uroot -proot -e "SHOW SLAVE STATUS\G;"
docker exec -it mysql-slave2 mysql -uroot -proot -e "SHOW SLAVE STATUS\G;"
```

确认以下两个状态为`Yes`：
- `Slave_IO_Running: Yes`
- `Slave_SQL_Running: Yes`

### 3. 测试数据同步

在主节点创建数据库和表：

```bash
docker exec -it mysql-master mysql -uroot -proot -e "
CREATE DATABASE test_db;
USE test_db;
CREATE TABLE users (id INT AUTO_INCREMENT PRIMARY KEY, name VARCHAR(50), email VARCHAR(50));
INSERT INTO users (name, email) VALUES ('Test User', 'test@example.com');
"
```

在从节点验证数据是否同步：

```bash
docker exec -it mysql-slave1 mysql -uroot -proot -e "SELECT * FROM test_db.users;"
docker exec -it mysql-slave2 mysql -uroot -proot -e "SELECT * FROM test_db.users;"
```

## 核心特性

### 1. GTID复制

使用全局事务标识符（GTID）模式，提高复制的可靠性和可管理性。

### 2. SSL连接

从节点使用SSL连接到主节点，确保数据传输的安全性。

### 3. 只读从节点

从节点配置为只读模式，防止直接写入：

```ini
read_only=1
super_read_only=1
```

### 4. 自动初始化

- 主节点自动创建复制用户
- 从节点自动连接到主节点并开始复制
- 智能等待MySQL启动，避免因启动顺序导致的问题

## 注意事项

1. **首次启动**：首次启动时，主从节点需要初始化数据，可能需要30秒以上时间
2. **密码安全**：生产环境中请修改`.env`文件中的默认密码
3. **数据持久化**：数据存储在Docker卷中，使用`docker-compose down -v`会删除所有数据
4. **SSL配置**：当前配置使用简单的SSL连接，生产环境中建议使用正式的SSL证书

## 故障排除

### 1. 从节点无法连接主节点

检查以下几点：
- 网络连接是否正常
- 主节点是否已创建复制用户
- 复制用户密码是否正确
- SSL配置是否正确

### 2. 复制中断

如果复制中断，可以尝试以下方法：

```bash
# 在从节点执行
docker exec -it mysql-slave1 mysql -uroot -proot -e "
STOP SLAVE;
RESET SLAVE ALL;
CHANGE REPLICATION SOURCE TO
SOURCE_HOST='mysql-master',
SOURCE_USER='repl',
SOURCE_PASSWORD='repl123456',
SOURCE_AUTO_POSITION=1,
SOURCE_SSL=1;
START SLAVE;
"
```

### 3. 查看日志

查看主节点日志：
```bash
docker logs mysql-master
```

查看从节点日志：
```bash
docker logs mysql-slave1
docker logs mysql-slave2
```

## 停止服务

停止服务并保留数据：
```bash
docker-compose down
```

停止服务并删除所有数据：
```bash
docker-compose down -v
```

## 配置修改

如果需要修改配置，可以编辑相应的配置文件，然后重启服务：

```bash
docker-compose restart
```

## 参考文档

- [MySQL 8.0 Replication Documentation](https://dev.mysql.com/doc/refman/8.0/en/replication.html)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
- [MySQL GTID Replication](https://dev.mysql.com/doc/refman/8.0/en/replication-gtids.html)

