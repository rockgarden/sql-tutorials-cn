# [如何在 SQL SELECT 语句中执行 IF...THEN 逻辑](https://www.baeldung.com/sql/select-conditional-logic)

如果您在 SQL 生态系统中有几年的经验，并且有兴趣与社区分享这些经验，请查看我们的贡献指南。

1. 概述

    无论是处理简单还是复杂的数据库，我们都可以在 SQL 中使用不同的逻辑条件来增强数据操作能力。例如，我们可以在 SQL SELECT 语句中使用 IF-THEN 逻辑，根据特定条件对数据执行各种任务。

    不过，在 SQL 中没有直接使用 IF-THEN 逻辑的方法，因为 SQL 中没有 IF 关键字。我们可以使用 CASE 语句或 IIF() 函数在 SQL 中实现 IF-THEN 逻辑。

    在本教程中，我们将探讨如何在 SQL Server、MySQL 和 PostgreSQL 等各种方言的 SQL 中实现 IF-THEN 逻辑。

2. 使用 CASE

    CASE 语句充当逻辑 IF-THEN-ELSE 条件语句。我们可以使用它在各种 SQL 数据库（包括 SQL Server、MySQL 和 PostgreSQL）的 SELECT 语句中执行条件分支。

    在 SQL SELECT 中，我们可以使用 WHEN-ELSE 语句代替传统的 IF-ELSE。它评估一个条件，并根据结果返回一个特定值。

    1. 语法

        让我们来看看 CASE 语句的语法：

        ```sql
        SELECT column1, column2,
        CASE
        WHEN condition1 THEN result1
        WHEN condition2 THEN result2
        ELSE result_default
        END AS alias_name
        FROM table_name;
        ```

        在这里，我们使用 SELECT 和 FROM 查询从表中选择多列，并使用 CASE 语句评估条件。

    2. 查询示例

        让我们假设一个例子，在这个例子中，我们希望根据学生的成绩为他们的成绩分配备注。为此，让我们考虑一下大学[数据库](/1-setup/schema/)中的考试表。我们将根据学生的成绩为 student_id 和 course_id 列分配备注，并将其放入新的 grade_remarks 列中。

        让我们为 EXAM 表中的学生成绩分配描述性备注：

        ```sql
        SELECT student_id, course_id,
        CASE
        WHEN grade = 'A+' THEN 'SUPER'
        WHEN grade = 'A' THEN 'EXCELLENT'
        WHEN grade = 'B+' THEN 'GOOD'
        WHEN grade = 'B' THEN 'SATISFACTORY'
        WHEN grade = 'C' THEN 'NEEDS IMPROVEMENT'
        ELSE 'POOR'
        END AS grade_remarks
        FROM EXAM;
        ```

        首先，我们从考试表中选择 student_id 和 course_id 列。然后，我们使用带有 WHEN 和 THEN 的 CASE 来输入基于成绩列的备注。例如，如果成绩列的值为 A+，那么 grade_remarks 列中的备注将为 Super。

3. 使用 IIF()

    我们还可以使用 IIF() 函数来执行 SQL 和 MySQL 中的 IF-THEN 逻辑，只需对语法稍作修改即可。不过，我们不能在 PostgreSQL 中使用该函数，因为它不像 CASE 那样受到普遍支持。

    此外，SQL SELECT 语句中的 IIF() 函数会根据表达式的评估结果返回两个值中的一个。

    1. 语法

        让我们来看看 SQL 中 IIF() 函数的语法：

        ```sql
        SELECT column1,
        IIF(condition, true_result, false_result) AS alias_name
        FROM table_name;
        ```

        在 MySQL 中，我们不能直接使用 IIF() 函数，但可以使用 IF() 函数实现类似功能。让我们看看它的语法：

        ```sql
        SELECT column1,
        IF(condition, true_result, false_result) AS alias_name
        FROM table_name;
        ```

        此外，PostgreSQL 不支持 IIF() 或 IF() 函数。

    2. 查询示例

        让我们重复上一节的示例，但这次我们将使用 IIF() 函数而不是 CASE 表达式。

        让我们使用 IIF() 对 SQL 中的成绩列进行评估：

        ```sql
        SELECT student_id, course_id,
        IIF(grade = 'A+', 'SUPER',
        IIF(grade = 'A', 'EXCELLENT',
        IIF(grade = 'B+', 'GOOD',
        IIF(grade = 'B', 'SATISFACTORY',
        IIF(grade = 'C', 'NEEDS IMPROVEMENT', 'POOR'))))
        ) AS grade_remarks
        FROM EXAM;
        ```

        在上述查询中，我们多次使用嵌套的 IIF() 函数来处理不同的成绩值。此外，让我们使用 IF() 函数代替 MySQL 的 IIF()：

        ```sql
        SELECT student_id, course_id,
        IF(grade = 'A+', 'SUPER',
        IF(grade = 'A', 'EXCELLENT',
        IF(grade = 'B+', 'GOOD',
        IF(grade = 'B', 'SATISFACTORY',
        IF(grade = 'C', 'NEEDS IMPROVEMENT', 'POOR'))))
        ) AS grade_remarks
        FROM EXAM;
        ```

        上述查询使用多个 IF() 函数来评估成绩列并返回相应的备注。

4. 使用 CHOOSE()

    我们还可以在 SQL SELECT 中使用 CHOOSE() 函数来做出决定。它就像条件语句的快捷方式，允许我们根据位置或索引从选项列表中进行选择。此外，我们还可以将 CHOOSE() 与 CASE 结合使用，在 SELECT 查询中创建简单的 IF-THEN 逻辑。

    1. 语法

        让我们看看如何使用 CHOOSE()

        ```sql
        SELECT column1,
        CHOOSE(CASE grade WHEN condition1 THEN 1 ELSE 2 END, 'true_result', 'false_result') AS alias_name
        FROM table_name;
        ```

        在上述查询中，我们使用 CHOOSE() 函数根据位置从列表中选择一个值，而位置则由评估成绩列中值的 CASE 语句决定。

    2. 查询示例

        让我们再次尝试上一节中的示例，但这次我们将使用 CHOOSE() 函数，而不是 IIF() 或 CASE 表达式。不过，为了完整地执行操作，CHOOSE() 函数内部间接需要使用 CASE 表达式。

        让我们在 SQL 中执行 IF-THEN 逻辑：

        ```sql
        SELECT student_id, course_id,
        CHOOSE(
        CASE grade
        WHEN 'A+' THEN 1
        WHEN 'A' THEN 2
        WHEN 'B+' THEN 3
        WHEN 'B' THEN 4
        WHEN 'C' THEN 5
        ELSE 6
        END,
        'SUPER', 'EXCELLENT', 'GOOD', 'SATISFACTORY', 'NEEDS IMPROVEMENT', 'POOR'
        ) AS grade_remarks
        FROM EXAM;
        ```

        在此查询中，我们创建了一个计算的 grade_remarks 列，用于将成绩映射到描述性备注，例如 A+ 变为 SUPER，B+ 变为 GOOD。

        此外，要在 MySQL 中获得相同的结果，我们可以使用 ELT() 函数：

        ```sql
        SELECT student_id, course_id,
        ELT(
        CASE grade
        WHEN 'A+' THEN 1
        WHEN 'A' THEN 2
        WHEN 'B+' THEN 3
        WHEN 'B' THEN 4
        WHEN 'C' THEN 5
        ELSE 6
        END,
        'SUPER', 'EXCELLENT', 'GOOD', 'SATISFACTORY', 'NEEDS IMPROVEMENT', 'POOR'
        ) AS grade_remarks
        FROM EXAM;
        ```

        此外，在 PostgreSQL 中，我们不能使用 CHOOSE 或 ELT() 函数来执行 IF-THEN 逻辑。

5. 结论

    在本文中，我们探讨了在 SQL SELECT 语句中执行 IF-THEN 逻辑的不同选项。这些选项包括使用 CASE、IIF() 和 CHOOSE()。

    此外，如果我们想要一个几乎在所有方言中都能使用的查询，就必须选择 CASE 语句。如果我们需要在 SELECT 中快速实现 IF-THEN 逻辑，也可以使用 IIF()、IF() 或 CHOOSE() 函数。
