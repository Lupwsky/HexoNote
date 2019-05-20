---
title: Spring-MyBatisPlus多数据源配置和基础使用
date: 2019-05-20 22:46:17
categories: Spring
---

MyBatis Plus 的使用方法, 基础就不取解释和记录了, 官方文档已经写得很详细了, 这些可以去参考[官方文档](https://baomidou.gitee.io/mybatis-plus-doc/#/page-plugin)

# 配置数据源

使用 Mybatis Plus 时配置数据源和使用 MyBatis 配置数据源基本上是一样的, 不同的是要将 MyBatis 的 SqlSessionFactory 替换成 MyBatisPlus 的 SqlSessionFactory, MyBatis 多数据源的详细配置可以参考配置之前多数据源这一节笔记, 使用 Mybatis Plus 时配置数据源配置类如下:

```java
@Configuration
@EnableConfigurationProperties({Db0DataSourceProperties.class, Db1DataSourceProperties.class, Db2DataSourceProperties.class})
@MapperScan(basePackages = "com.lupw.guava.datasource.db.mapper", sqlSessionFactoryRef = "dbFactory")
public class MultiDataSourceConfig {
    // 其他数据源的配置...

    @Bean(name = "dbFactory")
    public SqlSessionFactory getDataSourceFactory(PaginationInterceptor paginationInterceptor) throws Exception {
        // SqlSessionFactoryBean sessionFactoryBean = new SqlSessionFactoryBean();
        // sessionFactoryBean.setDataSource(getMultiRoutingDataSource());
        // sessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*/*.xml"));
        // return sessionFactoryBean.getObject();

        // MyBatis 的 SqlSessionFactory 替换成 MyBatisPlus 的 SqlSessionFactory
        MybatisSqlSessionFactoryBean sqlSessionFactoryBean =  new MybatisSqlSessionFactoryBean();
        sqlSessionFactoryBean.setDataSource(getMultiRoutingDataSource());
        sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*/*.xml"));
        return sqlSessionFactoryBean.getObject();
    }
}
```

<!-- more -->

# 使用方法

BaseMapper 里面定义了一些常用的 CRUD 方法, 创建一个 Mapper 接口继承它, 就能在启动的时候自动注入基本的 CRUD 操作, 直接面向对象操作, 支持的操作如下:

```java
public interface BaseMapper<T> extends Mapper<T> {
    int insert(T entity);
    int deleteById(Serializable id);
    int deleteByMap(@Param("cm") Map<String, Object> columnMap);
    int delete(@Param("ew") Wrapper<T> wrapper);
    int deleteBatchIds(@Param("coll") Collection<? extends Serializable> idList);
    int updateById(@Param("et") T entity);
    int update(@Param("et") T entity, @Param("ew") Wrapper<T> updateWrapper);
    T selectById(Serializable id);
    List<T> selectBatchIds(@Param("coll") Collection<? extends Serializable> idList);
    List<T> selectByMap(@Param("cm") Map<String, Object> columnMap);
    T selectOne(@Param("ew") Wrapper<T> queryWrapper);
    Integer selectCount(@Param("ew") Wrapper<T> queryWrapper);
    List<T> selectList(@Param("ew") Wrapper<T> queryWrapper);
    List<Map<String, Object>> selectMaps(@Param("ew") Wrapper<T> queryWrapper);
    List<Object> selectObjs(@Param("ew") Wrapper<T> queryWrapper);
    IPage<T> selectPage(IPage<T> page, @Param("ew") Wrapper<T> queryWrapper);
    IPage<Map<String, Object>> selectMapsPage(IPage<T> page, @Param("ew") Wrapper<T> queryWrapper);
}
```

创建 Mapper 接口继承 BaseMapper

```java
@Repository
public interface UserMapper extends BaseMapper<UserInfo> {
}
```

使用方法如下:

```java
// 查询全部
List<UserInfo> userInfoList = userMapper.selectList(null);
log.info("{}", userInfoList.toString());

// new QueryWrapper<UserInfo>().ge("id", 2).le("id", 5);
List<UserInfo> userInfoList1 = userMapper.selectList(new QueryWrapper<UserInfo>().lambda()
        .ge(UserInfo::getId, 2)
        .le(UserInfo::getId, 5));
log.info("{}", userInfoList1.toString());
```

可以看见对于一些 CRUD 是非常方便的, 不需要再去写 XML

# 分页

分页需要在创建 MybatisSqlSessionFactoryBean 时设置分页插件, 使用 IPage 才能正确的查询出分页信息, 配置如下:

```java
@Bean(name = "dbFactory")
public SqlSessionFactory getDataSourceFactory(PaginationInterceptor paginationInterceptor) throws Exception {
    // MyBatis 的 SqlSessionFactory 替换成 MyBatisPlus 的 SqlSessionFactory
    MybatisSqlSessionFactoryBean sqlSessionFactoryBean =  new MybatisSqlSessionFactoryBean();
    sqlSessionFactoryBean.setDataSource(getMultiRoutingDataSource());
    sqlSessionFactoryBean.setMapperLocations(new PathMatchingResourcePatternResolver().getResources("classpath:mapper/*/*.xml"));

    // 设置分页插件, 其他的插件也是一样的使用的时候添加到列表一起设置即可
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.add(paginationInterceptor);
    sqlSessionFactoryBean.setPlugins(interceptors.toArray(new Interceptor[0]));
    return sqlSessionFactoryBean.getObject();
}
```

使用方法:

```java
// 分页
int pageNum = 1, pageSize = 2;
IPage<UserInfo> userInfoIPage = userMapper.selectPage(new Page<>(pageNum, pageSize), null);
log.info("{}", JSONObject.toJSONString(userInfoIPage));
```

# XML 方式

Mybatis Plus 可以和 MyBatis 一样, 使用 XML 的方式写 SQL, 也支持分页查询, 通常复杂查询都会选择自己手写 SQL, 使用示例如下:

```java
@Repository(value = "db0UserMapper")
public interface UserMapper extends BaseMapper<UserInfo> {
    // 分页查询
    IPage<UserInfo> getUserInfoList(Page page, @Param("id") String id);

    int addUserInfo(@Param("name") String name, @Param("password") String password);
}
```

XML 文件内容如下 :

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd" >
<mapper namespace="com.lupw.guava.datasource.db.mapper.UserMapper">
    <insert id="addUserInfo" parameterType="java.lang.String">
        INSERT INTO user_info(`name`, password) VALUES(#{name}, #{password})
    </insert>

    <select id="getUserInfoList" parameterType="java.lang.String" resultType="com.lupw.guava.datasource.db.model.UserInfo">
        SELECT * FROM user_info WHERE id > #{id}
    </select>
</mapper>
```

示例代码:

```java
// 分页, XML 方式
IPage<UserInfo> userInfoIPage1 = userMapper.getUserInfoList(new Page<>(pageNum, 5), "0");
log.info("{}", JSONObject.toJSONString(userInfoIPage1));
```

# 实体类如何和表对应

上面 UserMapper 继承 `BaseMapper<UserInfo>`, 实体类如何和表对应起来的呢, 只需要在实体类中使用 @TableName 就能和相应的表对应起来了, 如下:

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@TableName("user_info")
public class UserInfo {
    long id;
    String name;
    String password;
}
```

详细可参考[官方文档-通用 CRUD 注解说明](https://baomidou.gitee.io/mybatis-plus-doc/#/generic-crud)