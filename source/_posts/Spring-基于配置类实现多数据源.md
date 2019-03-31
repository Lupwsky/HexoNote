---
title: Spring-基于配置类实现多数据源
date: 2019-02-20 23:07:43
categories: Spring
---

这篇笔记记录如何配置多数据源

# 添加依赖

使用 alibaba 的 druid 来管理数据库连接池

```xml
<!-- mysql-connector -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.11</version>
</dependency>

<!-- alibaba druid -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.1.2</version>
</dependency>

<!-- mybatis -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>1.3.0</version>
</dependency>
```

<!-- more -->

# 配置数据源

我这里的多数据源仅为了测试在数据库建立三个分库, 并将 db0 作为默认的库

```ini
spring.datasource.db0.name=db0
spring.datasource.db0.driver-class=com.mysql.cj.jdbc.Driver
spring.datasource.db0.url=jdbc:mysql://localhost:3306/db0?characterEncoding=utf8&serverTimezone=UTC&useSSL=false
spring.datasource.db0.username=root
spring.datasource.db0.password=123456
spring.datasource.db0.initial-size=1
spring.datasource.db0.min-idle=1
spring.datasource.db0.max-active=20
spring.datasource.db0.max-wait=1000

spring.datasource.db1.name=db1
spring.datasource.db1.driver-class=com.mysql.cj.jdbc.Driver
spring.datasource.db1.url=jdbc:mysql://localhost:3306/db1?characterEncoding=utf8&serverTimezone=UTC&useSSL=false
spring.datasource.db1.username=root
spring.datasource.db1.password=123456
spring.datasource.db1.initial-size=1
spring.datasource.db1.min-idle=1
spring.datasource.db1.max-active=20
spring.datasource.db1.max-wait=1000

spring.datasource.db2.name=db2
spring.datasource.db2.driver-class=com.mysql.cj.jdbc.Driver
spring.datasource.db2.url=jdbc:mysql://localhost:3306/db2?characterEncoding=utf8&serverTimezone=UTC&useSSL=false
spring.datasource.db2.username=root
spring.datasource.db2.password=123456
spring.datasource.db2.initial-size=1
spring.datasource.db2.min-idle=1
spring.datasource.db2.max-active=20
spring.datasource.db2.max-wait=1000
```

# 读取配置文件配置的数据源信息到实体类

创建 Db0DataSourceProperties, Db1DataSourceProperties 和 Db2DataSourceProperties 三个实体类, 使用 @ConfigurationProperties 注解, 配置文件配置的数据源信息，读取并自动封装成实体类, 并将这个实体类配置使用 @Component 注解配置成 bean, `如果这里已经将实体类配置成 bean 了, 那么在使用的使用就不需要在到类的上面添加 @EnableConfigurationProperties(Db0DataSourceProperties.class) 注解了`

```java
@Data
@Component(value = "db0DataSourceProperties")
@ConfigurationProperties(prefix = "spring.datasource.db0")
public class Db0DataSourceProperties {
    private String name;
    private String driverClass;
    private String url;
    private String username;
    private String password;
    private int initialSize;
    private int minIdle;
    private int maxActive;
    private int maxWait;
}

@Data
@Component(value = "db1DataSourceProperties")
@ConfigurationProperties(prefix = "spring.datasource.db1")
public class Db1DataSourceProperties {
    private String name;
    private String driverClass;
    private String url;
    private String username;
    private String password;
    private int initialSize;
    private int minIdle;
    private int maxActive;
    private int maxWait;
}

@Data
@Component(value = "db2DataSourceProperties")
@ConfigurationProperties(prefix = "spring.datasource.db2")
public class Db2DataSourceProperties {
    private String name;
    private String driverClass;
    private String url;
    private String username;
    private String password;
    private int initialSize;
    private int minIdle;
    private int maxActive;
    private int maxWait;
}
```

# XML映射文件

```xml
<!-- db0 -->
<mapper namespace="com.lupw.guava.datasource.db0.mapper.UserMapper">
    <insert id="addUserInfo" parameterType="java.lang.String">
        INSERT INTO user_info(`name`, password) VALUES(#{name}, #{password})
    </insert>
</mapper>

<!-- db1 -->
<mapper namespace="com.lupw.guava.datasource.db1.mapper.UserMapper">
    <insert id="addUserInfo" parameterType="java.lang.String">
        INSERT INTO user_info(`name`, password) VALUES(#{name}, #{password})
    </insert>
</mapper>

<!-- db2 -->
<mapper namespace="com.lupw.guava.datasource.db2.mapper.UserMapper">
    <insert id="addUserInfo" parameterType="java.lang.String">
        INSERT INTO user_info(`name`, password) VALUES(#{name}, #{password})
    </insert>
</mapper>
```

# DAO

```java
// db0
@Repository(value = "db0UserMapper")
public interface UserMapper {
    int addUserInfo(@Param("name") String name, @Param("password") String password);
}

// db1
@Repository(value = "db1UserMapper")
public interface UserMapper {
    int addUserInfo(@Param("name") String name, @Param("password") String password);
}

// db2
@Repository(value = "db2UserMapper")
public interface UserMapper {
    int addUserInfo(@Param("name") String name, @Param("password") String password);
}
```

# 配置多数据源和使用

以配置 db0 为例, 多数据源配置的时候注意, 必须要有一个主数据源, 需要在对应的 bean 上添加 @Primary 注解, 如下:

```java
@Configuration
@MapperScan(basePackages = Db0DataSourceConfig.MAPPER_SCAN_PACKAGE, 
        sqlSessionFactoryRef = Db0DataSourceConfig.DB0_SQL_SESSION_FACTORY_BEAN_NAME)
public class Db0DataSourceConfig {

    private final static String DB0_BEAN_NAME = "db0";
    final static String DB0_SQL_SESSION_FACTORY_BEAN_NAME = "db0Factory";

    /**
     * 关于 basePackages 扫描的包和 xml 的路径, 配置多个数据源配置时是可以相同的, 为了方便后续管理, 我一般都是分开的
     */
    final static String MAPPER_SCAN_PACKAGE = "com.lupw.guava.datasource.db0.mapper";
    private final static String DB0_MAPPER_LOCATIONS = "classpath:db0/*.xml";

    @Primary
    @Bean(name = DB0_BEAN_NAME)
    public DataSource getDataSource(Db0DataSourceProperties db0DataSourceProperties) {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(db0DataSourceProperties.getDriverClass());
        dataSource.setUrl(db0DataSourceProperties.getUrl());
        dataSource.setUsername(db0DataSourceProperties.getUsername());
        dataSource.setPassword(db0DataSourceProperties.getPassword());
        dataSource.setInitialSize(db0DataSourceProperties.getInitialSize());
        dataSource.setMinIdle(db0DataSourceProperties.getMinIdle());
        dataSource.setMaxActive(db0DataSourceProperties.getMaxActive());
        dataSource.setMaxWait(db0DataSourceProperties.getMaxWait());
        return dataSource;
    }

    @Primary
    @Bean(name = DB0_SQL_SESSION_FACTORY_BEAN_NAME)
    public SqlSessionFactory getDataSourceFactory(@Qualifier(DB0_BEAN_NAME) DataSource dataSource) throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(dataSource);
        sessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources(DB0_MAPPER_LOCATIONS));
        return sessionFactoryBean.getObject();
    }
}
```

多个数据源按照上面的方式就可以配置完成了, 分别注入对应的 DAO 就可以使用了

```java
@RestController
public class DbTestController {

    private final UserMapper db0UserMapper;
    private final com.lupw.guava.datasource.db1.mapper.UserMapper db1UserMapper;
    private final com.lupw.guava.datasource.db2.mapper.UserMapper db2UserMapper;

    @Autowired
    public DbTestController(UserMapper db0UserMapper,
                            com.lupw.guava.datasource.db1.mapper.UserMapper db1UserMapper,
                            com.lupw.guava.datasource.db2.mapper.UserMapper db2UserMapper) {
        this.db0UserMapper = db0UserMapper;
        this.db1UserMapper = db1UserMapper;
        this.db2UserMapper = db2UserMapper;
    }


    @GetMapping(value = "/db/test")
    public void dbTestController() {
        db0UserMapper.addUserInfo("lpw-db0", "123456");
        db1UserMapper.addUserInfo("lpw-db1", "123456");
        db2UserMapper.addUserInfo("lpw-db2", "123456");
    }
}
```