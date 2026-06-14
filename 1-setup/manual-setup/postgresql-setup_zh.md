# PostgreSQL 设置

本文件包含设置本地 PostgreSQL 数据库的操作说明。

## PostgreSQL 安装

下载并安装 PostgreSQL：<https://www.postgresql.org/download/> 以及 pgAdmin 管理工具：<https://www.pgadmin.org/download/>
（该脚本已在 PostgreSQL v16.2 和 pgAdmin 4 v8.5 版本下测试通过）

打开 pgAdmin 工具并使用您的凭据连接到服务器。然后打开一个新的查询窗口，按以下步骤运行 `university-postgresql.sql` 文件中的脚本：

- 首先，单独运行 `DROP DATABASE` 语句（否则会报错）
- 复制并单独运行 `CREATE DATABASE` 语句
- 复制并运行脚本的其余部分以创建数据表

最后，复制并运行 `populate-university-db` 文件中的脚本，以向数据表中填充数据。

## 常规操作

- MacOS
  - 更新工具：`brew update`
  - 安装或升级到 PG 17 的最新小版本（Homebrew 会自动跟进到 17.7 或更高）：`brew install postgresql@17`
  - 将活动版本切换到此安装包（keg）：`brew link postgresql@17`
  - 启动 postgresql@17 并在登录时自动重启：`brew services start postgresql@17`
- 确认版本：`psql --version`
- 检查系统进程：`ps aux | grep postgres`
- 直接使用当前系统用户身（默认用户）份进入数据库：`psql`
  - `CREATE DATABASE Current_Logged-in_Username;`
- 连接到默认数据库：`psql -d postgres`
  - 精确版本：`SELECT version();`
- 检查是执行并行Worker：`ps aux | grep "parallel worker"`
