# [CTE和子查询之间的区别](https://www.baeldung.com/sql/cte-vs-subquery)

1. 介绍

    使用[SQL](https://www.baeldung.com/cs/microservices-db-design#1-sql-vs-nosql)时，结构查询的两种最常见的方法是通用表表达式（Common Table Expressions, [CTEs](https://www.baeldung.com/sql/max-value-per-group#2-using-common-table-expressions)）和[子查询](https://www.baeldung.com/sql/max-value-per-group#1-using-subqueries)。因此，了解何时以及如何使用每个都可以优化查询性能并提高代码的可读性。

    在本教程中，我们将通过实际示例来探索和理解CTE和子查询之间的区别。本教程使用PostgreSQL数据库作为实际用例示例。此外，这适用于MySQL和SQL Server。

    此外，我们将使用[巴伊隆大学模式](https://www.baeldung.com/sql/full-schema)的模式来说明这个过程。

2. 为什么要理解CTE和子查询？

    在处理复杂的查询时，在使用CTE还是子查询之间做出决定可能会令人困惑。根据情况，每种都有其好处，例如代码清晰度、性能和可重复使用性。出于这些原因，我们将讨论这些概念，并查看示例，在后续章节中说明它们的差异。

3. 常见表格表达式（CTE）

    首先，让我们定义、探索语法，并为CTE创建一个用例场景。

    CTE在单个SELECT、INSERT、UPDATE或DELETE语句的执行范围内定义了临时结果集。此外，CTE是使用WITH子句创建的，它使复杂的查询更容易阅读和维护。

    1. CTE的基本语法

        在使用它们之前，让我们充分了解[查询](https://www.baeldung.com/sql/sql-statements-queries#2-sql-queries)[语句](https://www.baeldung.com/sql/sql-statements-queries#1-sql-statements)或子句的基本语法。以下是通用语法：

        ```sql
        WITH Cte_name AS (
            -- SQL query here
            SELECT column1, column2, ...
            FROM table_name
            WHERE condition
        )
        SELECT column1, column2, ...
        FROM Cte_name
        ```

        如上所示，WITH关键字引入了CTE，以及赋予CTE子句的Cte_name名称。因此，名称引用了之后的结果集。此外，在括号中，我们有一个标准的SQL查询，它可以根据需要简单或复杂。

        因此，一旦我们将CTE的名称定义为Cte_name，我们就可以在任何后续的主查询中将其当作表来使用。

    2. 用例

        在这种情况下，让我们使用CTE来查找所有使用大学数据库注册了两门以上课程的学生。对于上下文，我们将在示例中使用学生和注册表：

        - 学生：包含有关学生的信息。
        - 注册：记录每个学生的课程注册。

        以下是这些表格中的信息：

        ```sql
        University=# SELECT * FROM student LIMIT 3;
        id  |    name     | national_id | birth_date | enrollment_date | graduation_date | gpa 
        ------+-------------+-------------+------------+-----------------+-----------------+-----
        1001 | John Liu    |   123345566 | 2001-04-05 | 2020-01-15      | 2024-06-15      |   4
        1003 | Rita Ora    |   132345166 | 2001-01-14 | 2020-01-15      | 2024-06-15      | 4.2
        1007 | Philip Lose |   321345566 | 2001-06-15 | 2020-01-15      | 2024-06-15      | 3.8
        (3 rows)

        University=# SELECT * FROM registration LIMIT 3;
        id | semester | year |    reg_datetime     | course_id | student_id 
        ----+----------+------+---------------------+-----------+------------
        1 | SPRING   | 2022 | 2022-01-11 12:45:56 | CS111     |       1001
        2 | SPRING   | 2022 | 2022-01-11 12:45:56 | CS121     |       1001
        3 | SPRING   | 2022 | 2022-01-11 12:45:56 | CS122     |       1001
        (3 rows)
        ```

        上面，我们看到了我们正在使用的表格中的信息是什么样子的。因此，这让我们更好地了解CTE在用例中的使用。

        让我们使用CTE来输出使用大学模式中的注册和学生表注册超过12门课程的学生列表：

        ```sql
        -- Let's create the CTE here, named course_count
        WITH Course_count AS (
            SELECT student_id, COUNT(course_id) AS total_courses
            FROM Registration
            GROUP BY student_id
        )
        -- select a column from student and CTE (Course_count) table, and filter information
        SELECT s.name AS student_name, cc.total_courses As courses_registered
        FROM Student s
        JOIN Course_count cc ON s.id = cc.student_id
        WHERE cc.total_courses > 12;
        student_name | courses_registered 
        --------------+--------------------
        John Liu     |                 13
        Rose Rit     |                 13
        Roni Roto    |                 13
        (3 rows)
        ```

        如上所示，我们简化了从两个不同表格中过滤特定信息的整个过程。首先，我们创建了一个名为Course_count的CTE表，该表从注册表中存储我们所需的信息。

        然后，其次，我们用学生表加入了Course_count，并选择了我们需要的最后列：student_name和courses_registered。

    3. 类型

        SQL中共有表表达式（CTE）有两种主要类型，即非递归和递归(recursive)CTE。让我们继续更好地了解CTE类型。

        非递归CTE是CTE的标准类型，用于通过将复杂查询分解为更小的部分来简化复杂查询。值得注意的是，这些CTE在自己的定义中并不指代自己。此外，我们之前使用的查询是非递归的CTE。

        递归CTE在其定义中指代自己，允许它们执行递归操作，如遍历分层数据。

        现在，让我们使用递归CTE来找到高级算法课程的所有先决条件：

        ```sql
        WITH RECURSIVE prerequisite_chain AS (
            -- Anchor member: Start with the target course
            SELECT c.id, c.name, p.prerequisite_id
            FROM course c
            JOIN prerequisite p ON c.id = p.course_id
            WHERE c.name = 'Advanced Algorithms'
            
            UNION ALL
            
            -- Recursive member: Find all prerequisites for the courses in the current level
            SELECT c.id, c.name, p.prerequisite_id
            FROM course c
            JOIN prerequisite_chain pc ON c.id = pc.prerequisite_id
            JOIN prerequisite p ON c.id = p.course_id
        )
        SELECT DISTINCT name
        FROM prerequisite_chain;
                    name              
        --------------------------------
        Advanced Algorithms
        Algorithms: Intermediate Level
        (2 rows)
        ```

        让我们分解一下上面的CTE。初始查询通过将先决条件表中的course_id与课程表中的id相匹配来查找高级算法的先决条件。

        此外，CTE的第二个查询通过继续匹配presitry_id来递归地找到上一步中确定的课程的先决条件。随后，查询继续进行，直到找到所有直接和间接先决条件，生成一个完整的先决条件课程列表。

4. 子查询

    子查询嵌套在另一个查询中，括在括号中。总而言之，我们可以在SELECT、INSERT、UPDATE或DELETE语句中使用子查询，数据库每次执行父查询都会评估一次子查询。

    然而，一个重要的说明是，对于一次性使用案例来说，子查询可能更简单，但当嵌套深度时，可读性可能较低。

    1. 基本语法

        让我们简单地理解一下，子查询只是嵌套在另一个SQL查询中的查询。让我们回顾一下它的语法：

        ```sql
        SELECT column1, column2, ...
        FROM TableA
        WHERE column_name operator (
            SELECT column1
            FROM TableB
            WHERE condition
        );
        ```

        如上所示，括号包含子查询。子查询通过提供外部查询需要引用的结果来补充主查询。

        在这种情况下，子查询从TableB获取信息，并使用运算符，如IN、=、ANY、ALL等，将外部查询中的TableA的值与子查询的结果进行比较。

    2. 用例

        让我们使用相同的大学数据库场景来记录学生和注册表。在这里，让我们找到注册结构工程入门课程的学生姓名：

        ```sql
        SELECT name
        FROM student
        WHERE id IN (
            SELECT student_id
            FROM registration
            WHERE course_id = (
                SELECT id
                FROM course
                WHERE name = 'Introduction to Structural Engineering'
            )
        );
        name
        ----------

        Rose Rit
        (1 row)
        ```

        如我们的用例所示，子查询按照它们的嵌套或排列方式逐步运行。在我们的用例场景中，最里面的子查询检索结构工程入门课程的课程ID。

        中间子查询使用从最内侧的子查询中获得的课程ID，从注册表中查找为该课程注册的所有学生ID。

        外部查询从学生表中选择学生姓名，其中student_id与中间子查询返回的其中一个匹配。

    3. 类型

        为了加深对子查询的理解，让我们来探讨一下子查询的类型。子查询有四种类型，即单行子查询、多行子查询、标量子查询和相关子查询(single-row, multi-row, scalar, and correlated subqueries)。

        单行子查询仅返回一行，通常与单值比较运算符（如=、<、>等）配合使用。我们之前的用例查询是单行子查询的典型示例。

        在多行子查询中，子查询返回多行，通常与IN、ANY或ALL运算符一起使用：

        ```sql
        SELECT name
        FROM student
        WHERE id IN (
            SELECT student_id
            FROM registration
            WHERE course_id IN (
                SELECT course_id
                FROM teaching
                WHERE faculty_id = (
                    SELECT id
                    FROM faculty
                    WHERE name = 'Rory Ross'
                )
            )
        );
            name
        --------------

        Rose Rit
        Phellum Luis
        Roni Roto
        Piu Liu
        (4 rows)
        ```

        如上所示，查询返回多行，其实现类似于单行查询，其中执行从最内侧到外侧查询。从本质上讲，此查询列出了目前正在学习Rory Ross教授的课程的所有学生。

        在scalar子查询中，它返回单个值，并在表达式中使用或作为列选择的一部分。让我们来探索一个典型的例子：

        ```sql
        SELECT name, gpa,
            (SELECT AVG(gpa) FROM student) AS avg_gpa
        FROM student
        WHERE gpa > (SELECT AVG(gpa) FROM student);
            name       | gpa  |      avg_gpa
        -----------------+------+-------------------
        John Liu        |    4 | 3.889523835409255
        Rita Ora        |  4.2 | 3.889523835409255
        ...
        Albert Decosta  |    4 | 3.889523835409255
        Roni Roto       | 4.44 | 3.889523835409255
        (12 rows)
        ```

        在这里，子查询计算所有学生的平均GPA，主查询选择GPA高于平均水平的学生的姓名和GPA。

        相关子查询是引用外部查询中的列的子查询，使其依赖于外部查询的结果。

        ```sql
        SELECT name
        FROM student s
        WHERE (
            SELECT COUNT(*)
            FROM registration r
            WHERE r.student_id = s.id
        ) = (
            SELECT MAX(course_count)
            FROM (
                SELECT COUNT(*) AS course_count
                FROM registration
                GROUP BY student_id
            ) AS subquery
        );
        name
        -----------

        John Liu
        Rose Rit
        Roni Roto
        (3 rows)
        ```

        关联子查询用r.student_id = s.id计算每个学生的课程数量，该查询将内部和外部查询联系起来。主要查询选择已注册最多课程数量的学生姓名。

5. 差异

    让我们来看看CTE和子查询之间的关键区别：

    | 特征   | CTE                 | 子查询                                   |
    |------|---------------------|---------------------------------------|
    | 定义   | 使用 WITH 子句定义        | 嵌套在 SELECT、INSERT、UPDATE 或 DELETE 子句中 |
    | 可读性  | 对于复杂查询而言，一般更易读、更易管理 | 深度嵌套时可读性较差                            |
    | 可重用性 | 可在同一查询中多次引用         | 每次执行父查询时只使用一次                         |
    | 性能   | 可由 SQL 引擎进行不同的优化    | 有时会因多次评估而降低效率                         |

6. 结论

    在本文中，我们通过检查CTE和子查询的基本语法、用例和理解它们的差异，探索和深入研究了CTE和子查询之间的差异。CTE提高了具有多个结果集引用的复杂查询的可读性和可重复性。子查询适合更简单的一次性查询。
