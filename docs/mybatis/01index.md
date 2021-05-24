# Mybatis 源码解析
## Pom依赖(pom.xml)
```xml
<dependencies>
<!-- mybatis -->
    <dependency>
      <groupId>org.mybatis</groupId>
      <artifactId>mybatis</artifactId>
      <version>3.5.6</version>
    </dependency>
    <!-- 其他依赖 -->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>log4j</groupId>
      <artifactId>log4j</artifactId>
      <version>1.2.17</version>
    </dependency>
    
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.35</version>
    </dependency>
    
    <dependency>
      <groupId>org.projectlombok</groupId>
      <artifactId>lombok</artifactId>
      <version>1.18.16</version>
    </dependency> 
</dependencies>

```
## jdbc.properties
```properties
jdbc.drive=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mybatis_learn?useUnicode=true
jdbc.username=root
jdbc.password= ---
```

## Mybatis-config.xml
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//ibatis.apache.org//DTD Config 3.0//EN"
        "http://ibatis.apache.org/dtd/ibatis-3-config.dtd">
<configuration>
    <properties resource="jdbc.properties"/>
    <settings>
        <setting name="cacheEnabled" value="true"/>
        <setting name="logImpl" value="LOG4J"/>
    </settings>
    <typeAliases>
        <typeAlias alias="Article" type="com.song.dao.ArticleDao"/>
    </typeAliases>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.drive}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="mapper/ArticleMapper.xml"/>
    </mappers>
</configuration>
```


## po dao mapper
```java
package com.song.po;

import lombok.Data;
import lombok.ToString;

import java.util.Date;

/**
 * @author SMD
 * @version 1.0
 * 2021/5/21 21:26
 */
@Data
@ToString
public class Article {
    private Integer id;
    private String title;
    private String author;
    private String content;
    private Date createTime;

}

package com.song.dao;

import com.song.po.Article;
import org.apache.ibatis.annotations.Param;

import java.util.List;

/**
 * @author SMD
 * @version 1.0
 * 2021/5/21 21:28
 */
public interface ArticleDao {

    List<Article> findByAuthorAndCreateTime(@Param("author") String author, @Param("createTime") String createTime);
}


```
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.song.dao.ArticleDao">

    <select id="findByAuthorAndCreateTime" resultType="com.song.po.Article">
        select id, title, author, content, create_time
        from article where author = #{author} and create_time = #{createTime}
    </select>
</mapper>

```


## 测试代码
```java
public class MybatisTest {
    private SqlSessionFactory sqlSessionFactory;

    // 准备工作,引入配置文件,创建sqlSessionFactory;
    @Before
    public void initFactory() throws IOException {
        String resource = "mybatis-config.xml";
        InputStream sourceStream = Resources.getResourceAsStream(resource);
        // 创建sqlSessionFactory
        sqlSessionFactory = new SqlSessionFactoryBuilder().build(sourceStream);
        sourceStream.close();
    }

    @Test
    public void testMybatis() {
        // 创建sqlSession
        SqlSession session = sqlSessionFactory.openSession();
        // 获取Dao
        ArticleDao mapper = session.getMapper(ArticleDao.class);
        // 执行sql
        List<Article> lists = mapper.findByAuthorAndCreateTime("smd", "2021-05-22");
        System.out.println(lists);
        // 提交并归还sqlSession
        session.commit();
        session.close();
    }

}

```
## log
```text
2021-05-22 12:04:06,830 [main] DEBUG [org.apache.ibatis.datasource.pooled.PooledDataSource] - PooledDataSource forcefully closed/removed all connections.
2021-05-22 12:04:06,970 [main] DEBUG [org.apache.ibatis.transaction.jdbc.JdbcTransaction] - Opening JDBC Connection
2021-05-22 12:04:07,340 [main] DEBUG [org.apache.ibatis.datasource.pooled.PooledDataSource] - Created connection 1782148126.
2021-05-22 12:04:07,340 [main] DEBUG [org.apache.ibatis.transaction.jdbc.JdbcTransaction] - Setting autocommit to false on JDBC Connection [com.mysql.jdbc.JDBC4Connection@6a396c1e]
2021-05-22 12:04:07,343 [main] DEBUG [com.song.dao.ArticleDao.findByAuthorAndCreateTime] - ==>  Preparing: select id, title, author, content, create_time from article where author = ? and create_time = ?
2021-05-22 12:04:07,365 [main] DEBUG [com.song.dao.ArticleDao.findByAuthorAndCreateTime] - ==> Parameters: smd(String), 2021-05-22(String)
2021-05-22 12:04:07,392 [main] DEBUG [com.song.dao.ArticleDao.findByAuthorAndCreateTime] - <==      Total: 1
[Article(id=1, title=mybatis_learn, author=smd, content=mybatis_learn_content, createTime=null)]
2021-05-22 12:04:07,392 [main] DEBUG [org.apache.ibatis.transaction.jdbc.JdbcTransaction] - Resetting autocommit to true on JDBC Connection [com.mysql.jdbc.JDBC4Connection@6a396c1e]
2021-05-22 12:04:07,392 [main] DEBUG [org.apache.ibatis.transaction.jdbc.JdbcTransaction] - Closing JDBC Connection [com.mysql.jdbc.JDBC4Connection@6a396c1e]
2021-05-22 12:04:07,392 [main] DEBUG [org.apache.ibatis.datasource.pooled.PooledDataSource] - Returned connection 1782148126 to pool.

```