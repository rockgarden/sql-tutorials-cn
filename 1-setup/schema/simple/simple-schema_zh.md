# 简单大学模式  

SQL 基础  

1. 数据库结构  
    本节演示了大学数据库基本版本的数据库图。该模式包括三个表：`Department`（系）、`Student`（学生）和 `Course`（课程）。  

    ![**数据库模式图**](/1-setup/schema/simple/University-DB-ER-model.drawio.png)  
    如图所示，`Course` 表有一个外键引用指向 `Department` 表，而 `Student` 表没有任何外键引用。  

2. 设置并运行查询  
    我们将使用 PostgreSQL、MySQL 和 SQL Server 来运行所有查询。在本节中，我们将介绍如何设置数据库并向其中填充基本数据。  

    1. 手动设置  
        对于此设置，我们首先需要在机器上安装 PostgreSQL、MySQL 和 SQL Server。你可以在这里找到测试过的具体版本和其他安装说明：[手动设置](/1-setup/manual-setup)。  

        安装完成后，我们可以执行创建数据库和表的脚本。[PostgreSQL](./university-postgresql.sql)、[MySQL](./university-mysql.sql) 和 SQL Server 的脚本几乎相同，只有细微的语法差异。创建表后，我们可以从提供的脚本中运行 `INSERT` 查询来填充这些表的数据。  

        此外，安装一个数据库 GUI 客户端（例如 DBVisualizer）会很有帮助。此工具可以连接到所有三种数据库引擎，并高效地运行查询。  

    2. 使用 Docker 进行设置  
        为了避免手动设置数据库，我们可以使用 [Docker](https://www.baeldung.com/ops/docker-guide) 简化这一过程。首先，在机器上安装 Docker。安装完成后，我们可以使用 [Docker Compose](https://www.baeldung.com/ops/docker-compose) 自动完成数据库设置，包括表的创建和数据填充。  

        让我们看看如何使用 Docker 设置数据库的步骤。首先，我们需要克隆包含必要配置的 [`sql-tutorials`](../../docker-setup/) GitHub 仓库。然后执行以下命令进行设置：  

        ```bash
        cd 1-setup/docker-setup
        docker compose up
        ```

        这将启动四个组件的 Docker 实例：每个数据库一个实例，另一个用于名为 Adminer 的简单 GUI 数据库 Web 客户端。  

        所有数据库都会自动配置为大学模式，并填充示例数据。Adminer 客户端可以通过 [http://localhost:8080](http://localhost:8080) 访问。我们可以通过提供 `docker-compose.yml` 文件中的凭据连接到所需的数据库。需要注意的是，当使用 Adminer 时，Linux 上的主机名应为 `localhost`，而在 Mac 和 Windows 上则应为 `host.docker.internal`。此外，我们还可以使用任何数据库客户端（例如 DBVisualizer），将主机设置为 `localhost` 进行连接。  

        如果不想运行所有数据库，我们也可以只运行其中一个。为此，可以导航到所需的数据库目录并运行以下命令：  

        ```bash
        cd 1-setup/docker-setup/postgresql
        docker compose up
        ```

        这只会启动 PostgreSQL 实例，并使用相同的模式。  

    3. 清理  
        我们可以使用以下命令完全清理 Docker 设置：  

        ```bash
        cd 1-setup/docker-setup
        docker compose down
        ```

        此命令会删除整个数据库中的所有数据和模式。重新运行 Docker Compose 将从头开始设置数据库。
