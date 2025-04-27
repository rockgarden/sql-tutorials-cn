# 在 SQL 中合并两行  

SQL 函数

COALESCE

1. 概述  
    在 [SQL](https://www.baeldung.com/cs/microservices-db-design#1-sql-vs-nosql) 中合并两行涉及将两条记录的数据组合成单条记录。这在数据整合和清理中尤为常见。  

    在本文中，我们将使用 [Baeldung University](/1-setup/schema/simple) 数据库中的 `Student` 表来演示合并两行的不同方法。

2. 使用 COALESCE 进行更新  
    一种合并两行的方法是结合 `UPDATE` 语句和 [`COALESCE`](https://dev.mysql.com/doc/refman/8.4/en/comparison-operators.html) 函数。`COALESCE` 函数返回其参数列表中的第一个非空值，因此在需要合并某些字段可能为[空](https://dev.mysql.com/doc/refman/8.4/en/problems-with-null.html)的数据时非常有用。  

    例如，我们考虑合并 Vikas Jain（id=1011）和 Ritu Raj（id=1610）的示例。首先，我们显示要合并的数据：  

    ```sql
    SELECT * FROM Student WHERE id IN (1011, 1610);
    +------+------------+-------------+------------+-----------------+-----------------+--------+
    | id   | name       | national_id | birth_date | enrollment_date | graduation_date | gpa    |
    |------+------------+-------------+------------+-----------------+-----------------+--------|
    | 1011 | Vikas Jain | 321345662   | 2001-07-18 | 2020-01-15      | <null>          | 3.3    |
    | 1610 | Ritu Raj   | 3203455662  | 2002-02-05 | 2021-01-15      | 2025-06-15      | <null> |
    +------+------------+-------------+------------+-----------------+-----------------+--------+
    ```

    查询显示了合并前 Vikas Jain 和 Ritu Raj 的数据。  

    接下来，我们使用 `UPDATE` 语句结合 `COALESCE` 函数来合并这些行：  

    ```sql
    UPDATE Student AS target
    SET name = COALESCE(target.name, source.name),
        national_id = COALESCE(target.national_id, source.national_id),
        birth_date = COALESCE(target.birth_date, source.birth_date),
        enrollment_date = COALESCE(target.enrollment_date, source.enrollment_date),
        graduation_date = COALESCE(target.graduation_date, source.graduation_date),
        gpa = COALESCE(target.gpa, source.gpa)
    FROM Student AS source
    WHERE target.id = 1011 AND source.id = 1610;
    ```

    - [ ] You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near 'FROM Student AS source WHERE target.id = 1011 AND source.id = 1610'

    在此场景中，目标行是要保留的行（id = 1011），源行是要合并到目标行的行（id = 1610）。`COALESCE` 函数确保保留第一个非空值。  

    此外，在合并数据后，我们删除源行以确保没有重复项：  

    ```sql
    DELETE FROM Student WHERE id = 1610;
    ```  

    最后，我们显示数据以验证合并结果：  

    ```sql
    SELECT * FROM Student WHERE id IN (1011);
    +------+------------+-------------+------------+-----------------+-----------------+-----+
    | id   | name       | national_id | birth_date | enrollment_date | graduation_date | gpa |
    |------+------------+-------------+------------+-----------------+-----------------+-----|
    | 1011 | Vikas Jain | 321345662   | 2001-07-18 | 2020-01-15      | 2025-06-15      | 3.3 |
    +------+------------+-------------+------------+-----------------+-----------------+-----+
    ```  

    结果表明数据已成功合并，源行中的相应值替换了目标行中的空值。Vikas Jain 的记录现在包含了 Ritu Raj 的毕业日期。

3. 使用 INSERT INTO 和 DELETE  
    另一种合并两行的方法是通过组合两行的数据创建一个新行，然后删除原始行以确保没有重复项。此外，这种方法确保创建了一个新记录，而无需直接修改现有行。  

    例如，让我们通过组合 Param Mohan（id=1717）和 Siren Lobo（id=1719）的数据来创建一个新的合并行：  

    ```sql
    INSERT INTO Student (id, name, national_id, birth_date, enrollment_date, graduation_date, gpa)
    SELECT 
        1000,
        COALESCE(s1.name, s2.name),
        COALESCE(s1.national_id, s2.national_id),
        COALESCE(s1.birth_date, s2.birth_date),
        COALESCE(s1.enrollment_date, s2.enrollment_date),
        COALESCE(s1.graduation_date, s2.graduation_date),
        COALESCE(s1.gpa, s2.gpa)
    FROM 
        (SELECT * FROM Student WHERE id = 1717) s1,
        (SELECT * FROM Student WHERE id = 1719) s2;
    ```  

    此查询将在 `Student` 表中插入一个新行，组合两行的数据。  

    接下来，我们删除原始行以消除冗余：  

    ```sql
    DELETE FROM Student WHERE id IN (1717, 1719);
    ```  

    最后，我们验证新创建行的合并结果：  

    ```sql
    SELECT * FROM Student WHERE id = 1000;
    +------+-------------+-------------+------------+-----------------+-----------------+------+
    | id   | name        | national_id | birth_date | enrollment_date | graduation_date | gpa  |
    |------+-------------+-------------+------------+-----------------+-----------------+------|
    | 1000 | Param Mohan | 1023456545  | 2002-05-15 | 2021-01-15      | 2025-06-15      | 2.75 |
    +------+-------------+-------------+------------+-----------------+-----------------+------+
    ```  

    我们可以看到数据已成功合并，空值已被替换。

4. 公用表表达式（CTE）  
    使用公用表表达式（CTE）可以简化合并过程。特别是，它为选择和更新行提供了清晰的结构。  

    例如，让我们合并 Potu Singh（id=2017）和 Julia Roberts（id=2008）的行：  

    ```sql
    WITH MergedStudent AS (
        SELECT 
            2017 AS id,
            COALESCE(MAX(name), MIN(name)) AS name,
            COALESCE(MAX(national_id), MIN(national_id)) AS national_id,
            COALESCE(MAX(birth_date), MIN(birth_date)) AS birth_date,
            COALESCE(MAX(enrollment_date), MIN(enrollment_date)) AS enrollment_date,
            COALESCE(MAX(graduation_date), MIN(graduation_date)) AS graduation_date,
            COALESCE(MAX(gpa), MIN(gpa)) AS gpa
        FROM 
            (SELECT * FROM Student WHERE id = 2017
            UNION ALL
            SELECT * FROM Student WHERE id = 2008) s
    )
    SELECT * FROM MergedStudent;
    ```  

    查询显示了 Potu Singh 和 Julia Roberts 的合并数据。  

    首先，查询使用名为 `MergedStudent` 的 CTE 来选择并合并行。在 CTE 内部，它对由 ID 标识的两行执行 `UNION ALL` 操作。然后，它对每个列应用 `COALESCE` 函数，以确保保留非空值。通过使用聚合函数（如 `MAX` 和 `MIN`），查询有效地整合了数据，为每个字段选择了第一个非空值。  

    最后，外部 `SELECT` 语句从 CTE 中检索合并后的行，显示 Potu Singh 和 Julia Roberts 的整合数据。

    恢复数据：

    ```sql
    insert into `Student` (`c0`, `c1`, `c2`, `c3`, `c4`, `c5`, `c6`) values (2008, 'Julia Roberts', '1212446677', '2003-06-12', '2022-01-15', '2025-06-15', 3.04), (2017, 'Potu Singh', '1312445677', '2003-03-11', '2022-01-15', NULL, NULL)
    ```

5. 结论  
    在本文中，我们探讨了在 SQL 中合并两行的各种方法。通过结合使用 `UPDATE` 和 `COALESCE`、`INSERT INTO SELECT` 以及公用表表达式（CTE），我们可以有效地整合数据并确保数据完整性。

    恢复数据：

    ```sql
    CREATE TEMPORARY TABLE Tmp_Student
    (
        id INT PRIMARY KEY NOT null,
        name VARCHAR (60),
        national_id BIGINT NOT Null, 
        birth_date DATE,
        enrollment_date DATE,
        graduation_date DATE,
        gpa FLOAT,
        UNIQUE (id)
    );
    INSERT INTO Tmp_Student (id, name, national_id, birth_date, enrollment_date, graduation_date, gpa) VALUES
        (1011, 'Vikas Jain', '321345662', '2001-07-18', '2020-01-15', NULL, 3.3),
        (1610, 'Ritu Raj', '3203455662', '2002-02-05', '2021-01-15', '2025-06-15', NULL),
        (2008, 'Julia Roberts', '1212446677', '2003-06-12', '2022-01-15', '2025-06-15', 3.04),
        (2017, 'Potu Singh', '1312445677', '2003-03-11', '2022-01-15', NULL, NULL);
    UPDATE Student s
    JOIN Tmp_Student ts ON s.id = ts.id
    SET
        s.name = ts.name, 
        s.national_id = ts.national_id, 
        s.birth_date = ts.birth_date, 
        s.enrollment_date = ts.enrollment_date, 
        s.graduation_date = ts.graduation_date, 
        s.gpa = ts.gpa;
    ```
