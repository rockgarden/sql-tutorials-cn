# 处理 SQLException

<https://www.geeksforgeeks.org/how-to-handle-sqlexception-in-jdbc/?ref=ml_lbp>

`Error querying database. Cause: org.springframework.jdbc.CannotGetJdbcConnectionException: Failed to obtain JDBC Connection; nested exception is java.sql.SQLException: interrupt`

当 JDBC 在与数据源交互过程中遇到错误时，它会抛出一个 SQLException 实例，而不是 Exception。（在此上下文中，数据源表示连接对象所连接的数据库）。

SQLException 实例包含以下信息，可帮助您确定错误原因：

错误描述。通过调用 SQLException.getMessage 方法获取包含该描述的字符串对象。

SQLState 代码。这些代码及其各自的含义已由 ISO/ANSI 和 Open Group (X/Open) 标准化，但有些代码保留给数据库供应商自行定义。该字符串对象由五个字母数字字符组成。调用 SQLException.getSQLState 方法可获取该代码。

错误代码。这是一个整数值，用于标识导致 SQLException 实例抛出的错误。它的值和含义与具体实现有关，可能是底层数据源返回的实际错误代码。调用 SQLException.getErrorCode 方法可获取该错误。

原因。SQLException 实例可能有一个因果关系，它由一个或多个导致 SQLException 实例抛出的 Throwable 对象组成。要浏览这个原因链，可递归调用方法 SQLException.getCause，直到返回一个空值。

任何链式异常的引用。如果发生一个以上的错误，则通过此异常链引用这些异常。通过对抛出的异常调用 SQLException.getNextException 方法来检索这些异常。

## 检索异常

以下方法（JDBCTutorialUtilities.printSQLException）可输出 SQLException 中包含的 SQLState、错误代码、错误描述和原因（如果有的话），以及与之相关的任何其他异常链：

```java
public static void printSQLException(SQLException ex) {

    for (Throwable e : ex) {
        if (e instanceof SQLException) {
            if (ignoreSQLException(
                ((SQLException)e).
                getSQLState()) == false) {

                e.printStackTrace(System.err)；
                System.err.println("SQLState： “ +
                    ((SQLException)e).getSQLState())；

                System.err.println("Error Code： “ +
                    ((SQLException)e).getErrorCode())；

                System.err.println("Message： “ + e.getMessage())；

                Throwable t = ex.getCause()；
                while(t != null) {
                    System.out.println(“Cause: ” + t)；
                    t = t.getCause()；
                }
            }
        }
    }
}
```

例如，如果以 Java DB 作为 DBMS 调用方法 CoffeesTable.dropTable，表 COFFEES 不存在，并删除对 JDBCTutorialUtilities.ignoreSQLException 的调用，输出将类似于下面的内容：

- SQLState： 42Y55
- 错误代码： 30000
- 信息： 无法对'TESTDB.COFFEES'执行'DROP TABLE' ，因为它不存在。

与其打印 SQLException 信息，不如先检索 SQLState，然后相应地处理 SQLException。例如，如果 SQLState 等于代码 42Y55（并且您使用 Java DB 作为 DBMS），方法 JDBCTutorialUtilities.ignoreSQLException 将返回 true，这将导致 JDBCTutorialUtilities.printSQLException 忽略 SQLException：

```java
public static boolean ignoreSQLException(String sqlState) {

    if (sqlState == null) {
        System.out.println("The SQL state is not defined!");
        return false;
    }

    // X0Y32: Jar file already exists in schema
    if (sqlState.equalsIgnoreCase("X0Y32"))
        return true;

    // 42Y55: Table already exists in schema
    if (sqlState.equalsIgnoreCase("42Y55"))
        return true;

    return false;
}
```

## 检索警告

SQLWarning 对象是 SQLException 的子类，用于处理数据库访问警告。警告不会像异常那样停止应用程序的执行；它们只是提醒用户某些事情没有按计划进行。例如，警告可能会让你知道，你试图撤销的权限没有被撤销。或者，警告可能会告诉你在请求断开连接时发生了错误。

警告可以在 Connection 对象、Statement 对象（包括 PreparedStatement 和 CallableStatement 对象）或 ResultSet 对象上报告。这些类中的每个类都有一个 getWarnings 方法，必须调用该方法才能查看调用对象上报告的第一个警告。如果 getWarnings 返回一个警告，则可以调用 SQLWarning 方法 getNextWarning 来获取其他警告。执行语句会自动清除前一条语句的警告，因此警告不会累积。不过，这意味着如果要检索语句上报告的警告，必须在执行另一条语句之前进行。

JDBCTutorialUtilities.java 中的以下方法说明了如何获取有关在语句或 ResultSet 对象上报告的任何警告的完整信息：

```java
public static void getWarningsFromResultSet(ResultSet rs)
    throws SQLException {
    JDBCTutorialUtilities.printWarnings(rs.getWarnings());
}

public static void getWarningsFromStatement(Statement stmt)
    throws SQLException {
    JDBCTutorialUtilities.printWarnings(stmt.getWarnings());
}

public static void printWarnings(SQLWarning warning)
    throws SQLException {

    if (warning != null) {
        System.out.println("\n---Warning---\n");

    while (warning != null) {
        System.out.println("Message: " + warning.getMessage());
        System.out.println("SQLState: " + warning.getSQLState());
        System.out.print("Vendor error code: ");
        System.out.println(warning.getErrorCode());
        System.out.println("");
        warning = warning.getNextWarning();
    }
}
```

最常见的警告是数据截断警告（DataTruncation warning），它是 SQLWarning 的子类。所有 DataTruncation 对象的 SQLState 都是 01004，表示在读写数据时出现了问题。通过 DataTruncation 方法，你可以知道哪一列或参数的数据被截断，截断是在读取还是写入操作中发生的，本应传输多少字节，以及实际传输了多少字节。

## 分类 SQLException

您的 JDBC 驱动程序可能会抛出一个 SQLException 子类，该子类与常见的 SQLState 或与特定 SQLState 类值无关的常见错误状态相对应。这样，您就可以编写更具可移植性的错误处理代码。这些异常是以下其中一个类的子类：

- SQLNonTransientException
- SQLTransientException
- SQLRecoverableException

有关这些子类的更多信息，请参阅 java.sql 包的最新 Javadoc 或 JDBC 驱动程序的文档。

### SQLException 的其他子类

还可以抛出以下 SQLException 子类：

当批量更新操作过程中发生错误时，会抛出 BatchUpdateException。除了 SQLException 提供的信息外，BatchUpdateException 还提供了错误发生前执行的所有语句的更新计数。
SQLClientInfoException 会在一个或多个客户端信息属性无法在连接上设置时抛出。除了 SQLException 提供的信息外，SQLClientInfoException 还提供了未设置的客户端信息属性列表。
