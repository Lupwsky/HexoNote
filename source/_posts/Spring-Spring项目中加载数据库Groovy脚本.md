---
title: Spring-Spring项目中加载数据库Groovy脚本
date: 2019-04-19 23:16:56
categories: Spring
---

项目中需要报表统计功能, 由于公司发版管理严格, 发布版本需要走申请流程, 报表的统计有时需要根据条业务进行一些逻辑的更改, 为了不需要更改一点点逻辑就要去申请发版流程, 决定将 Groovy 脚本的内容保存在数据库中, 每次更新后从数据库中加载最新 Groovy 脚本, 再热加载到 JVM 中, 由于是基于 SpringBoot 的项目, 在加载后手动将生成的 bean 加入容器中进行管理, 实现步骤简单记录如下 (主要是自己笔记, 大概寄一个思路, 不是记录得很详细)

# 添加依赖

```xml
<dependency>
    <groupId>org.codehaus.groovy</groupId>
    <artifactId>groovy-all</artifactId>
    <version>2.4.13</version>
</dependency>
```

<!-- more -->

# 定义接口

定义一个接口, Groovy 脚本实现这个接口

```java
public interface BaseGroovySpot {
    String test();
    String test2();
}
```

# 定义脚本

实现 BaseGroovySpot 接口, test 方法像数据库中插入一条数据, test2 方法读取数据, 编写好后, 将内容复制到数据库中保存

```groovy
package com.lupw.guava.groovy

import com.lupw.guava.datasource.db.mapper.UserMapper
import groovy.sql.Sql
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.beans.factory.annotation.Qualifier
import org.springframework.stereotype.Component

import javax.sql.DataSource

/**
 * @author pwlu 2019/4/19
 */
@Component
class GroovyGroovySpot implements BaseGroovySpot {

    @Autowired
    private UserMapper db0UserMapper;

    @Autowired
    @Qualifier("db0")
    private final DataSource dataSource0;

    @Override
    public String test() {
        String result
        int i = db0UserMapper.addUserInfo("lpw", "test")
        if (i > 0) {
            result = "添加成功"
            println("添加成功")
        } else {
            result = "添加失败"
            println("添加失败")
        }
        return result
    }

    @Override
    public String test2() {
        def sql = new Sql(dataSource0)
        sql.eachRow("SELECT id, name, password FROM user_info WHERE id >= '0'") { 
          row -> println("id = " + row[0] +
                         ", name = " + row[1] +
                         ", password = " + row[2])
        }
        return "success"
    }
}
```

# 测试代码

先从调用 groovyAddBean 方法从数据库中读取 Groovy 脚本, 生成 bean 后委托给 Spring 容器更改, 接着调用 groovyTest 方法, 调用 Groovy 脚本中实现的方法 

```java
@Slf4j
@RestController
public class GroovyTestController {

    private final UserMapper db0UserMapper;
    private final ApplicationContext applicationContext;

    @Autowired
    public GroovyTestController(UserMapper db0UserMapper,
                                ApplicationContext applicationContext) {
        this.db0UserMapper = db0UserMapper;
        this.applicationContext = applicationContext;
    }

    @GetMapping(value = "/groovy/test")
    public void groovyTest() {
        BaseGroovySpot baseGroovySpot 
          = applicationContext.getBean("groovyBean", BaseGroovySpot.class);
        log.info("content = {}", baseGroovySpot.test());
        log.info("content = {}", baseGroovySpot.test2());
    }

    @GetMapping(value = "/groovy/add/bean")
    public void groovyAddBean() {
        String content = db0UserMapper.getGroovyContent("1");

        Class clazz = new GroovyClassLoader().parseClass(content);
        BeanDefinitionBuilder beanDefinitionBuilder 
          = BeanDefinitionBuilder.genericBeanDefinition(clazz);
        BeanDefinition beanDefinition = beanDefinitionBuilder.getRawBeanDefinition();
        beanDefinition.setScope(BeanDefinition.SCOPE_SINGLETON);

        AutowireCapableBeanFactory autowireCapableBeanFactory 
          = applicationContext.getAutowireCapableBeanFactory();
        autowireCapableBeanFactory
          .applyBeanPostProcessorsAfterInitialization(beanDefinition, "groovyBean");

        BeanDefinitionRegistry beanRegistry 
          = (BeanDefinitionRegistry) autowireCapableBeanFactory;
        if (beanRegistry.containsBeanDefinition("groovyBean")) {
            beanRegistry.removeBeanDefinition("groovyBean");
        }
        beanRegistry.registerBeanDefinition("groovyBean", beanDefinition);
    }
}
```

更改数据库中的 Groovy 脚本的内容, 再进行测试, 即使没有重启应用, 可以看到内容发生了变化, 其他相关的代码如下

```xml
<insert id="addUserInfo" parameterType="java.lang.String">
    INSERT INTO user_info(`name`, password) VALUES(#{name}, #{password})
</insert>

<select id="getGroovyContent" parameterType="java.lang.String" 
        resultType="java.lang.String">
    SELECT content FROM groovy WHERE id = #{id} AND is_delete = '0'
</select>
```

为了能让修改的 Groovy  实时生效, 这里需要实现一个定时任务定时的从数据库中读取 Groovy 的最新内容, 根据某个标志位判断内容是否有更改, 如果有更改将最新的 Groovy 加载, 并委托给 Spring 容器管理, 当然 Spring 已经有相关功能的实现, 下一节会记录 Spring 的实现方式

# 测试结果

运行测试了多次, 其中的一次输出结果如下

```text
c.l.g.datasource.MultiRoutingDataSource  : 当前数据源为空, 使用默认的数据源
添加成功
c.l.guava.groovy.GroovyTestController    : content = 添加成功
id = 1, name = lpw, password = test
id = 2, name = lpw, password = test
id = 3, name = lpw, password = test
id = 4, name = lpw, password = test
id = 5, name = lpw, password = test
id = 6, name = lpw, password = test
id = 7, name = lpw, password = test
id = 8, name = lpw, password = test
c.l.guava.groovy.GroovyTestController    : content = success
```
