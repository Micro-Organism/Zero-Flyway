# Zero-Flyway
Zero-Flyway
# 1. 概述
## 1.1. 简介
我们都知道，Git/ SVN 是代码界的版本控制工具，那么，Flyway 就是一款数据库界的版本控制工具，它可以记录数据库的变化记录。 
可能很多公司都是通过人工去维护、同步数据库脚本，但经常会遇到疏忽而遗漏的情况，举个简单的例子：
> 我们在开发环境对某个表新增了一个字段，而提交测试时却忘了提交该 SQL 脚本，导致出现 bug 而测试中断，从而影响开发、测试的工作效率。

有了 Flyway，我们可以按版本约定，统一管理所有的 SQL 脚本变更，在所有环境自动同步数据库，而无需人为手工控制，再也不用担心因数据库不同步而导致的各种环境问题。

# 2. 功能

# 3. 使用
## 3.1. 安装mysql 5.7
### 3.1.1. mysql5.7.cnf
```yaml
[mysqld]
user=mysql                     # MySQL启动用户
default-storage-engine=INNODB  # 创建新表时将使用的默认存储引擎
character-set-server=utf8mb4      # 设置mysql服务端默认字符集
pid-file        = /var/run/mysqld/mysqld.pid  # pid文件所在目录
socket          = /var/run/mysqld/mysqld.sock # 用于本地连接的socket套接字
datadir         = /var/lib/mysql              # 数据文件存放的目录
#log-error      = /var/log/mysql/error.log
#bind-address   = 127.0.0.1                   # MySQL绑定IP
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION # 定义mysql应该支持的sql语法，数据校验等!

# 允许最大连接数
max_connections=200

# ================= ↓↓↓ mysql主从同步配置start ↓↓↓ =================

# 同一局域网内注意要唯一
server-id=1
# 开启二进制日志功能 & 日志位置存放位置`/var/lib/mysql`
log-bin=mysql-bin
#log-bin=/var/lib/mysql/mysql-bin
# binlog格式
# 1. STATEMENT：基于SQL语句的模式，binlog 数据量小，但是某些语句和函数在复制过程可能导致数据不一致甚至出错；
# 2. MIXED：混合模式，根据语句来选用是 STATEMENT 还是 ROW 模式；
# 3. ROW：基于行的模式，记录的是行的完整变化。安全，但 binlog 会比其他两种模式大很多；
binlog_format=ROW
# FULL：binlog记录每一行的完整变更 MINIMAL：只记录影响后的行
binlog_row_image=FULL
# 日志文件大小
# max_binlog_size=1G
max_binlog_size=100M
# 定义清除过期日志的时间(这里设置为7天)
expire_logs_days=7
binlog-do-db=demo

# ================= ↑↑↑ mysql主从同步配置end ↑↑↑ =================

# 启动MySQL时自动执行此初始化文件
init-file=/etc/mysql/init-file.sql

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4  # 设置mysql客户端默认字符集
```

### 3.1.2. mysql5.7 docker-compose.yml
```yaml
version: '3'
services:
  mysql:
    image: registry.cn-hangzhou.aliyuncs.com/zhengqing/mysql:5.7  # 原镜像`mysql:5.7`
    container_name: mysql_3306                                    # 容器名为'mysql_3306'
    restart: unless-stopped                                       # 指定容器退出后的重启策略为始终重启，但是不考虑在Docker守护进程启动时就已经停止了的容器
    volumes:                                                      # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./mysql/my.cnf:/etc/mysql/my.cnf"
      - "./mysql/init-file.sql:/etc/mysql/init-file.sql"
      - "./mysql/data:/var/lib/mysql"
#      - "./mysql/conf.d:/etc/mysql/conf.d"
      - "./mysql/log/mysql/error.log:/var/log/mysql/error.log"
      - "./mysql/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d" # 可执行初始化sql脚本的目录 -- tips:`/var/lib/mysql`目录下无数据的时候才会执行(即第一次启动的时候才会执行)
    environment:                        # 设置环境变量,相当于docker run命令中的-e
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
      MYSQL_ROOT_PASSWORD: root         # 设置root用户密码
      MYSQL_DATABASE: demo              # 初始化的数据库名称
    ports:                              # 映射端口
      - "3306:3306"
```
### 3.1.3. 启动
```shell script
docker-compose -f docker-compose.yml -p mysql5.7 up -d
```

### 3.1.4. 命令
```shell script
show variables like 'log_%';
```

## 3.2. 安装mysql 8.0
### 3.2.1. mysql8.0.cnf
```yaml
# 服务端参数配置
[mysqld]
user=mysql                            # MySQL启动用户
default-storage-engine=INNODB         # 创建新表时将使用的默认存储引擎
character-set-server=utf8mb4          # 设置mysql服务端默认字符集
collation-server=utf8mb4_general_ci   # 数据库字符集对应一些排序等规则，注意要和character-set-server对应
default-authentication-plugin=mysql_native_password

max_connections=1000    # 允许最大连接数
max_connect_errors=100  # 最大错误连接数

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4  # 设置mysql客户端默认字符集
```
### 3.2.2. mysql8.0 docker-compose.yaml
```yaml
# 可参考： https://hub.docker.com/_/mysql
version: '3'
services:
  mysql:
    image: registry.cn-hangzhou.aliyuncs.com/zhengqing/mysql:8.0  # 原镜像`mysql:8.0`
    container_name: mysql8                                    # 容器名为'mysql8'
    restart: unless-stopped                                               # 指定容器退出后的重启策略为始终重启，但是不考虑在Docker守护进程启动时就已经停止了的容器
    volumes:                                                      # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./mysql/my.cnf:/etc/mysql/my.cnf"
      - "./mysql/data:/var/lib/mysql"
#      - "./mysql/conf.d:/etc/mysql/conf.d"
      - "./mysql/mysql-files:/var/lib/mysql-files"
    environment:                        # 设置环境变量,相当于docker run命令中的-e
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
      MYSQL_ROOT_PASSWORD: root         # 设置root用户密码
      MYSQL_DATABASE: demo              # 初始化的数据库名称
    privileged: true
    user: root
    ports:                              # 映射端口
      - "3308:3306"
```
### 3.2.3. 启动
```shell script
docker-compose -f docker-compose.yml -p mysql8 up -d
```
### 3.2.4. 命令
```shell script
show variables like 'log_%';
```

## 3.3. 安装mysql-master-slave
### 3.3.1. mysql-master.cnf
```yaml
[mysqld]
user=mysql                     # MySQL启动用户
default-storage-engine=INNODB  # 创建新表时将使用的默认存储引擎
character-set-server=utf8mb4      # 设置mysql服务端默认字符集
pid-file        = /var/run/mysqld/mysqld.pid  # pid文件所在目录
socket          = /var/run/mysqld/mysqld.sock # 用于本地连接的socket套接字
datadir         = /var/lib/mysql              # 数据文件存放的目录
#log-error      = /var/log/mysql/error.log
#bind-address   = 127.0.0.1                   # MySQL绑定IP
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION # 定义mysql应该支持的sql语法，数据校验等!

# 允许最大连接数
max_connections=200

# ================= ↓↓↓ mysql主从同步配置start ↓↓↓ =================

# 同一局域网内注意要唯一
server-id=3306
# 开启二进制日志功能 & 日志位置存放位置`/var/lib/mysql`
#log-bin=mysql-bin
log-bin=/var/lib/mysql/mysql-bin
# binlog格式
# 1. STATEMENT：基于SQL语句的模式，binlog 数据量小，但是某些语句和函数在复制过程可能导致数据不一致甚至出错；
# 2. MIXED：混合模式，根据语句来选用是 STATEMENT 还是 ROW 模式；
# 3. ROW：基于行的模式，记录的是行的完整变化。安全，但 binlog 会比其他两种模式大很多；
binlog_format=ROW
# FULL：binlog记录每一行的完整变更 MINIMAL：只记录影响后的行
binlog_row_image=FULL
# 日志文件大小
# max_binlog_size=1G
max_binlog_size=100M
# 定义清除过期日志的时间(这里设置为7天)
expire_logs_days=7

# ================= ↑↑↑ mysql主从同步配置end ↑↑↑ =================

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4  # 设置mysql客户端默认字符集
```

### 3.3.2. mysql-slave.cnf
```yaml
[mysqld]
user=mysql                     # MySQL启动用户
default-storage-engine=INNODB  # 创建新表时将使用的默认存储引擎
character-set-server=utf8mb4      # 设置mysql服务端默认字符集
pid-file        = /var/run/mysqld/mysqld.pid  # pid文件所在目录
socket          = /var/run/mysqld/mysqld.sock # 用于本地连接的socket套接字
datadir         = /var/lib/mysql              # 数据文件存放的目录
#log-error      = /var/log/mysql/error.log
#bind-address   = 127.0.0.1                   # MySQL绑定IP
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION # 定义mysql应该支持的sql语法，数据校验等!

# 允许最大连接数
max_connections=200

# ================= ↓↓↓ mysql主从同步配置start ↓↓↓ =================

# 同一局域网内注意要唯一，从库只需要设置 server_id 即可
server-id=3307

# ================= ↑↑↑ mysql主从同步配置end ↑↑↑ =================

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4  # 设置mysql客户端默认字符集
```

### 3.3.3. docker-compose.yml
```yaml
version: '3'

# 网桥 -> 方便相互通讯
networks:
  mysql:
    driver: bridge

services:
  mysql-master:
    image: registry.cn-hangzhou.aliyuncs.com/zhengqing/mysql:5.7  # 原镜像`mysql:5.7`
    container_name: mysql_master                                  # 容器名为'mysql_master'
    restart: unless-stopped                                       # 指定容器退出后的重启策略为始终重启，但是不考虑在Docker守护进程启动时就已经停止了的容器
    volumes:                                                      # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./mysql/master/my.cnf:/etc/mysql/my.cnf"
      - "./mysql/master/data:/var/lib/mysql"
      #      - "./mysql/master/conf.d:/etc/mysql/conf.d"
      - "./mysql/master/log/mysql/error.log:/var/log/mysql/error.log"
    environment:                        # 设置环境变量,相当于docker run命令中的-e
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
      MYSQL_ROOT_PASSWORD: root         # 设置root用户密码
      MYSQL_DATABASE: demo              # 初始化的数据库名称
    ports:                              # 映射端口
      - "3306:3306"
    networks:
      - mysql

  mysql-slave:
    image: registry.cn-hangzhou.aliyuncs.com/zhengqing/mysql:5.7  # 原镜像`mysql:5.7`
    container_name: mysql_slave                                   # 容器名为'mysql_slave'
    restart: unless-stopped                                       # 指定容器退出后的重启策略为始终重启，但是不考虑在Docker守护进程启动时就已经停止了的容器
    volumes:                                                      # 数据卷挂载路径设置,将本机目录映射到容器目录
      - "./mysql/slave/my.cnf:/etc/mysql/my.cnf"
      - "./mysql/slave/data:/var/lib/mysql"
      #      - "./mysql/slave/conf.d:/etc/mysql/conf.d"
      - "./mysql/slave/log/mysql/error.log:/var/log/mysql/error.log"
    environment:                        # 设置环境变量,相当于docker run命令中的-e
      TZ: Asia/Shanghai
      LANG: en_US.UTF-8
      MYSQL_ROOT_PASSWORD: root         # 设置root用户密码
      MYSQL_DATABASE: demo              # 初始化的数据库名称
    ports:                              # 映射端口
      - "3307:3306"
    depends_on:
      - mysql-master
    networks:
      - mysql
```

### 3.3.4. 启动
```bash
docker-compose -f docker-compose.yml -p mysql-master-slave up -d
```

### 3.3.5. 测试
```shell
# ================== ↓↓↓↓↓↓ 配置主库 ↓↓↓↓↓↓ ==================
# 进入主库
docker exec -it mysql_master /bin/bash
# 登录mysql
mysql -uroot -proot
#  创建用户slave，密码123456
CREATE USER 'slave'@'%' IDENTIFIED BY '123456';
# 授予slave用户 `REPLICATION SLAVE`权限和`REPLICATION CLIENT`权限，用于在`主` `从` 数据库之间同步数据
GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'slave'@'%';
# 授予所有权限则执行命令: GRANT ALL PRIVILEGES ON *.* TO 'slave'@'%';
# 使操作生效
FLUSH PRIVILEGES;
# 查看状态
show master status;
# 注：File和Position字段的值slave中将会用到，在slave操作完成之前不要操作master，否则将会引起状态变化，即File和Position字段的值变化 !!!
# +------------------+----------+--------------+------------------+-------------------+
# | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
# +------------------+----------+--------------+------------------+-------------------+
# | mysql-bin.000003 |      769 |              |                  |                   |
# +------------------+----------+--------------+------------------+-------------------+
# 1 row in set (0.00 sec)


# ================== ↓↓↓↓↓↓ 配置从库 ↓↓↓↓↓↓ ==================
# 进入从库
docker exec -it mysql_slave /bin/bash
# 登录mysql
mysql -uroot -proot
change master to master_host='www.zhengqingya.com',master_port=3306, master_user='slave', master_password='123456', master_log_file='mysql-bin.000003', master_log_pos= 769, master_connect_retry=30;
# 开启主从同步过程  【停止命令：stop slave;】
start slave;
# 查看主从同步状态
show slave status \G
# Slave_IO_Running 和 Slave_SQL_Running 都是Yes的话，就说明主从同步已经配置好了！
# 如果Slave_IO_Running为Connecting，SlaveSQLRunning为Yes，则说明配置有问题，这时候就要检查配置中哪一步出现问题了哦，可根据Last_IO_Error字段信息排错或谷歌…
# *************************** 1. row ***************************
#                Slave_IO_State: Waiting for master to send event
#                   Master_Host: www.zhengqingya.com
#                   Master_User: slave
#                   Master_Port: 3306
#                 Connect_Retry: 30
#               Master_Log_File: mysql-bin.000003
#           Read_Master_Log_Pos: 769
#                Relay_Log_File: c598d8402b43-relay-bin.000002
#                 Relay_Log_Pos: 320
#         Relay_Master_Log_File: mysql-bin.000003
#              Slave_IO_Running: Yes
#             Slave_SQL_Running: Yes
#               Replicate_Do_DB:
```

### 3.3.6. 解决主从同步数据不一致问题
```shell
# 注意：操作的时候停止主库数据写入

# 在从库查看主从同步状态
docker exec -it mysql_slave /bin/bash
mysql -uroot -proot
show slave status \G
#              Slave_IO_Running: Yes
#             Slave_SQL_Running: No

# 1、手动同步主从库数据
# 先在从库停止主从同步
stop slave;
# 导出主库数据
mysqldump -h www.zhengqingya.com -P 3306 -uroot -proot --all-databases > /tmp/all.sql
# 导入到从库
mysql -uroot -proot
source /tmp/all.sql;

# 2、开启主从同步
# 查看主库状态 => 拿到File和Position字段的值
docker exec -it mysql_master /bin/bash
mysql -uroot -proot
show master status;
# 从库操作
change master to master_host='www.zhengqingya.com',master_port=3306, master_user='slave', master_password='123456', master_log_file='mysql-bin.000004', master_log_pos= 488117, master_connect_retry=30;
start slave;
# 查看主从同步状态
show slave status \G
#              Slave_IO_Running: Yes
#             Slave_SQL_Running: Yes
```

# 4. 其他

# 5. 参考