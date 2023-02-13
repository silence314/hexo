---
title: Mybatis比JDBC好在哪
tags:
  - Mybatis
typora-root-url: ../../themes/butterfly/source
date: 2020-11-01 10:06:42
description: 自己在项目中查询hive时是用JDBC以及Druid数据库连接池连接的，但是Mybatis好在哪里呢？改写的还是要写，甚至多了很多标签要了解
cover: /blogImg/Mybatis架构图.png
categories: 数据库框架
---

# JDBC是怎么查询的

通过JDBC查询数据库数据，一般需要以下七个步骤：

1. 加载JDBC驱动；
2. 建立并获取数据库连接；
3. 创建 JDBC Statements 对象；
4. 设置SQL语句的传入参数；
5. 执行SQL语句并获得查询结果；
6. 对查询结果进行转换处理并将处理结果返回；
7. 释放相关资源（关闭Connection，关闭Statement，关闭ResultSet）；

```java
public static List<Map<String,Object>> queryForList(){  
    Connection connection = null;  
    ResultSet rs = null;  
    PreparedStatement stmt = null;  
    List<Map<String,Object>> resultList = new ArrayList<Map<String,Object>>();  
          
    try {  
        // 加载JDBC驱动  
        Class.forName("oracle.jdbc.driver.OracleDriver").newInstance();  
        String url = "jdbc:oracle:thin:@localhost:1521:ORACLEDB";  
              
        String user = "trainer";   
        String password = "trainer";   
              
        // 获取数据库连接  
        connection = DriverManager.getConnection(url,user,password);   
              
        String sql = "select * from userinfo where user_id = ? ";  
        // 创建Statement对象（每一个Statement为一次数据库执行请求）  
        stmt = connection.prepareStatement(sql);  
              
        // 设置传入参数  
        stmt.setString(1, "zhangsan");  
              
        // 执行SQL语句  
        rs = stmt.executeQuery();  
              
        // 处理查询结果（将查询结果转换成List<Map>格式）  
        ResultSetMetaData rsmd = rs.getMetaData();  
        int num = rsmd.getColumnCount();  
              
        while(rs.next()){  
            Map map = new HashMap();  
            for(int i = 0;i < num;i++){  
                String columnName = rsmd.getColumnName(i+1);  
                map.put(columnName,rs.getString(columnName));  
            }  
            resultList.add(map);  
        }  
              
    } catch (Exception e) {  
        e.printStackTrace();  
    } finally {  
        try {  
            // 关闭结果集  
            if (rs != null) {  
                rs.close();  
                rs = null;  
            }  
            // 关闭执行  
            if (stmt != null) {  
                stmt.close();  
                stmt = null;  
            }  
            if (connection != null) {  
                connection.close();  
                connection = null;  
            }  
        } catch (SQLException e) {  
            e.printStackTrace();  
        }  
    }        
    return resultList;  
}
```

当然可以通过数据库连接池来复用连接的部分，我在Hive查询时也是这样做的。

# Mybatis是怎么做的

上面我们看到了实现JDBC有七个步骤，哪些步骤是可以进一步封装的，减少我们开发的代码量。

## 第一步优化：连接获取和释放

**问题1：**

数据库连接频繁的开启和关闭本身就造成了**资源的浪费，影响系统的性能**。和线程类似，需要一个线程池

**解决问题：**

数据库连接的获取和关闭我们**可以使用数据库连接池来解决资源浪费的问题**。通过连接池就可以反复利用已经建立的连接去访问数据库了。减少连接的开启和关闭的时间。

**问题2：**

但是现在**连接池多种多样，可能存在变化**，有可能采用DBCP的连接池，也有可能采用容器本身的JNDI数据库连接池。

**解决问题：**

我们可以**通过DataSource进行隔离解耦**，我们统一从DataSource里面获取数据库连接，**DataSource具体由DBCP实现还是由容器的JNDI实现都可以**，所以我们将DataSource的具体实现通过让用户配置来应对变化。

## 第二步优化：SQL统一存取

**问题：**

我们使用JDBC进行操作数据库时，**SQL语句基本都散落在各个JAVA类中**，这样有三个不足之处：

第一，可读性很差，不利于维护以及做性能调优。

第二，改动Java代码需要重新编译、打包部署。

第三，不利于取出SQL在数据库客户端执行（取出后还得删掉中间的Java代码，编写好的SQL语句写好后还得通过＋号在Java进行拼凑）。

**解决问题：**

我们可以考虑不把SQL语句写到Java代码中，那么把SQL语句放到哪里呢？首先需要有一个统一存放的地方，我们可以将这些**SQL语句统一集中放到配置文件或者数据库里面（以key-value的格式存放）**。然后通过SQL语句的key值去获取对应的SQL语句。

既然我们将SQL语句都统一放在配置文件或者数据库中，**那么这里就涉及一个SQL语句的加载问题**。

## 第三步优化：传入参数映射和动态SQL

**问题：**

很多情况下，我们都可以通过在SQL语句中设置占位符来达到使用传入参数的目的，这种方式本身就有一定局限性，它是按照一定顺序传入参数的，要与占位符一一匹配。但是，如果我们**传入的参数是不确定的**（比如列表查询，根据用户填写的查询条件不同，传入查询的参数也是不同的，有时是一个参数、有时可能是三个参数），那么我们就得**在后台代码中自己根据请求的传入参数去拼凑相应的SQL语句**，这样的话还是**避免不了在Java代码里面写SQL语句的命运**。既然我们已经把SQL语句统一存放在配置文件中了，**怎么做到能够根据前台传入参数的不同，动态生成对应的SQL语句呢？**

**解决问题：**

第一，我们先解决这个动态问题，**按照我们正常的程序员思维是，通过if和else这类的判断来进行是最直观的**，这个时候我们想到了JSTL中的<if test=""></if>这样的标签，那么，能不能将这类的标签引入到SQL语句中呢？假设可以，那么我们这里就需要一个专门的SQL解析器来解析这样的SQL语句，但是，if判断的变量来自于哪里呢？传入的值本身是可变的，那么我们得为这个值定义一个不变的变量名称，而且这个变量名称必须和对应的值要有对应关系，可以通过这个变量名称找到对应的值，这个时候我们想到了key-value的Map。解析的时候根据变量名的具体值来判断。

假如前面可以判断没有问题，那么假如判断的结果是true，那么就需要输出的标签里面的SQL片段，但是怎么解决在标签里面使用变量名称的问题呢？这里我们需要**使用一种有别于SQL的语法来嵌入变量（比如使用＃变量名＃）**。这样，SQL语句经过解析后就可以动态的生成符合上下文的SQL语句。

还有，**怎么区分开占位符变量和非占位变量？**有时候我们单单使用占位符是满足不了的，占位符只能为查询条件占位，SQL语句其他地方使用不了。**这里我们可以使用#变量名#表示占位符变量，使用变量名表示非占位符变量**。

## 第四步优化：结果映射和结果缓存

**问题：**

执行SQL语句、获取执行结果、对执行结果进行转换处理、释放相关资源是一整套下来的。假如是执行查询语句，那么执行SQL语句后，返回的是一个ResultSet结果集，**这个时候我们就需要将ResultSet对象的数据取出来，不然等到释放资源时就取不到这些结果信息了**。我们从前面的优化来看，以及将获取连接、设置传入参数、执行SQL语句、释放资源这些都封装起来了，只剩下结果处理这块还没有进行封装，如果能封装起来，每个数据库操作都不用自己写那么一大堆Java代码，直接调用一个封装的方法就可以搞定了。

**解决问题：**

我们分析一下，一般对执行结果的有哪些处理，**有可能将结果不做任何处理就直接返回，也有可能将结果转换成一个JavaBean对象返回、一个Map返回、一个List返回等**，结果处理可能是多种多样的。从这里看，我们必须告诉SQL处理器两点：**第一，需要返回什么类型的对象；第二，需要返回的对象的数据结构怎么跟执行的结果映射**，这样才能将具体的值copy到对应的数据结构上。

接下来，**我们可以进而考虑对SQL执行结果的缓存来提升性能**。缓存数据都是key-value的格式，那么这个key怎么来呢？怎么保证唯一呢？即使同一条SQL语句几次访问的过程中由于传入参数的不同，得到的执行SQL语句也是不同的。那么缓存起来的时候是多对。**但是SQL语句和传入参数两部分合起来可以作为数据缓存的key值**。

## 第五步优化：解决重复SQL语句问题

**问题：**

由于我们将所有SQL语句都放到配置文件中，**这个时候会遇到一个SQL重复的问题**，几个功能的SQL语句其实都差不多，有些可能是SELECT后面那段不同、有些可能是WHERE语句不同。有时候表结构改了，那么我们就需要改多个地方，不利于维护。

**解决问题：**

当我们的代码程序出现重复代码时怎么办？**将重复的代码抽离出来成为独立的一个类，然后在各个需要使用的地方进行引用**。对于SQL重复的问题，我们也可以采用这种方式，通过将SQL片段模块化，**将重复的SQL片段独立成一个SQL块，然后在各个SQL语句引用重复的SQL块**，这样需要修改时只需要修改一处即可。

# Mybatis框架设计

![Mybatis架构图](/blogImg/Mybatis架构图.png)

## 接口层

MyBatis和数据库的交互有两种方式：

1. 使用传统的MyBatis提供的API；
2. 使用Mapper接口；

### 传统的MyBatis提供的API

**这是传统的传递Statement Id 和查询参数给 SqlSession 对象，使用 SqlSession对象完成和数据库的交互**；MyBatis提供了非常方便和简单的API，供用户实现对数据库的增删改查数据操作，以及对数据库连接信息和MyBatis 自身配置信息的维护操作。

![传统Mybatis工作模式](/blogImg/传统Mybatis工作模式.jpg)

上述使用MyBatis 的方法，是**创建一个和数据库打交道的SqlSession对象，然后根据Statement Id 和参数来操作数据库**，这种方式固然很简单和实用，但是**它不符合面向对象语言的概念和面向接口编程的编程习惯**。由于面向接口的编程是面向对象的大趋势，MyBatis 为了适应这一趋势，增加了第二种使用MyBatis 支持接口（Interface）调用方式。

### Mapper接口

**MyBatis 将配置文件中的每一个<mapper> 节点抽象为一个 Mapper 接口：**

**这个接口中声明的方法和<mapper> 节点中的<select|update|delete|insert> 节点项对应**，即<select|update|delete|insert> 节点的id值为Mapper 接口中的方法名称，**parameterType 值表示Mapper 对应方法的入参类型**，而**resultMap 值则对应了Mapper 接口表示的返回值类型或者返回结果集的元素类型**。

```xml
 <mapper namespace="com.vipkid.teacher.dao.mapper.w9.ShippingMethodLogMapper">
  <select id="countByParam" parameterType="com.vipkid.teacher.param.ShippingMethodLogParam" resultType="int">
    select
    count(1)
    from shipping_method_log
    <include refid="paramWhere"/>
  </select>
</mapper>
```

**根据MyBatis 的配置规范配置好后，通过SqlSession.getMapper(XXXMapper.class)方法，MyBatis 会根据相应的接口声明的方法信息，通过动态代理机制生成一个Mapper 实例**，我们使用Mapper接口的某一个方法时，MyBatis会根据这个方法的方法名和参数类型，确定Statement Id，底层还是通过SqlSession.select("statementId",parameterObject);或者SqlSession.update("statementId",parameterObject); 等等来实现对数据库的操作，**MyBatis引用Mapper 接口这种调用方式，纯粹是为了满足面向接口编程的需要**。（其实还有一个原因是在于，面向接口的编程，使得用户在接口上可以使用注解来配置SQL语句，这样就可以脱离XML配置文件，实现“0配置”）。

![Mapper接口和Mapper.xml配置文件之间的对应关系](/blogImg/Mapper接口和Mapper.xml配置文件之间的对应关系.jpg)

## 数据处理层

**数据处理层可以说是MyBatis的核心**，从大的方面上讲，它要完成两个功能：

1. 通过传入参数构建动态SQL语句；
2. SQL语句的执行以及封装查询结果集成List<E>；

### 参数映射和动态SQL语句生成

动态语句生成可以说是MyBatis框架非常优雅的一个设计，**MyBatis 通过传入的参数值，来动态地构造SQL语句**，使得MyBatis 有很强的灵活性和扩展性。

**参数映射指的是对于java 数据类型和jdbc数据类型之间的转换：**这里有包括两个过程：**查询阶段**，我们要将java类型的数据，转换成jdbc类型的数据，通过 preparedStatement.setXXX() 来设值；另一个就是**对resultset查询结果集的jdbcType 数据转换成java 数据类型**。

### SQL语句的执行以及封装查询结果集成List<E>

动态SQL语句生成之后，MyBatis 将执行SQL语句，并将可能返回的结果集转换成List<E> 列表。**MyBatis 在对结果集的处理中，支持结果集关系一对多和多对一的转换**，并且有两种支持方式，**一种为嵌套查询语句的查询，还有一种是嵌套结果集的查询**。

## 框架支撑层

1. 事务管理机制
**事务管理机制对于ORM框架而言是不可缺少的一部分**，事务管理机制的质量也是考量一个ORM框架是否优秀的一个标准。
2. 连接池管理机制
由于创建一个数据库连接所占用的资源比较大，**对于数据吞吐量大和访问量非常大的应用而言，连接池的设计就显得非常重要**。
3. 缓存机制
为了提高数据利用率和减小服务器和数据库的压力，**MyBatis 会对于一些查询提供会话级别的数据缓存**，会将对某一次查询，放置到SqlSession 中，在允许的时间间隔内，对于完全相同的查询，MyBatis会直接将缓存结果返回给用户，而不用再到数据库中查找。
4. SQL语句的配置方式
传统的MyBatis 配置SQL语句方式就是使用XML文件进行配置的，但是这种方式不能很好地支持面向接口编程的理念，**为了支持面向接口的编程，MyBatis 引入了Mapper接口的概念，面向接口的引入，对使用注解来配置SQL语句成为可能，用户只需要在接口上添加必要的注解即可，不用再去配置XML文件了**，但是，目前的MyBatis 只是对注解配置SQL语句提供了有限的支持，某些高级功能还是要依赖XML配置文件配置SQL 语句。

## 引导层

**引导层是配置和启动MyBatis配置信息的方式**。MyBatis 提供两种方式来引导MyBatis ：**基于XML配置文件的方式和基于Java API 的方式**。

## 主要构件及其相互关系

从MyBatis代码实现的角度来看，MyBatis的主要的核心部件有以下几个：

> **SqlSession：**作为MyBatis工作的主要顶层API，表示和数据库交互的会话，完成必要数据库增删改查功能；
>
> **Executor：**MyBatis执行器，是MyBatis 调度的核心，负责SQL语句的生成和查询缓存的维护；
>
> **StatementHandler：**封装了JDBC Statement操作，负责对JDBC statement 的操作，如设置参数、将Statement结果集转换成List集合。
>
> **ParameterHandler：**负责对用户传递的参数转换成JDBC Statement 所需要的参数；
>
> **ResultSetHandler：**负责将JDBC返回的ResultSet结果集对象转换成List类型的集合；
>
> **TypeHandler：**负责java数据类型和jdbc数据类型之间的映射和转换；
>
> **MappedStatement：**MappedStatement维护了一条<select|update|delete|insert>节点的封装；
>
> **SqlSource：**负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中，并返回；
>
> **BoundSql：**表示动态生成的SQL语句以及相应的参数信息；
>
> **Configuration：**MyBatis所有的配置信息都维持在Configuration对象之中；

**它们的关系如下图所示：**

![Mybatis的结构层次](/blogImg/Mybatis的结构层次.jpg)

# 工作过程分析

1. **开启一个数据库访问会话---创建SqlSession对象**

<img src="/blogImg/SqlSession的作用.jpg" alt="SqlSession的作用" style="zoom:50%;" />

2. **为SqlSession传递一个配置的Sql语句的Statement Id和参数，然后返回结果：**

```java
List<Employee> result = sqlSession.selectList("com.louis.mybatis.dao.EmployeesMapper.selectByMinSalary",params);
```

sqlSession.selectList()方法：

```java
    @Override
    public <E> List<E> selectList(String statement, Object parameter) {
        return this.selectList(statement, parameter, RowBounds.DEFAULT);
    }

    @Override
    public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
        try {
            MappedStatement ms = configuration.getMappedStatement(statement);
            return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
        } catch (Exception e) {
            throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
        } finally {
            ErrorContext.instance().reset();
        }
    }
```

MyBatis在初始化的时候，会将MyBatis的配置信息全部加载到内存中，**使用org.apache.ibatis.session.Configuration实例来维护**。使用者可以使用sqlSession.getConfiguration()方法来获取。**MyBatis的配置文件中配置信息的组织格式和内存中对象的组织格式几乎完全对应的**。

**加载到内存中会生成一个对应的MappedStatement对象，然后会以key="com.louis.mybatis.dao.EmployeesMapper.selectByMinSalary" ，value为MappedStatement对象的形式维护到Configuration的一个Map中**。当以后需要使用的时候，只需要通过Id值来获取就可以了。

从上述的代码中我们可以看到SqlSession的职能是：**SqlSession根据Statement ID, 在mybatis配置对象Configuration中获取到对应的MappedStatement对象，然后调用mybatis执行器来执行具体的操作**。

3. **MyBatis执行器Executor根据SqlSession传递的参数执行query()方法**

```java
/** 
   * BaseExecutor 类部分代码 
   * 
   */  
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
      // 1. 根据具体传入的参数，动态地生成需要执行的SQL语句，用BoundSql对象表示    
      BoundSql boundSql = ms.getBoundSql(parameter);  
      // 2. 为当前的查询创建一个缓存Key  
      CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);  
      return query(ms, parameter, rowBounds, resultHandler, key, boundSql);  
}  
 
@SuppressWarnings("unchecked")  
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {  
       ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());  
       if (closed) throw new ExecutorException("Executor was closed.");  
       if (queryStack == 0 && ms.isFlushCacheRequired()) {  
           clearLocalCache();  
       }  
       List<E> list;  
       try {  
           queryStack++;  
           list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;  
           if (list != null) {  
               handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);  
           } else {  
               // 3.缓存中没有值，直接从数据库中读取数据    
               list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);  
           }  
       } finally {  
           queryStack--;  
       }  
       if (queryStack == 0) {  
           for (DeferredLoad deferredLoad : deferredLoads) {  
               deferredLoad.load();  
           }  
           deferredLoads.clear(); // issue #601  
           if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {  
               clearLocalCache(); // issue #482  
           }  
       }  
       return list;  
}

private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {  
      List<E> list;  
      localCache.putObject(key, EXECUTION_PLACEHOLDER);  
      try {  
         
          //4. 执行查询，返回List 结果，然后    将查询的结果放入缓存之中  
          list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);  
      } finally {  
          localCache.removeObject(key);  
      }  
      localCache.putObject(key, list);  
      if (ms.getStatementType() == StatementType.CALLABLE) {  
          localOutputParameterCache.putObject(key, parameter);  
      }  
      return list;  
} 

```

```java
/** 
   * 
   * SimpleExecutor类的doQuery()方法实现 
   * 
   */  
public <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {  
      Statement stmt = null;  
      try {  
          Configuration configuration = ms.getConfiguration();  
          //5. 根据既有的参数，创建StatementHandler对象来执行查询操作  
          StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);  
          //6. 创建java.Sql.Statement对象，传递给StatementHandler对象  
          stmt = prepareStatement(handler, ms.getStatementLog());  
          //7. 调用StatementHandler.query()方法，返回List结果集  
          return handler.<E>query(stmt, resultHandler);  
       } finally {  
           closeStatement(stmt);  
       }  
}
```

上述的Executor.query()方法几经转折，**最后会创建一个StatementHandler对象，然后将必要的参数传递给StatementHandler**，使用StatementHandler来完成对数据库的查询，最终返回List结果集。

**从上面的代码中我们可以看出，Executor的功能和作用是：**

> 1. 根据传递的参数，完成SQL语句的动态解析，生成BoundSql对象，供StatementHandler使用；
> 2. 为查询创建缓存，以提高性能；
> 3. 创建JDBC的Statement连接对象，传递给StatementHandler对象，返回List查询结果；

4. **StatementHandler对象负责设置Statement对象中的查询参数、处理JDBC返回的resultSet，将resultSet加工为List 集合返回：**

接着上面的Executor第六步，看一下：prepareStatement() 方法的实现：

```java
private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
      Statement stmt;  
      Connection connection = getConnection(statementLog);  
      stmt = handler.prepare(connection);  
      //对创建的Statement对象设置参数，即设置SQL 语句中 ? 设置为指定的参数  
      handler.parameterize(stmt);  
      return stmt;  
}
```

以上我们可以总结StatementHandler对象主要完成两个工作：

> 1. 对于JDBC的PreparedStatement类型的对象，创建的过程中，我们使用的是SQL语句字符串会包含 若干个? 占位符，我们其后再对占位符进行设值。
> 	 **StatementHandler通过parameterize(statement)方法对Statement进行设值；**
> 2. StatementHandler通过List<E> query(Statement statement, ResultHandler resultHandler)方法来完成执行Statement，和将Statement对象返回的resultSet封装成List；

5. StatementHandler 的parameterize(statement) 方法的实现：

```java
/** 
   * StatementHandler 类的parameterize(statement) 方法实现  
   */  
public void parameterize(Statement statement) throws SQLException {  
      //使用ParameterHandler对象来完成对Statement的设值    
      parameterHandler.setParameters((PreparedStatement) statement);  
}  
```

```java
/** 
   *  
   * ParameterHandler类的setParameters(PreparedStatement ps) 实现 
   * 对某一个Statement进行设置参数 
   */  
public void setParameters(PreparedStatement ps) throws SQLException {  
      ErrorContext.instance().activity("setting parameters").object(mappedStatement.getParameterMap().getId());  
      List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();  
      if (parameterMappings != null) {  
          for (int i = 0; i < parameterMappings.size(); i++) {  
              ParameterMapping parameterMapping = parameterMappings.get(i);  
              if (parameterMapping.getMode() != ParameterMode.OUT) {  
                  Object value;  
                  String propertyName = parameterMapping.getProperty();  
                  if (boundSql.hasAdditionalParameter(propertyName)) { // issue #448 ask first for additional params  
                      value = boundSql.getAdditionalParameter(propertyName);  
                  } else if (parameterObject == null) {  
                      value = null;  
                  } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {  
                      value = parameterObject;  
                  } else {  
                      MetaObject metaObject = configuration.newMetaObject(parameterObject);  
                      value = metaObject.getValue(propertyName);  
                  }  
         
                  // 每一个Mapping都有一个TypeHandler，根据TypeHandler来对preparedStatement进行设置参数  
                  TypeHandler typeHandler = parameterMapping.getTypeHandler();  
                  JdbcType jdbcType = parameterMapping.getJdbcType();  
                  if (value == null && jdbcType == null) jdbcType = configuration.getJdbcTypeForNull();  
                  // 设置参数  
                  typeHandler.setParameter(ps, i + 1, value, jdbcType);  
              }  
          }  
      }  
}
```

从上述的代码可以看到,StatementHandler的parameterize(Statement) 方法调用了 ParameterHandler的setParameters(statement) 方法，
 **ParameterHandler的setParameters(Statement)方法负责 根据我们输入的参数，对statement对象的 ? 占位符处进行赋值。**

6. **StatementHandler 的List<E> query(Statement statement, ResultHandler resultHandler)方法的实现：**

```java
 /** 
    * PreParedStatement类的query方法实现 
    */  
public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {  
      //1.调用preparedStatemnt。execute()方法，然后将resultSet交给ResultSetHandler处理    
      PreparedStatement ps = (PreparedStatement) statement;  
      ps.execute();  
      //2. 使用ResultHandler来处理ResultSet  
      return resultSetHandler.<E> handleResultSets(ps);  
}  
```

从上述代码我们可以看出，StatementHandler 的List<E> query(Statement statement, ResultHandler resultHandler)方法的实现，是调用了ResultSetHandler的handleResultSets(Statement) 方法。**ResultSetHandler的handleResultSets(Statement) 方法会将Statement语句执行后生成的resultSet 结果集转换成List<E> 结果集**：

```java
/**   
   * ResultSetHandler类的handleResultSets()方法实现 
   *  
   */  
public List<Object> handleResultSets(Statement stmt) throws SQLException {  
      final List<Object> multipleResults = new ArrayList<Object>();  
 
      int resultSetCount = 0;  
      ResultSetWrapper rsw = getFirstResultSet(stmt);  
 
      List<ResultMap> resultMaps = mappedStatement.getResultMaps();  
      int resultMapCount = resultMaps.size();  
      validateResultMapsCount(rsw, resultMapCount);  
     
      while (rsw != null && resultMapCount > resultSetCount) {  
          ResultMap resultMap = resultMaps.get(resultSetCount);  
       
          //将resultSet  
          handleResultSet(rsw, resultMap, multipleResults, null);  
          rsw = getNextResultSet(stmt);  
          cleanUpAfterHandlingResultSet();  
          resultSetCount++;  
      }
 
      String[] resultSets = mappedStatement.getResulSets();  
      if (resultSets != null) {  
          while (rsw != null && resultSetCount < resultSets.length) {  
              ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);  
              if (parentMapping != null) {  
                  String nestedResultMapId = parentMapping.getNestedResultMapId();  
                  ResultMap resultMap = configuration.getResultMap(nestedResultMapId);  
                  handleResultSet(rsw, resultMap, null, parentMapping);  
              }  
              rsw = getNextResultSet(stmt);  
              cleanUpAfterHandlingResultSet();  
              resultSetCount++;  
          }  
      }  
      return collapseSingleResultList(multipleResults);  
}  
```

