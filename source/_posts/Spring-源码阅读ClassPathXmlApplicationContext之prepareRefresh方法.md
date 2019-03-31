---
title: Spring-源码阅读ClassPathXmlApplicationContext之prepareRefresh方法
date: 2019-03-25 22:36:03
categories:  Spring
---

# ClassPathXmlApplicationContext 的简单用法

```java
ApplicationContext applicationContext = new ClassPathXmlApplicationContext("beans/UserBeans.xml");
UserInfo userInfo = applicationContext.getBean("userInfo", UserInfo.class);
log.info("name = {}, email = {}", userInfo.getName(), userInfo.getEmail());
```

<!-- more -->

从 ClassPathXmlApplicationContext 的构造方法开始对源码进行分析, 进入跟踪构造法方法, 主要调用了 setConfigLocations() 和 refresh() 方法, setConfigLocations() 方法用于配置文件路径的解析, 如果配置文件路径里面有 ${KEY} 表示的系统属性会在 resolvePath() 方法中被替换掉, 并且支持 SPEL 语法, 源码如下:

```java
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, @Nullable ApplicationContext parent) throws BeansException {
    super(parent);
    this.setConfigLocations(configLocations);
    if (refresh) {
        this.refresh();
    }
}

public void setConfigLocations(@Nullable String... locations) {
    if (locations != null) {
        Assert.noNullElements(locations, "Config locations must not be null");
        this.configLocations = new String[locations.length];
        for (int i = 0; i < locations.length; i++) {
            this.configLocations[i] = resolvePath(locations[i]).trim();
        }
    }
    else {
        this.configLocations = null;
    }
}
```

# prepareRefresh 方法

refresh() 方法描述了整个 ClassPathXmlApplicationContext 初始化的过程, 调用的第一个方法就是 prepareRefresh() 方法, 主要用于初始化前的有些准备工作, 其源码如下:

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
    }
}

protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    if (logger.isDebugEnabled()) {
        if (logger.isTraceEnabled()) {
            logger.trace("Refreshing " + this);
        } else {
            logger.debug("Refreshing " + getDisplayName());
        }
    }

    initPropertySources();
    getEnvironment().validateRequiredProperties();

    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

首先调用了 initPropertySources() 方法, 用于初始化之前的准备工作, 一般用于对环境变量进行检测或者对系统属性进行设置和检测, 这个方法在源码中时没有任何实现的, 用户可以重写 initPropertySources 方法来实现自己的验证属性的逻辑, 是 Spring 开放式结构的体现, 源码如下:

```java
protected void initPropertySources() {
    // For subclasses: do nothing by default.
}
```

接着调用了 validateRequiredProperties() 方法, 用于对需要验证系统属性进行验证, 其源码实现如下, 可以知道只是简单的判断属性的值是否为 null, 如果需要更加复杂的验证还是自己去手动在 initPropertySources() 方法中去实现, 源码如下:

```java
@Override
public void validateRequiredProperties() {
    MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
    for (String key : this.requiredProperties) {
        if (this.getProperty(key) == null) {
            ex.addMissingRequiredProperty(key);
        }
    }
    if (!ex.getMissingRequiredProperties().isEmpty()) {
        throw ex;
    }
}
```

# 重写 initPropertySources() 方法

使用 ClassPathXmlApplicationContext 时, 如果需要重写 initPropertySources() 方法需要自定义一个类继承 ClassPathXmlApplicationContext, 然后重写 initPropertySources() 方法, 使用示例如下, 这里实现一个简单的系统属性检测:

```java
@Slf4j
public class DefineClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {

    DefineClassPathXmlApplicationContext(String... configLocations) throws BeansException {
        super(configLocations);
    }

    @Override
    protected void initPropertySources() {
        log.info("Spring 初始化之前的准备工作, 一般用于对环境变量进行检测或者对系统属性进行设置和检测");

        // 系统的相关属性, JVM 启动的时候使用 -D 参数可以添加自定义的属性值
        Map<String, Object> jvmProperties = this.getEnvironment().getSystemProperties();
        jvmProperties.forEach((key, value) -> log.info("key = {}, value = {}", key, value));

        // 系统的环境变量
        Map<String, Object> systemProperties = this.getEnvironment().getSystemEnvironment();
        systemProperties.forEach((key, value) -> log.info("key = {}, value = {}", key, value));

        // 自己手动检测, 检测不存在手动抛出异常
        String javaHome = "JAVA_HOME";
        if (Objects.equals(null, systemProperties.get(javaHome)) || Objects.equals("", systemProperties.get(javaHome))) {
            throw new RuntimeException("JAVA_HOME 没有配置");
        }

        // 调用 setRequiredProperties() 方法后, 通过 validateRequiredProperties() 方法来检测
        // 如果该属性不存在则抛出 MissingRequiredPropertiesException 异常
        // 假设 JAVA_HOME 属性不存在抛出的异常提示 = MissingRequiredPropertiesException: The following properties were declared as required but could not be resolved: [JAVA_HOME]
        this.getEnvironment().setRequiredProperties("JAVA_HOME");
    }
}
```

使用方法和普通的 ClassPathXmlApplicationContext 方法一样, 如下:

```java
ApplicationContext applicationContext = new DefineClassPathXmlApplicationContext("beans/UserBeans.xml");
UserInfo userInfo = applicationContext.getBean("userInfo", UserInfo.class);
log.info("name = {}, email = {}", userInfo.getName(), userInfo.getEmail());
```

# SpringBoot 项目中重写 initPropertySources() 方法

SpringBoot 项目中是 WEB 项目, 需要继承的是 AnnotationConfigServletWebServerApplicationContext, 如下:

```java
@Slf4j
public class DefineAnnotationConfigServletWebServerApplicationContext extends AnnotationConfigServletWebServerApplicationContext {
    @Override
    protected void initPropertySources() {
        // 可以对不同环境下的属进行检测或者设置不同环境下所需属性
        // 观察日志, 可以看到 initPropertySources 方法被执行了两次, 是因为除了 AbstractApplicationContext 类中有调用
        // 在 ServletWebServerApplicationContext 类的 onRefresh 中执行 createWebServer 方法时也会调用一次 initPropertySources 方法
        log.info("[系统配置属性检测] 项目启动, 开始检测");
        String[] activeProfiles = this.getEnvironment().getActiveProfiles();
        if (Arrays.stream(activeProfiles).allMatch(activeProfile -> Objects.equals("dev", activeProfile))) {
            log.info("[系统配置属性检测] 当前是 DEV 环境, 对 JAVA_HOME 属性进行检测");
            this.getEnvironment().setRequiredProperties("JAVA_HOME");
        } else {
            log.info("[系统配置属性检测] 当前非 DEV 环境, 对 JAVA_HOME1 属性进行检测");
            this.getEnvironment().setRequiredProperties("JAVA_HOME1");
        }
        log.info("[系统配置属性检测] 通过检测");
    }
}
```

然后在启动项中设置 DefineAnnotationConfigServletWebServerApplicationContext 类型为创建容器的类型即可, 如下:

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(Application.class);
        springApplication.setApplicationContextClass(DefineAnnotationConfigServletWebServerApplicationContext.class);
        springApplication.run(args);
    }
}
```