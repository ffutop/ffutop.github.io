---
title: Spring JDBC 源码学习
layout: post
author: fangfeng
categories:
  - 技术
date: 2018-01-15
tags:
  - Spring
  - JDBC
typora-copy-images-to: ipic
---

## 概览

在学习 Spring-JDBC 之前，我们有必要从 Java 原生提供的 JDBC 开始，对 JDBC 操作的一整套完整的流程有一个清晰的概念。

<!--more-->
```java
/**
 * copied from https://www.tutorialspoint.com/jdbc/jdbc-sample-code.htm
 * updated by DorMOUSENone
 */

//STEP 1. 引入必须的包
import java.sql.*;

public class Example {
	// JDBC 驱动名 与 DB URL 
    static final String JDBC_DRIVER = "com.mysql.jdbc.Driver";  
    static final String DB_URL = "jdbc:mysql://<host>:<port>/<dbName>";

  	// 数据库登录验证 (用户名、密码等)
    static final String USER = "username";
    static final String PASS = "password";
   
	public static void main(String[] args) {
		Connection conn = null;
		Statement stmt = null;
		try{
			//STEP 2: 注册驱动(注册到驱动管理器 DriverManager 类中)
			Class.forName("com.mysql.jdbc.Driver");

            //STEP 3: 创建一个连接
          	System.out.println("Connecting to database...");
          	conn = DriverManager.getConnection(DB_URL,USER,PASS);

          	//STEP 4: 执行一个查询
          	System.out.println("Creating statement...");
          	stmt = conn.createStatement();
          	String sql;
          	sql = "SELECT id, first, last, age FROM Employees";
          	ResultSet rs = stmt.executeQuery(sql);

          	//STEP 5: 将结果从结果集(ResultSet)中取出
          	while(rs.next()){
             	//根据列名逐一取出数据
             	int id  = rs.getInt("id");
             	int age = rs.getInt("age");
             	String first = rs.getString("first");
             	String last = rs.getString("last");

             	//展示结果
             	System.out.print("ID: " + id);
             	System.out.print(", Age: " + age);
             	System.out.print(", First: " + first);
             	System.out.println(", Last: " + last);
          	}
          	//STEP 6: 清理环境
          	rs.close();
          	stmt.close();
          	conn.close();
       	}catch(SQLException se){
          	//处理 JDBC 错误
          	se.printStackTrace();
       	}catch(Exception e){
          	//处理 Class.forName() 引起的错误
          	e.printStackTrace();
       	}finally{
          	// finally 代码库来关闭资源
          	try{
            	if(stmt!=null)
                	stmt.close();
          	}catch(SQLException se2){
          	}// 不做任何处理
          	try{
           		if(conn!=null)
                	conn.close();
            }catch(SQLException se){
               	se.printStackTrace();
            }
       	}
       	System.out.println("Goodbye!");
	}
}
```

从上面的通用 JDBC 代码可以看到，利用 Java 原生提供的 java.sql.* 包可以完成注册驱动，创建连接，执行查询，处理结果等一系列一整套操作。

而 Spring-JDBC 对于 Java 原生提供的 JDBC ，对一系列数据及操作进行了整合：

1. 实现了 DataSource 接口，用于整合数据源配置，将各种零散的属性值（诸如 URL、用户名、 密码等）整合成完整的一个对象，便于重用；也实现了 getConnection(…) 接口，便于直接通过 DataSource 获取连接。
2. 对执行查询的流程进行了封装。
3. 对结果集的处理提供了多种接口，针对不同的应用场景可自由选择所必须的实现类。例如直接通过 JDBC 获得 Bean ，而无需如上例 39-42 行所示，逐一编码进行提取。

本节提供对《Spring-JDBC 源码学习》完整学习过程的一个梳理：

1. [JdbcTemplate](https://blog.csdn.net/dormousenone/article/details/79035440) 一节作为学习 Spring-jdbc 的切入点，是 org.springframework.jdbc.core 包中的核心类，是一个提供不同场景下数据库操作的模板类。主要描述其持有的属性 DataSource 等以及其实现的方法。
2. [DataSource](https://blog.csdn.net/dormousenone/article/details/79037012) 一节承接 JdbcTemplate 一节，提供 Spring-jdbc 对数据源的一个包装与应用。同时描述其核心方法 getConnection(…) 的实现。
3. [DriverManager](https://blog.csdn.net/dormousenone/article/details/79042212) 描述其对不同数据库供应商提供的驱动的管理与使用方法。同时简单描述通过具体驱动获得一个连接的实现。
4. [PreparedStatement & CallableStatement](https://blog.csdn.net/dormousenone/article/details/79046865) 一节主要表述 Spring-jdbc 如何对执行查询的流程进行了封装。特别是对于 PreparedStatement 与 CallableStatement 这类预置可变 SQL 语句，在执行前必须对其中可变参数进行补全的 Statement 。
5. [ResultSet](https://blog.csdn.net/dormousenone/article/details/79062275) 描述 Spring-jdbc 如何对结果集进行处理，提供了多种不同的接口实现不同的处理逻辑。这种封装后的操作可以极大地简化直接使用 Java 原生 JDBC 所必须的硬编码提取数据的问题。

## JdbcTemplate

![](https://ws2.sinaimg.cn/large/006tNc79gy1fn90ghu9ufj30ki0jm75t.jpg)

JdbcTemplate 类作为 org.springframework.core 包下的核心类，对 Java 实现的 JDBC 进行了一定程度的封装，可以简化 JDBC 的使用，同时避免许多常见的错误。由该类来执行 JDBC 工作流，应用程序只需要提供 SQL 语句就可以获得 DB 执行结果（😀，当然实际操作上没有描述的这么简单）。

**使用 JdbcTemplate 类只需要实现两个回调接口 PreparedStatementCreator (创建一个 PreparedStatement，给定连接，提供 SQL 语句以及必要的参数), ResultSetExtractor (用于花式获取结果集)。当然，不实现上述两个接口也可以进行简单的数据库操作，比如只通过 JdbcTemplate 获取一个连接(Connection) 或者执行一个静态 SQL Update。**

*看不懂无所谓，先继续向下看，会做更详细的讲解。*

### JdbcAccessor

首先了解 JdbcTemplate 的实现，JdbcTemplate 继承了 JdbcAccessor 抽象类，该抽象类有两个主要属性—— **DataSource dataSource** & **SQLExceptionTranslator exceptionTranslator** ，以及一个懒加载标识符 **lazyInit** 。

其中，

- DataSource 可以认为是一个存储有与数据库相关的属性实例，主要方法包括 `Connection getConnection()` & `Connection getConnection(String username, String password)` 。
- SQLExceptionTranslator 利用策略模式将 Java 定义的 SQLException 转换成 Spring 声明的 DataAccessException。

JdbcAccessor 实现的 InitializingBean 接口，InitializingBean 接口为 Bean 提供了一个初始化实例的方法 —— afterPropertiesSet() 方法，凡是继承该接口的类，一般都在类构造方法中执行该方法。同样的，声明初始化实例调用的方法还可采用 `<bean id="" class="" init-method="myInitMethod"/>` 在初始化 bean 时执行 myInitMethod() 方法。具体执行顺序为 ：

1. bean 的属性注入
2. 调用 afterPropertiesSet() 方法
3. 执行 myInitMethod() 方法

此处继承该接口是为了实现 DataSource 验证以及 SQLExceptionTranslator 的懒加载 OR NOT 的选择。

```java
@Override
public void afterPropertiesSet() {
	if (getDataSource() == null) {	// 判断是否注入了 DataSource
		throw new IllegalArgumentException("Property 'dataSource' is required");
	}
	if (!isLazyInit()) {	// 根据懒加载标识符选择执行与否
		getExceptionTranslator();	// 获取一个 SQLExceptionTranslator 实例
	}
}
```

该方法在 BeanFactory 完成该 bean 的依赖注入后执行，将首先判断 DataSource 是否已经被注入，再根据懒加载标识符来决定是否实例化一个 SQLExceptionTranslator 。

### JdbcOperations

同时，JdbcTemplate 类实现了 JdbcOperations 接口，该接口定义了基本的 JDBC 操作。基本方法如下：

1. `<T> T execute(ConnectionCallback<T> action);`
2. `<T> T execute(StatementCallback<T> action);`
3. `<T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action);`
4. `<T> T execute(CallableStatementCreator csc, CallableStatementCallback<T> action);`

![](https://ws3.sinaimg.cn/large/006tNc79gy1fncp2ncqd6j31j00au0uv.jpg)

其它的 execute(…) , query(…), update(…) 等最终都将调用上述 4 种方法其一来完成目的。

详细观察上述方法的入参， ConnectionCallback, StatementCallback, PreparedStatementCallback 和 CallableStatementCallback 四个 Callback 接口，分别都是函数式接口，其中的唯一方法 doInXXX() 将在 execute() 中被调用，以此实现获得 ResultSet 并返回 Result 。execute 中调用 doInXXX() 的通用代码如下 （以 Statement 为例）：

*对于不理解**回调**的同学，请自行了解概念*

```java
public <T> T execute(StatementCallback<T> action) throws DataAccessException {
	// 通过工具类 DataSourceUtils 获取一个连接
  	Connection con = DataSourceUtils.getConnection(obtainDataSource());
  	// 一个 Statement 空实例，PreparedStatement, CallableStatement 类似
	Statement stmt = null;
	try {
		stmt = con.createStatement();	// 通过连接(Connection)获取一个 Statement
		applyStatementSettings(stmt);	// 配置 Statement 参数
      	// 回调执行 doInXXX() 方法, 并获得 result
		T result = action.doInStatement(stmt);	
		handleWarnings(stmt);
		return result;
	}
	catch (SQLException ex) {
		// Release Connection early, to avoid potential connection pool deadlock
		// in the case when the exception translator hasn't been initialized yet.
		String sql = getSql(action);
		JdbcUtils.closeStatement(stmt);
		stmt = null;
		DataSourceUtils.releaseConnection(con, getDataSource());
		con = null;
		throw translateException("StatementCallback", sql, ex);
	}
	finally {
		JdbcUtils.closeStatement(stmt);
		DataSourceUtils.releaseConnection(con, getDataSource());
	}
}
```

对于 Statement, PreparedStatement, CallableStatement 的区别，首先下图表现了三个接口的继承关系。

- Statement 可以支持静态 SQL 语句
- PreparedStatement 支持可变参数的 SQL 语句
- CallableStatement 支持可变参数的 SQL 语句，并且支持定制 DB 的输出结果

![](https://ws3.sinaimg.cn/large/006tNc79gy1fncpaec17yj30ew0hcdh7.jpg)

## DataSource

上一节在粗略地了解了 JdbcTemplate 提供的方法之后，下面先来对 DataSource 做一点了解。

### Java 提供的 DataSource 定义

DataSource 是 Java 核心库提供的接口。位于 javax.sql package 下。

DataSource 接口可以被视作是一个提供物理 DB 实例连接(Connection) 的工厂，通过 DataSource 持有的各种属性（包括 DB Url, Username, Password 等）来获取一个连接(Connection) 。DataSource 接口有三种不同的实现方案：

1. 最基本的实现——生产一个标准连接(Connection) 对象
2. 连接池方案——生产会被自动添加到连接池的对象
3. 分布式事物实现——生产一个可以支持分布式事物，并默认被添加到连接池的连接对象

包括两个对外提供连接(Connection) 对象的方法，

```java
Connection getConnection() throws SQLException;
Connection getConnection(String username, String password) throws SQLException;
```

其父接口 CommonDataSource 提供设置/获取 LogWriter，登录 DB 超时时间和获取父 Logger 的方法。

### Spring-JDBC 扩展的 DataSource 定义

在 Spring-jdbc 下，DataSources 最顶级的类是 AbstractDataSource ，对 DataSource 的所有父接口方法做了实现。但保留 getConnection() 方法由子类实现。

在 AbstractDriverBasedDataSource 中，定义了大量的参数，诸如 url, username 等，这些都被用来定位并定义与数据库实例的连接。

```java
public abstract class AbstractDriverBasedDataSource extends AbstractDataSource {

   @Nullable
   private String url;

   @Nullable
   private String username;

   @Nullable
   private String password;

   @Nullable
   private String catalog;

   @Nullable
   private String schema;

   @Nullable
   // 可以看到此处有一个 Properties 类
   private Properties connectionProperties;
   
   // 省略若干方法
  

	@Override
	public Connection getConnection() throws SQLException {
      	// 调用内部方法 getConnectionFromDriver()
		return getConnectionFromDriver(getUsername(), getPassword());
	}

	@Override
	public Connection getConnection(String username, String password) throws SQLException {
      	// 调用内部方法 getConnectionFromDriver()
		return getConnectionFromDriver(username, password);
	}
  
   // 定义了一个获取 Connection 的方法，由 getConnection() 方法调用，
   // 此方法主要是将属性做了一个整合
   // 具体获取 Connection 的逻辑仍然下放到子类实现 见 40 行
   protected Connection getConnectionFromDriver(@Nullable String username, @Nullable String password) throws SQLException {
		Properties mergedProps = new Properties();
		Properties connProps = getConnectionProperties();
		if (connProps != null) {
			mergedProps.putAll(connProps);
		}
		if (username != null) {
			mergedProps.setProperty("user", username);
		}
		if (password != null) {
			mergedProps.setProperty("password", password);
		}
		
     	// 获取 Connection 逻辑下放
		Connection con = getConnectionFromDriver(mergedProps);
		if (this.catalog != null) {
			con.setCatalog(this.catalog);
		}
		if (this.schema != null) {
			con.setSchema(this.schema);
		}
		return con;
	}
  
  	// 该类中获取 Connection 的方法是抽象方法
    protected abstract Connection getConnectionFromDriver(Properties props) throws SQLException;

}
```

整合方案为将除 url 外的所有参数整合在同一个 Properties 对象中 (其中，Properties 可以被认为是一个线程安全的 Hash Map) 。最终调用 `Connection getConnectionFromDriver(Properties props)` 获取连接。

AbstractDriverBasedDataSource 抽象类的两个子类 DriverManagerDataSource 和 SimpleDriverDataSource 都以不同方式获得了连接(Connection)，但总结而言，获取连接(Connection) 的任务被委托给了 Driver 来实现（其中，Driver 有 Java 定义接口，并由各数据库提供商提供，例如 MySQL 提供了 mysql-connector-java-XXX.jar 来完成针对相应数据库的具体实现）。

```java
// ----------------------------
// SimpleDriverDataSource 的实现
// ----------------------------
@Override
protected Connection getConnectionFromDriver(Properties props) throws SQLException {
	Driver driver = getDriver();
	String url = getUrl();
	Assert.notNull(driver, "Driver must not be null");
	if (logger.isDebugEnabled()) {
		logger.debug("Creating new JDBC Driver Connection to [" + url + "]");
	}
  	// 哈哈，重点在这... driver 在该类中被预先注入
	return driver.connect(url, props);
}

// -----------------------------
// DriverManagerDataSource 的实现
// -----------------------------
@Override
protected Connection getConnectionFromDriver(Properties props) throws SQLException {
	String url = getUrl();
	Assert.state(url != null, "'url' not set");
	if (logger.isDebugEnabled()) {
		logger.debug("Creating new JDBC DriverManager Connection to [" + url + "]");
	}
  	// 调了个内部函数
	return getConnectionFromDriverManager(url, props);
}

protected Connection getConnectionFromDriverManager(String url, Properties props) throws SQLException {
    // 委托给 DriverManager 类来获取连接
    // DriverManager 的主要操作是遍历在该管理类中注册的 Driver
    // 每个 Driver 实例都去尝试一下，能不能获得一个连接
    // 第一次在某个 Driver 中拿到一个连接即返回连接 (Connection)
	return DriverManager.getConnection(url, props);
}
```

简要的类图如下：

![](https://ws4.sinaimg.cn/large/006tNc79gy1fncrgqcjqoj31es15ajxj.jpg)

## DriverManager

上面提到 DataSource 获取连接(Connection) 的操作实质上将委托具体的 Driver 来提供 Connection 。有两种不同的方式，包括经由 DriverManager 遍历所有处于管理下的 Driver 尝试获取连接，或者在 DataSource 实例中直接声明一个特定的 Driver 来获取连接。

对于获取连接的具体操作，挖坑-待填。只描述简单的数据库供应商提供的 Driver 如何与 java 相联系。

###在 DriverManager 中注册 Driver 实例

通常在与数据库交互逻辑的 Java 代码中，都会有 `Class.forName("com.mysql.jdbc.Driver")` (此处以 MySQL 提供的 mysql-connector-java-XXX.jar 为例，下同）的代码块，加载指定的 com.mysql.jdbc.Driver 为 java.lang.Class 类。

*当然，在 JDBC 4.0 标准下，可以不必再显示声明 `Class.forName("")` 语句，Driver 也同样会在 DriverManager 初始化时自动注册。*

```java
// Class 类中对于 forName(String className) 的方法
// 作用为返回一个 java.lang.Class 实例。
public static Class<?> forName(String className) throws ClassNotFoundException {...}
```

同时， JVM 在加载类的过程中会执行类中的 static 代码块。下述 row 10 ~ 16 的代码片段将被执行。唯一的逻辑就是 new 一个 com.mysql.jdbc.Driver 实例，并将实例注册(registerDriver) 到 **java.sql.DriverManager** 中。 

```java
package com.mysql.jdbc;

public class Driver extends NonRegisteringDriver implements java.sql.Driver {
	// ~ Static fields/initializers
	// ---------------------------------------------

	//
	// Register ourselves with the DriverManager
	//
	static {
		try {
			java.sql.DriverManager.registerDriver(new Driver());
		} catch (SQLException E) {
			throw new RuntimeException("Can't register driver!");
		}
	}

	// ~ Constructors
	// -----------------------------------------------------------

	/**
	 * Construct a new driver and register it with DriverManager
	 * 
	 * @throws SQLException
	 *             if a database error occurs.
	 */
	public Driver() throws SQLException {
		// Required for Class.forName().newInstance()
	}
}
```

下面再来看一下 DriverManager 中的 registerDriver() 方法。

```java
public class DriverManager {

  	// DriverManager 维护一个线程安全的 Driver 列表
  	// 此处的 DriverInfo 里面即包装了 Driver 
    private final static CopyOnWriteArrayList<DriverInfo> registeredDrivers = 
      	new CopyOnWriteArrayList<>();

  	// 在 DriverManager 中注册 Driver
	public static synchronized void registerDriver(java.sql.Driver driver)
        throws SQLException {
        registerDriver(driver, null);
    }
  
	public static synchronized void registerDriver(java.sql.Driver driver,
            DriverAction da)
        throws SQLException {

      	/* 如果当前 Driver 不在列表中，即添加到列表。 */
        if(driver != null) {
            registeredDrivers.addIfAbsent(new DriverInfo(driver, da));
        } else {
            // This is for compatibility with the original DriverManager
            throw new NullPointerException();
        }

        println("registerDriver: " + driver);

    }
}
```

### 通过 DriverManager 获取连接(Connection)

上一节有提到过可以通过 DriverManager 来遍历获取连接，也可以直接声明具体 Driver 并获取连接。下面代码展示的是通过 DriverManager 获取连接的操作。 ~~哈哈哈，反正最后都是由具体驱动实现获取连接。~~

```java
public class DriverManager {
  	// 获取连接的 public 接口 (1)
	public static Connection getConnection(String url,
		java.util.Properties info) throws SQLException {
		return (getConnection(url, info, Reflection.getCallerClass()));
	}
	// 获取连接的 public 接口 (2)
    public static Connection getConnection(String url,
        String user, String password) throws SQLException {
        java.util.Properties info = new java.util.Properties();

        if (user != null) {
            info.put("user", user);
        }
        if (password != null) {
            info.put("password", password);
        }

        return (getConnection(url, info, Reflection.getCallerClass()));
    }
	// 获取连接的 public 接口 (3)
    public static Connection getConnection(String url)
        throws SQLException {
        java.util.Properties info = new java.util.Properties();
        return (getConnection(url, info, Reflection.getCallerClass()));
    }
 
  	// 获取连接的内部逻辑实现
    private static Connection getConnection(
        String url, java.util.Properties info, Class<?> caller) 
      	throws SQLException {
        /*
         * When callerCl is null, we should check the application's
         * (which is invoking this class indirectly)
         * classloader, so that the JDBC driver class outside rt.jar
         * can be loaded from here.
         */
        ClassLoader callerCL = caller != null ? caller.getClassLoader() : null;
        synchronized(DriverManager.class) {
            // synchronize loading of the correct classloader.
            if (callerCL == null) {
                callerCL = Thread.currentThread().getContextClassLoader();
            }
        }
		// url 是定位 DBMS 最重要的参数，不能为空
        if(url == null) {
            throw new SQLException("The url cannot be null", "08001");
        }

        println("DriverManager.getConnection(\"" + url + "\")");

      	// 遍历所有注册的 Driver ，并都尝试获取连接(Connection)
        SQLException reason = null;

        for(DriverInfo aDriver : registeredDrivers) {
          	// 判断注册的 Driver 是否由 ClassLoader callerCL 加载，不是则跳过
            if(isDriverAllowed(aDriver.driver, callerCL)) {
                try {
                    println("    trying " + aDriver.driver.getClass().getName());
                    // 获取连接，:) 还是由 driver 实例自行提供
                  	Connection con = aDriver.driver.connect(url, info);
                    if (con != null) {
                        // Success!
                        println("getConnection returning " + 
                                aDriver.driver.getClass().getName());
                        return (con);
                    }
                } catch (SQLException ex) {
                    if (reason == null) {
                        reason = ex;
                    }
                }

            } else {
                println("    skipping: " + aDriver.getClass().getName());
            }

        }

        // 如果运行到下列代码，则表明获取连接失败，抛出错误
        if (reason != null)    {
            println("getConnection failed: " + reason);
            throw reason;
        }

        println("getConnection: no suitable driver found for "+ url);
        throw new SQLException("No suitable driver found for "+ url, "08001");
    }

}
```

*简单的提一嘴，Connection 仍然只是一个针对 Java -> DB Server 的上层接口，如果想要更深层次地了解 Connection 与 DB Server 的交互，可以尝试去看一下 com.mysql.jdbc.MysqlIO 类，MySQL 实现的 JDBC4Connection 类也是在使用该类来实现对 DB Server 交互。（哈哈，只看过 MySQL 提供的 Driver 包）。*

## PreparedStatement & CallableStatement

在了解了 DataSource 获取连接(Connection) 的实质以及 JdbcTemplate 的通用接口之后，使用 Spring-jdbc 进行数据库相关的操作可以直截了当的利用如下代码进行实现（此处仅展示通过 Java 硬编码的形式进行实现，XML 配置方法类似）。

```java
public static void main(String... args) {
  	// 定义数据源
	DriverManagerDataSource dataSource = new DriverManagerDataSource();
  	// 配置参数
	dataSource.setUrl("jdbc:mysql://<host>:<port>/<dbName>?<props>");
   	dataSource.setUsername("<username>");
    dataSource.setPassword("<passwd>");
  	
  	// 实例化一个 JDBC 工具类
  	JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
  	// 执行相关 CRUD 操作	
  	jdbcTemplate.execute();
}
```

回顾第一节所讲的 JdbcTemplate 的 4 个基础方法：

1. `<T> T execute(ConnectionCallback<T> action);`
2. `<T> T execute(StatementCallback<T> action);`
3. `<T> T execute(PreparedStatementCreator psc, PreparedStatementCallback<T> action);`
4. `<T> T execute(CallableStatementCreator csc, CallableStatementCallback<T> action);`

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fndsm5tl7aj30zg0byq47.jpg)

四个基础方法都有一个特定的回调函数将通过预配置的 DataSource 得到的 Connection 或 更进一步的 Statement or PreparedStatement or CallableStatement 作为入参来执行定义的唯一方法。

以 `<T> T execute(StatementCallback<T> action);` 为例来了解一下方法的核心逻辑。

```java
public <T> T execute(StatementCallback<T> action) throws DataAccessException {
	// 通过工具类 DataSourceUtils 获取一个连接
  	Connection con = DataSourceUtils.getConnection(obtainDataSource());
  	// 一个 Statement 空实例，PreparedStatement, CallableStatement 类似
	Statement stmt = null;
	try {
		stmt = con.createStatement();	// 通过连接(Connection)获取一个 Statement
		applyStatementSettings(stmt);	// 配置 Statement 参数
      	// 回调执行 doInXXX() 方法, 并获得 result
		T result = action.doInStatement(stmt);	
		handleWarnings(stmt);
		return result;
    } catch() {
      
    } finally {
      
    }
}
```

但是，对于 PreparedStatement 和 CallableStatement 而言，获取一个 *XXX*Statement 的方式就有所不同了。

```java
// 获取 Statement 实例
Statement stmt = con.createStatement();
```

```java
// 获取 PreparedStatement 实例
// psc 是一个 PreparedStatementCreator 接口实现的实例
PreparedStatement ps = psc.createPreparedStatement(con);
```

```java
// 获取 CallableStatement 实例
// csc 是一个 CallableStatementCreator 接口实现的实例
CallableStatement cs = csc.createCallableStatement(con);
```

可以看到 Statement 直接通过 Connection 获取实例，但是 PreparedStatement 和 CallableStatement 就有所不同，其区别就在于 PreparedStatement 和 CallableStatement 两个 Statement 都是可以定制入参的，更甚者， CallableStatement 可以定制 DB 执行结果的出参。当然，核心还是 `con.prepareStatement() OR con.prepareCall()` 方法，只不过是将获取 XXXStatement 的操作下放给 XXXStatementCreator 实例类实现，给予使用者重复的自主权，同时也是逻辑解耦的一种操作。

例如：

SimplePreparedStatementCreator 这个 PreparedStatementCreator 接口的实现类，只是简单的调用了 `con.prepareStatement(sql)` 方法。

```java
private static class SimplePreparedStatementCreator 
  	implements PreparedStatementCreator, SqlProvider {

	private final String sql;

	@Override
	public PreparedStatement createPreparedStatement(Connection con) 
      	throws SQLException {
		return con.prepareStatement(this.sql);
	}
}
```

而对于 PreparedStatementCreatorImpl ，在 createPreparedStatement(Connection con) 的实现上，又添加了更多的操作。

```java
private class PreparedStatementCreatorImpl
		implements PreparedStatementCreator, PreparedStatementSetter, SqlProvider, ParameterDisposer {

	private final String actualSql;

	private final List<?> parameters;

	@Override
	public PreparedStatement createPreparedStatement(Connection con) throws SQLException {
		PreparedStatement ps;
		if (generatedKeysColumnNames != null || returnGeneratedKeys) {
			if (generatedKeysColumnNames != null) {
              	// 获取一个 PreparedStatement 实例，下同
				ps = con.prepareStatement(this.actualSql, 
                                          generatedKeysColumnNames);
			}
			else {
				ps = con.prepareStatement(this.actualSql, 
                                         PreparedStatement.RETURN_GENERATED_KEYS);
			}
		}
		else if (resultSetType == ResultSet.TYPE_FORWARD_ONLY 
                 && !updatableResults) {
			ps = con.prepareStatement(this.actualSql);
		}
		else {
			ps = con.prepareStatement(this.actualSql, resultSetType,
				updatableResults ? ResultSet.CONCUR_UPDATABLE : 
                                      ResultSet.CONCUR_READ_ONLY);
		}
      	// 将可变 SQL (例如 SELECT * FROM msg WHERE id = ?) 的 ? 用实际参数替换
		setValues(ps);
		return ps;
	}
}
```

从上述两种 PreparedStatementCreator 接口的不同实现，也可以从另一种角度理解到，函数式接口将获取目标出参的具体逻辑交给使用者定义，给予了使用者充分的自主权，同时也是一种业务逻辑的解耦。



在上面代码 PreparedStatementCreatorImpl 类的实现中，我们看到第 32 行代码 `setValue(ps)` 。此处的方法是由接口 PreparedStatementSetter 定义的。主要目的是将可变 SQL 中的 ? 参数用实际参数进行一个替换。

```java
// 以 SQL (SELECT * FROM msg WHERE id = ? AND day = ?) 为例
/** 纯 Java 核心库的实现 PreparedStatement 参数注入 */
ps.setInt(1, 1763);
ps.setString(2, "2018-01-01");
ps.executeQuery();

/** 以 Spring-jdbc 实现 PreparedStatement 参数注入 */
// setValues() 由接口 PreparedStatementSetter 定义，封装了注入参数的具体实现逻辑
// 可以由使用者自行定义
setValues(ps);	
ps.executeQuery();
```

![](https://ws1.sinaimg.cn/large/006tKfTcgy1fndyybh4q1j31kw0u3afx.jpg)

上面类图表示了 PreparedStatementSetter 及其实现类的相关依赖。

Setter 的主要目标即为对 SQL 中的 ? 参数进行注入。

~~个人精力有限，对Spring-jdbc在Statement上对参数注入的回调理解有限。~~

## ResultSet

在前几节已经提到讲了数据源、驱动管理器以及 Statement 之后，利用 JDBC 的最重要的目的就是对 DB 进行操作，并获得预期结果。对于查询语句而言，结果应该是若干记录行；就更新语句而言，结果可能是影响的行数。而 Spring-jdbc 对 ResultSet 额外进行的封装，即是将原本散乱的结果进行一个整合，例如整合成一个(一组)完整的 Bean 来进行展示。

在 JdbcTemplate 中，四个基本方法入参都包括一个回调接口，而在执行回调获得 ResultSet 之后，方法并不是直接返回，而是进行了一定的操作。

以一个 PreparedStatementCallback 的实例类为例，在 doInPreparedStatement() 方法中，获得了 ResultSet ，但是仍通过 rse.extractData(rs) 语句进行了处理后再返回结果

```java
public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
  	// 声明一个 ResultSet 
	ResultSet rs = null;
	try {
		if (pss != null) {	// setValues() 方法填充 PreparedStatement 中的可变参数 ? 
			pss.setValues(ps);
		}
		rs = ps.executeQuery();	// 执行查询 sql ，获取结果
		return rse.extractData(rs);	// 重点... 该语句一定是对结果进行了一些操作.
	}
	finally {
		JdbcUtils.closeResultSet(rs);
		if (pss instanceof ParameterDisposer) {
			((ParameterDisposer) pss).cleanupParameters();
		}
	}
}
```

在来看一下究竟在返回结果前进行了什么操作。

由于是一个回调接口的实现类，rse 应该在外部方法中。

```java
public <T> T query(
		PreparedStatementCreator psc, @Nullable final PreparedStatementSetter pss, 
  		final ResultSetExtractor<T> rse)
		throws DataAccessException {

	return execute(psc, new PreparedStatementCallback<T>() {
		@Override
		public T doInPreparedStatement(PreparedStatement ps) throws SQLException {
			...	
        }
	});
}
```

可以看到 rse 是一个 ResultSetExtractor<T> 的实例。从名称上可以看出 Extractor 即为提取器，结果集提取器。

```java
/** 函数式接口，提供的唯一方法为 extractData(...) */
@FunctionalInterface
public interface ResultSetExtractor<T> {

	@Nullable
	T extractData(ResultSet rs) throws SQLException, DataAccessException;

}
```

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fndzz0pqnrj31a60i676v.jpg)

Spring-jdbc 中现有的实现有三个类，其中 RowCallbackHandlerResultSetExtractor 和 RowMapperResultSetExtractor 两个类从上面类图及命名即可看出，两个是作为一个适配器而存在的，将 extractData() 进行转换，分别由 RowCallbackHandler 和 RowMapper 实例进行具体操作。而 AbstractLobStreamingResultSetExtractor 类从名称上看，也是一个类似流操作的实现类(其中的 Lob 指的是 DB 中的 BLOB 与 CLOB 类型，在 Spring-jdbc 中也加入了额外的支持) 。

### RowCallbackHandler

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fnh2uw3a0aj30h20jeta4.jpg)

从名称上看，这是一个与 ResultSet 结果集行相关的回调接口。processRow(ResultSet rs) 方法将处理行相关的逻辑。

从它的一个实现类 RowCountCallbackHandler 来说，其最主要的私有属性 rowCount 即是统计 ResultSet 的结果行数。下面是一个具体使用的该类的案例：

```java
public static void main(String[] args) throws ClassNotFoundException {

    DriverManagerDataSource dataSource = new DriverManagerDataSource();
    dataSource.setUrl("jdbc:mysql://<host>:<port>/<dbName>");
    dataSource.setUsername("<username>");
    dataSource.setPassword("<password>");

    JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);
    RowCountCallbackHandler rcch = new RowCountCallbackHandler();

    jdbcTemplate.query("SELECT * FROM info WHERE id='2018'", (RowCallbackHandler) rcch);
	
  	System.out.println(rcch.getRowCount());	//获取结果集行数
    System.out.println(rcch.getColumnCount());	// 获取结果集列数
    for (String arg : rcch.getColumnNames()) {	// 打印结果集每一列名称
        System.out.println("ColumnNames : " + arg);
    }
    for (int i : rcch.getColumnTypes()) {	// 打印结果集每一列类型(Types 为枚举类)
        System.out.println("ColumnTypes : " + i);
    }
}
```

具体查看 RowCountCallbackHandler 类的 processRow() 方法可以看到，其获取列相关信息都来自于 ResultSetMetaData 。而结果行数来源于 Iterator 迭代。

```java
@Override
public final void processRow(ResultSet rs) throws SQLException {
	if (this.rowCount == 0) {
		ResultSetMetaData rsmd = rs.getMetaData();
		this.columnCount = rsmd.getColumnCount();
		this.columnTypes = new int[this.columnCount];
		this.columnNames = new String[this.columnCount];
		for (int i = 0; i < this.columnCount; i++) {
			this.columnTypes[i] = rsmd.getColumnType(i + 1);
			this.columnNames[i] = JdbcUtils.lookupColumnName(rsmd, i + 1);
		}
		// could also get column names
	}
	processRow(rs, this.rowCount++);
}
```

### RowMapper

上面 RowCallbackHandler 接口从逻辑上划分是用于处理 ResultSet 元数据信息的。而 RowMapper 较上一个接口而言，有更高的实用性。

![](https://ws3.sinaimg.cn/large/006tKfTcgy1fnh3h739kaj30z60nago4.jpg)

特别是其实现类 BeanPropertyRowMapper<T> 可以将散乱的 ResultSet 的结果数据整理成一个个 Object Bean 作为返回结果。而省去了对逐一读取 ResultSet 行记录并填充 Bean 属性的麻烦。

```java
JdbcTemplate jdbcTemplate = new JdbcTemplate(dataSource);

BeanPropertyRowMapper<Model> rowMapper = new BeanPropertyRowMapper<>(Model.class);
List<Model> list = jdbcTemplate.query("SELECT * FROM info WHERE id = '2018'", rowMapper);
/** 获得的 Model list 的对应属性将通过 Java 的反射机制进行填充
 *	List<Model> list 即结果
 */
```

当然，要求是 Model 类中的属性与 DB table 中的列名保持一致（大小写无要求）。

而其它两个类的实现也是基于反射机制，实现其相应的业务逻辑。

```text
  __                    __                  
 / _| __ _ _ __   __ _ / _| ___ _ __   __ _ 
| |_ / _` | '_ \ / _` | |_ / _ \ '_ \ / _` |
|  _| (_| | | | | (_| |  _|  __/ | | | (_| |
|_|  \__,_|_| |_|\__, |_|  \___|_| |_|\__, |
                 |___/                |___/ 
```
