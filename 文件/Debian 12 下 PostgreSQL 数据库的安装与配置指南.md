# Debian 12 下 PostgreSQL 数据库的安装与配置指南

## 更新系统

在安装任何新软件之前，确保安装的软件是最新版本。打开终端并输入以下命令：

```bash
sudo apt update && sudo apt upgrade
```

这个命令将更新软件包列表并安装所有可用的更新。可能需要输入管理员密码以确认操作。

------

## 安装 PostgreSQL

一旦系统更新完成，我们可以开始安装 PostgreSQL。在终端中输入以下命令：

```bash
sudo apt install postgresql postgresql-contrib
```

这个命令将安装 PostgreSQL 数据库服务器以及相关的附加组件。

在安装过程中，可能会看到一些提示信息，询问是否要继续安装。按下 `Y` 键并按回车以确认。安装过程可能需要一些时间，具体取决于您的系统和网络速度。

安装完成后，您将会看到一条消息，表明 PostgreSQL 已经成功安装并已启动。可以通过以下命令验证它的状态：

```bash
sudo systemctl status postgresql
```

这将显示 PostgreSQL 服务的当前状态。如果一切正常，应该看到服务状态为 `active (running)`。

------

### 以 `postgres` 用户身份运行 PostgreSQL

然后，你可以以 `postgres` 用户身份启动 PostgreSQL 命令行工具（`psql`）：

```bash
sudo -u postgres psql
```

如果系统提示你没有权限使用 `sudo`，确保你是 `root` 用户，然后再次执行以上命令。

------

### 修改 `postgres` 用户的密码

一旦你进入 `psql`，你就可以修改 `postgres` 用户的密码：

```sql
ALTER USER postgres WITH PASSWORD '123456';
```

------

### 退出 `psql`

修改完成后，输入 `\q` 退出 PostgreSQL 命令行：

```sql
\q
```

------

## 修改 `pg_hba.conf` 配置文件

### 1. 本地连接（修改 `peer` 为 `md5`）

打开 `pg_hba.conf` 文件，修改以下内容：

```bash
# DO NOT DISABLE!
# If you change this first entry you will need to make sure that the
# database superuser can access the database using some other method.
# Noninteractive access to all databases is required during automatic
# maintenance (custom daily cronjobs, replication, and similar tasks).
#
# Database administrative login by Unix domain socket
local   all             postgres                                md5

# TYPE  DATABASE        USER            ADDRESS                 METHOD

# "local" is for Unix domain socket connections only
local   all             all                                     md5
# IPv4 local connections:
host    all             all             127.0.0.1/32            md5
# IPv6 local connections:
host    all             all             ::1/128                 md5
# Allow replication connections from localhost, by a user with the
# replication privilege.
local   replication     all                                     md5
host    replication     all             127.0.0.1/32            md5
host    replication     all             ::1/128                 md5
# Allow remote connections from any IP address (ensure that listen_addresses is set to '*' in postgresql.conf)
host    all             all             0.0.0.0/0               md5
host    all             all             ::0/0                   md5
```

------

### 2. 额外（配置远程连接）

**远程连接配置**：您需要为所有远程连接添加条目。例如，允许来自任何地方的连接：

```bash
# Allow remote connections from any IP address
host    all             all             0.0.0.0/0               md5
host    all             all             ::0/0                   md5
```

**注意**：`0.0.0.0/0` 表示所有 IPv4 地址，`::0/0` 表示所有 IPv6 地址。

------

### 3. 配置 `postgresql.conf` 文件

确保 `postgresql.conf` 文件中的 `listen_addresses` 设置为 `*`，以允许 PostgreSQL 监听所有的 IP 地址。

```bash
listen_addresses = '*'
```

> **注意**：去掉前面的 `#` 符号，确保配置生效。

------

### 4. 重启 PostgreSQL 服务

修改完配置文件后，重新加载 PostgreSQL 配置或重启 PostgreSQL 服务：

```bash
sudo systemctl restart postgresql
```

或者：

```bash
sudo pg_ctlcluster <version> main restart
```

------

## 连接数据库

现在，您可以通过以下命令尝试从远程机器连接：

```bash
psql -h <remote_ip_address> -U postgres -W
```

- `-h <remote_ip_address>`：指定服务器的 IP 地址。
- `-U postgres`：指定用户。
- `-W`：提示输入密码。

或通过以下命令从本地机器连接并输入密码回车：

```bash
psql -U postgres -W
```

------

## 使用SFTP工具编辑会出现的额外问题

使用 Tabby 自带的或其他工具自带的 SFTP 工具可能会出现配置文件无法访问的情况，建议使用 vim 工具进行编辑。

**错误信息**：

```bash
psql: 错误: 连接到套接字 "/var/run/postgresql/.s.PGSQL.5432" 上的服务器失败: 没有那个文件或目录
        服务器是否在本地运行并接受该套接字上的连接?
```

### 问题根本原因

`pg_hba.conf` 文件的权限或所有者配置错误，导致 PostgreSQL 无法访问该文件。

------

## **解决步骤**

### 1. 检查 `pg_hba.conf` 文件的权限

执行以下命令检查文件权限和所有者：

```bash
ls -l /etc/postgresql/15/main/pg_hba.conf
```

### 2. 修改权限和所有者

如果权限或所有者不正确，执行以下命令修复：

```bash
sudo chown postgres:postgres /etc/postgresql/15/main/pg_hba.conf
sudo chmod 640 /etc/postgresql/15/main/pg_hba.conf
```

------

### 3. 重启 PostgreSQL 服务

修复完成后，重新启动 PostgreSQL 服务：

```bash
sudo systemctl restart postgresql
```