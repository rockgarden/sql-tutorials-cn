# 行列互转

在 MySQL 中，行列转换（也称为“透视表”或“转置”）是一种常见的操作，用于将行数据转换为列数据，或将列数据转换为行数据。这种操作通常用于报表生成或数据分析。

## 行转列

行转列 （Row-to-Column）：指的是将数据表中的行数据转换为列数据的操作，通常用于数据透视（Pivot）或重新组织数据结构。

在MySQL中，将行数据转换为列（行转列）通常使用`CASE`表达式结合聚合函数（如`MAX`、`SUM`）和`GROUP BY`子句实现。

- 静态列：直接使用`CASE`表达式明确指定列名。
- 动态列：需通过动态SQL生成，适用于列值不固定的场景。
- 合并字符串：`GROUP_CONCAT`适合非结构化展示。

以下是具体方法及示例：

1. 静态列（已知固定值）

    假设有一张成绩表 `scores`，结构如下：

    | student_id | subject | score |
    |------------|---------|-------|
    | 1          | 数学    | 90    |
    | 1          | 语文    | 85    |
    | 2          | 数学    | 88    |
    | 2          | 语文    | 92    |

    目标：将每个学生的成绩按科目转换为列。

    SQL实现：

    ```sql
    SELECT
        student_id AS 学生ID,
        MAX(CASE WHEN subject = '数学' THEN score END) AS 数学,
        MAX(CASE WHEN subject = '语文' THEN score END) AS 语文
    FROM scores
    GROUP BY student_id;
    ```

    输出结果：

    | 学生ID | 数学 | 语文 |
    |--------|------|------|
    | 1      | 90   | 85   |
    | 2      | 88   | 92   |

    实例：

    ```sql
    SELECT
    fid,
    b.pid,
    MAX(cert_id) AS cert_id,
    MAX(
        CASE
        WHEN field_id = 57 THEN field_value
        END
    ) AS `作业类别`,
    MAX(
        CASE
        WHEN field_id = 58 THEN field_value
        END
    ) AS `颁证单位`,
    MAX(
        CASE
        WHEN field_id = 59 THEN field_value
        END
    ) AS `领证日期`,
    MAX(
        CASE
        WHEN field_id = 60 THEN field_value
        END
    ) AS `换证日期`,
    MAX(
        CASE
        WHEN field_id = 75 THEN field_value
        END
    ) AS `复审日期`,
    MAX(
        CASE
        WHEN field_id = 76 THEN field_value
        END
    ) AS `证件号`
    FROM
    doc_certificate_field_value a
    LEFT JOIN doc_file b ON a.fid = b.id
    where
    cert_id = 37
    GROUP BY
    fid;
    ```

2. 动态列（列名不固定）

    若需要转换的列是动态的（如科目不固定），需通过动态SQL生成语句。MySQL中需借助存储过程实现。

    示例代码：

    ```sql
    -- 创建存储过程
    DELIMITER $$
    CREATE PROCEDURE pivot_subjects()
    BEGIN
        SET @sql = NULL;

        -- 获取所有科目，拼接为CASE语句
        SELECT GROUP_CONCAT(DISTINCT
            CONCAT(
                'MAX(CASE WHEN subject = ''', subject, ''' THEN score END) AS `', subject, '`'
            )
        ) INTO @sql
        FROM scores;

        -- 拼接完整SQL并执行
        SET @sql = CONCAT('SELECT student_id, ', @sql, ' FROM scores GROUP BY student_id');
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END$$
    DELIMITER ;

    -- 调用存储过程
    CALL pivot_subjects();
    ```

    输出结果（假设科目为数学、语文、英语）：

    | student_id | 数学 | 语文 | 英语 |
    |------------|------|------|------|
    | 1          | 90   | 85   | 80   |
    | 2          | 88   | 92   | 75   |

3. 使用`GROUP_CONCAT`简化输出

    若仅需合并多行数据为单列字符串（非严格的行转列），可使用`GROUP_CONCAT`：

    ```sql
    SELECT
        student_id,
        GROUP_CONCAT(CONCAT(subject, ':', score)) AS 成绩
    FROM scores
    GROUP BY student_id;
    ```

    输出结果：

    | student_id | 成绩               |
    |------------|--------------------|
    | 1          | 数学:90, 语文:85  |
    | 2          | 数学:88, 语文:92  |

## 列转行

列转行 （Column-to-Row）：指的是将数据表中的列数据转换为行数据的操作，通常称为 Unpivot （逆透视）操作。这种操作在数据分析或数据重组中非常常见。

在 MySQL 中，没有直接的 `UNPIVOT` 函数（与某些数据库如 SQL Server 不同），但可以通过以下方法实现列转行：

- 在 MySQL 中，可以通过 `UNION ALL` 或动态 SQL 实现列转行。

1. 使用 `UNION ALL`

    假设有一个表 `sales`，包含以下数据：

    ```sql
    +------+-----+-----+-----+
    | year | Jan | Feb | Mar |
    +------+-----+-----+-----+
    | 2023 | 100 | 200 | 150 |
    | 2024 | 300 | 250 | 350 |
    +------+-----+-----+-----+
    ```

    我们希望将月份列（`Jan`, `Feb`, `Mar`）转换为行。

    查询：

    ```sql
    SELECT year, 'Jan' AS month, Jan AS sales FROM sales
    UNION ALL
    SELECT year, 'Feb' AS month, Feb AS sales FROM sales
    UNION ALL
    SELECT year, 'Mar' AS month, Mar AS sales FROM sales;
    ```

    结果：

    ```sql
    +------+-------+-------+
    | year | month | sales |
    +------+-------+-------+
    | 2023 | Jan   | 100   |
    | 2023 | Feb   | 200   |
    | 2023 | Mar   | 150   |
    | 2024 | Jan   | 300   |
    | 2024 | Feb   | 250   |
    | 2024 | Mar   | 350   |
    +------+-------+-------+
    ```

2. 动态生成 SQL（适用于列名未知的情况）

    如果列名是动态或数量较多的情况，可以结合 `INFORMATION_SCHEMA.COLUMNS` 和动态 SQL 来生成查询语句。

    示例：

    ```sql
    SET @sql = NULL;

    -- 动态生成列转行的 SQL
    SELECT GROUP_CONCAT(
        CONCAT('SELECT year, ''', COLUMN_NAME, ''' AS month, ', COLUMN_NAME, ' AS sales FROM sales')
        SEPARATOR ' UNION ALL '
    ) INTO @sql
    FROM INFORMATION_SCHEMA.COLUMNS
    WHERE TABLE_NAME = 'sales' AND COLUMN_NAME NOT IN ('year');

    -- 执行动态 SQL
    PREPARE stmt FROM @sql;
    EXECUTE stmt;
    DEALLOCATE PREPARE stmt;
    ```

    示例-带参存储过程：

    ```sql
    DELIMITER //
    CREATE PROCEDURE pivot_to_unpivot(IN db_name VARCHAR(100), IN table_name VARCHAR(100))
    BEGIN
        DECLARE done INT DEFAULT FALSE;
        DECLARE col_name VARCHAR(100);
        DECLARE col_list TEXT DEFAULT '';
        DECLARE cur CURSOR FOR 
            SELECT column_name 
            FROM information_schema.columns 
            WHERE table_schema = db_name AND table_name = table_name;
        DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;
        
        OPEN cur;
        read_loop: LOOP
            FETCH cur INTO col_name;
            IF done THEN
                LEAVE read_loop;
            END IF;
            SET col_list = CONCAT(col_list, 
                IF(col_list = '', '', ' UNION ALL '),
                'SELECT id, ''', col_name, ''' AS column_name, ', col_name, ' AS value FROM ', table_name);
        END LOOP;
        CLOSE cur;
        
        SET @sql = col_list;
        PREPARE stmt FROM @sql;
        EXECUTE stmt;
        DEALLOCATE PREPARE stmt;
    END //
    DELIMITER ;

    -- 调用存储过程
    CALL pivot_to_unpivot('your_database', 'your_table');
    ```

3. 使用 CROSS JOIN 结合 CASE WHEN

    这种方法更灵活，适合动态处理：

    ```sql
    SELECT 
        t.id,
        c.column_name,
        CASE c.column_name
            WHEN 'column1' THEN t.column1
            WHEN 'column2' THEN t.column2
            WHEN 'column3' THEN t.column3
        END AS value
    FROM 
        your_table t
    CROSS JOIN (
        SELECT 'column1' AS column_name
        UNION ALL SELECT 'column2'
        UNION ALL SELECT 'column3'
    ) c;
    ```

4. MySQL 8.0+ 使用 JSON 函数

    MySQL 8.0及以上版本可以使用JSON函数实现更动态的转换：

    ```sql
    SELECT 
        t.id,
        jt.col_name AS column_name,
        jt.col_value AS value
    FROM 
        your_table t,
        JSON_TABLE(
            CONCAT('{"column1":"', t.column1, '","column2":"', t.column2, '","column3":"', t.column3, '"}'),
            '$' COLUMNS (
                col_name VARCHAR(20) PATH '$[*]',
                col_value VARCHAR(100) PATH '$[*]'
            )
        ) AS jt;
    ```

5. 注意事项

   - 数据类型一致性：确保转换后的值具有一致的数据类型
   - NULL值处理：考虑使用COALESCE或IFNULL处理NULL值
   - 性能考虑：对于大型表，UNION ALL方法可能会影响性能
