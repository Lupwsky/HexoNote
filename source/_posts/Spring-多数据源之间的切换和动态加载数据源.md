---
title: Spring-多数据源之间的切换和动态加载数据源
date: 2019-02-20 23:33:58
categories: Spring
---

实现了多数据源配置后, 在实际使用的时候会有一些问题, 一个是数据源的事物, 例如在一个方法里面, 只对其中一个库进行更改, 在方法上添加 @Transactional 注解并指定事物管理器, 可以使用正常使用相应数据源的事物, 但是如果需要在同一个方法里面修改多个库中的表,  @Transactional 注解就会出现问题了, 另外多个数据源只是分库, 就像上一篇笔记例子里面的只是库名不同, 表结构相同时, 如果也像上一篇笔记例子里面这样去使用也是比较繁琐的, 需要一种更加灵活的方式来实现, Spring 提供的 AbstractRoutingDataSource 就实现动态数据源切换, 它可以充当 DataSource 的路由中介, 能有在运行时, 根据某种 key 值来动态切换到真正的 DataSource 上

# 保存线程当前数据源的 KEY

```java
@Slf4j
public class MultiRoutingDataSourceContext {
    private final static ThreadLocal<String> DB_KEY = new ThreadLocal<>();

    private MultiRoutingDataSourceContext() {
    }

    public static String getCurrentDbKey() {
        return DB_KEY.get();
    }

    public static void setCurrentDbKey(String tenant) {
        DB_KEY.set(tenant);
    }

    public static void clear() {
        DB_KEY.remove();
    }
}
```

<!-- more -->

# 继承 AbstractRoutingDataSource 类

```java
@Slf4j
public class MultiRoutingDataSource extends AbstractRoutingDataSource {

    /**
     * 见 AbstractRoutingDataSource 源码, 会自动的根据名字获取 DataSource 的 Bean, 并切换数据源
     *
     * @return
     */
    @Override
    protected Object determineCurrentLookupKey() {
        String currentDbKey = MultiRoutingDataSourceContext.getCurrentDbKey();
        if (StringUtils.isEmpty(currentDbKey)) {
            log.error("当前数据源为空, 使用默认的数据源");
        } else {
            log.info("当前数据源 = {}, 使用默认的数据源", currentDbKey);
        }
        return currentDbKey;
    }
}
```

# 配置数据源和使用

```java
// 多个 basePackages = {"", ""}
@Configuration
@EnableConfigurationProperties({Db0DataSourceProperties.class, Db1DataSourceProperties.class, Db2DataSourceProperties.class})
@MapperScan(basePackages = "com.lupw.guava.datasource.db.mapper", sqlSessionFactoryRef = "dbFactory")
public class MultiDataSourceConfig {
    private final Db0DataSourceProperties db0DataSourceProperties;
    private final Db1DataSourceProperties db1DataSourceProperties;
    private final Db2DataSourceProperties db2DataSourceProperties;

    @Autowired
    public MultiDataSourceConfig(Db0DataSourceProperties db0DataSourceProperties,
                                 Db1DataSourceProperties db1DataSourceProperties,
                                 Db2DataSourceProperties db2DataSourceProperties) {
        this.db0DataSourceProperties = db0DataSourceProperties;
        this.db1DataSourceProperties = db1DataSourceProperties;
        this.db2DataSourceProperties = db2DataSourceProperties;
    }

    @Bean(name = "db0")
    public DataSource getDb0DataSource() {
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

    @Bean(name = "db1")
    public DataSource getDb1DataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(db1DataSourceProperties.getDriverClass());
        dataSource.setUrl(db1DataSourceProperties.getUrl());
        dataSource.setUsername(db1DataSourceProperties.getUsername());
        dataSource.setPassword(db1DataSourceProperties.getPassword());
        dataSource.setInitialSize(db1DataSourceProperties.getInitialSize());
        dataSource.setMinIdle(db1DataSourceProperties.getMinIdle());
        dataSource.setMaxActive(db1DataSourceProperties.getMaxActive());
        dataSource.setMaxWait(db1DataSourceProperties.getMaxWait());
        return dataSource;
    }

    @Bean(name = "db2")
    public DataSource getDb2DataSource() {
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setDriverClassName(db2DataSourceProperties.getDriverClass());
        dataSource.setUrl(db2DataSourceProperties.getUrl());
        dataSource.setUsername(db2DataSourceProperties.getUsername());
        dataSource.setPassword(db2DataSourceProperties.getPassword());
        dataSource.setInitialSize(db2DataSourceProperties.getInitialSize());
        dataSource.setMinIdle(db2DataSourceProperties.getMinIdle());
        dataSource.setMaxActive(db2DataSourceProperties.getMaxActive());
        dataSource.setMaxWait(db2DataSourceProperties.getMaxWait());
        return dataSource;
    }

    @Bean(name = "multiRoutingDataSource")
    public DataSource getMultiRoutingDataSource() {
        MultiRoutingDataSource dataSource = new MultiRoutingDataSource();
        dataSource.setDefaultTargetDataSource(getDb0DataSource());
        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put("db0", getDb0DataSource());
        dataSourceMap.put("db1", getDb1DataSource());
        dataSourceMap.put("db2", getDb2DataSource());
        dataSource.setTargetDataSources(dataSourceMap);
        return dataSource;
    }

    @Bean(name = "dbFactory")
    public SqlSessionFactory getDataSourceFactory() throws Exception {
        SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        sessionFactoryBean.setDataSource(getMultiRoutingDataSource());
        sessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*/*.xml"));
        return sessionFactoryBean.getObject();
    }

    @Bean(name = "multiRoutingTransaction")
    public PlatformTransactionManager platformTransactionManager() {
        return new DataSourceTransactionManager(getMultiRoutingDataSource());
    }
}
```

在使用的使用通过 `MultiRoutingDataSourceContext.setCurrentDbKey(String dbKey)` 就能切换库了, 当然也可以使用 AOP 功能使用注解来完成切换, 由于实际项目中, 在一个方法里面切换数据源时常有的事, 使用注解在这种情况下个人认为不太灵活, 使用示例如下:

```java
@RestController
public class DbTestController {
    private final UserMapper db0UserMapper;

    @Autowired
    public DbTestController(UserMapper db0UserMapper) {
        this.db0UserMapper = db0UserMapper;
    }

    @GetMapping(value = "/db/test")
    @Transactional(rollbackFor = Exception.class)
    public void dbTestController() {
        // 默认使用 db0
        db0UserMapper.addUserInfo("lpw-db0", "123456");

        // 切换到数据源 db1
        MultiRoutingDataSourceContext.setCurrentDbKey("db1");
        db0UserMapper.addUserInfo("lpw-db1", "123456");

        // 切换到数据源 db2
        MultiRoutingDataSourceContext.setCurrentDbKey("db2");
        db0UserMapper.addUserInfo("lpw-db2", "123456");

        // 测试事物
        throw new RuntimeException();
    }
}
```

# 关于动态加载数据源

在实际项目中, 数据库的连接信息都放在数据库中, 一开始无法获取全部的数据源的配置信息, 需要进行动态的加载或者删除数据源, 实际项目中参考了以下文章实现:

* [Spring 配置动态数据源 (动态添加免重启)](https://my.oschina.net/u/1189928/blog/1650013)
* [Spring 动态创建, 加载, 使用多数据源](https://www.jianshu.com/p/c57772c8b802)

# 关于分布式事物

* [再有人问你分布式事务, 把这篇扔给他](https://juejin.im/post/5b5a0bf9f265da0f6523913b)
* [聊聊分布式事务, 再说说解决方案](https://www.cnblogs.com/savorboard/p/distributed-system-transaction-consistency.html)
* [分布式事务 CAP 理解论证和解决方案](https://blog.csdn.net/weixin_40533111/article/details/85069536)
* [分布式事务管理之 JTA 与链式事务](https://mp.weixin.qq.com/s?__biz=MzI4Njc5NjM1NQ==&mid=2247487469&idx=1&sn=79949284b766731c58703374cdeb846c&chksm=ebd630c1dca1b9d75dac461a14660e5ae379183ca37ddb809258f5ba1dbd210a850410a43321&mpshare=1&scene=1&srcid=0128NSYfQuGGtSIBk0uGv1dh#rd)
* [Mysql 和 Druid 结合 Atomikos 实现分布式事务](https://www.jianshu.com/p/0dde641295af)
