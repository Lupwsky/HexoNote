---
title: Spring-源码阅读ClassPathXmlApplicationContext之prepareBeanFactory方法
date: 2019-03-31 12:40:49
categories:  Spring
---

prepareBeanFactory 方法是 ClassPathXmlApplicationContext 初始化时调用 refresh 方法里面调用的第三个方法, refresh 方法部分源码如下:

```java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        prepareRefresh();
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        prepareBeanFactory(beanFactory);
        // ......
    }
}
```

<!-- more -->

# prepareBeanFactory 源码

prepareBeanFactory 用于对容器进行相关的设置, 并配置其标准的特性, 源码如下:

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置类加载器, 用于加载 Bean 使用
    beanFactory.setBeanClassLoader(getClassLoader());

    // 设置 Bean 表达式解析器, 使在 XML 配置文件中支持 SpEL 语法
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));

    // 设置默认的属性编辑器
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // Bean 处理器
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

    // 设置需要忽略的依赖
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 设置依赖
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // Register early post-processor for detecting inner beans as ApplicationListeners.
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // Detect a LoadTimeWeaver and prepare for weaving, if found.
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 像容器种注册和环境变量和属性变量相关的 Bean
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }

    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
    }

    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
    }
}
```

# 设置 Bean 处理器

首先了解 Aware 接口, Spring 中提供了一些 Aware 接口结尾的接口, 如常见的 ApplicationContextAware, BeanFactoryAware, BeanNameAware, ServletContextAware, ResourceLoaderAware, MessageSourceAware, ApplicationEventPublisherAware 等, Spring 实现的依赖注入时也尽量让容器和 Bean 解耦, 即 Bean 对容器的存在是无感知的, 但是有时候在使用 Bean 的时候是必须要获取 Spring 里面的一些资源 (对 Bean 而言是外部的资源), 例如 Bean 在容器里的名称, 这个时候就需要通过 Spring 提供的这些以 Aware 结尾的接口可以让 Bean 感知到容器的存在, 来实现对外部资源的获取, 下面以 ApplicationContextAware 为例子, 获取 ApplicationContext 资源, 让工具类继承 ApplicationContextAware, 保存 ApplicationContext 在工具类中, 以便于在任意地方获取容器里面的 Bean 并使用, `注: 因为 ApplicationContext 接口集成了 EnvironmentCapable, ListableBeanFactory, HierarchicalBeanFactory 和 MessageSource 接口ApplicationEventPublisher, ResourcePatternResolver (Spring 5.1.5) 接口, 所以继承ApplicationContextAware 并获取到 ApplicationContext 后可以获得容器的所有资源和功能`

```java
public class ApplicationContextUtils implements ApplicationContextAware {
    @Getter
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        applicationContext = applicationContext;
    }
}
```

接着就可以在任意地方通过此工具类获取 ApplicationContext 的实例, 如获取容器中的 Bean:

```java
MyBeanName myBean = ApplicationContextUtils.getApplicationContext().getBean("myBeanName");
```

这里列出一些常见的 Aware 接口:

| 接口名 | 作用 |
| --- | --- |
| BeanFactoryAware | 获得当前 BeanFactory 实例 |
| ApplicationContextAware | 获取当前 ApplicationContext 实例 |
| ApplicationEventPublisherAware | 获取事件发布器 |
| ResourceLoaderAware | 获得资源加载器, 通过它能获取各种能加载的资源 |
| MessageSourceAware |  获取当前容器加载文本资源 |
| BeanNameAware | 获取 Bean 在容器中的名字 |

所有的 Aware 的子接口:
`ApplicationContextAware, ApplicationEventPublisherAware, BeanClassLoaderAware, BeanFactoryAware, BeanNameAware, BootstrapContextAware, EmbeddedValueResolverAware, EnvironmentAware, ImportAware, LoadTimeWeaverAware, MessageSourceAware, NotificationPublisherAware, ResourceLoaderAware, SchedulerContextAware, ServletConfigAware, ServletContextAware`

再看 addBeanPostProcessor(new ApplicationContextAwareProcessor(this)), 设置 Bean 的处理器, 会在每个 Bean 初始化的时候对 Bean 进行相应的处理, 大部分对容器功能的扩展是基于此功能实现的, 处理器需要实现 BeanPostProcessor 接口的 postProcessBeforeInitialization 接口 (前置处理) 和 postProcessAfterInitialization 接口 (后置处理), 这里面传入的是一个 ApplicationContextAwareProcessor 类型的处理器, ApplicationContextAwareProcessor 处理器, 实现了 BeanPostProcessor 接口, 主要功能是在 Bean 初始化前感知容器, 获取容器的资源, 其核心方法是 postProcessBeforeInitialization 前置处理中的 invokeAwareInterfaces 方法, 如下:

```java
@Override
@Nullable
public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
    AccessControlContext acc = null;
    if (System.getSecurityManager() != null &&
            (bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
                    bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
                    bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }

    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    } else {
        invokeAwareInterfaces(bean);
    }
    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }
}
```

意味着在使用 ClassPathXmlApplicationContext 初始化 Bean 时, 如果 Bean 实现了任意 EnvironmentAware, EmbeddedValueResolverAware, ResourceLoaderAware, ApplicationEventPublisherAware, 
MessageSourceAware 和 ApplicationContextAware 接口, 那么旧可以获取到对应的资源了, 例如让 Bean 获取 ApplicatioContext 的实例:

```java
@Slf4j
@Data
public class UserInfo implements ApplicationContextAware {
    private String name;
    private String email;
    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        applicationContext = applicationContext;
        log.info("获取到 applicationContext 资源");
    }
}

public static void main(String[] args) {
    DefineClassPathXmlApplicationContext applicationContext = new DefineClassPathXmlApplicationContext("beans/UserBeans.xml");
    UserInfo userInfo = applicationContext.getBean("userInfo", UserInfo.class);
    log.info("{}", userInfo.getApplicationContext().equals(applicationContext));
}
```

# 设置忽略的依赖

EnvironmentAware, EmbeddedValueResolverAware, ResourceLoaderAware, ApplicationEventPublisherAware,
MessageSourceAware 和 ApplicationContextAware 接口都是在 Bean 初始化的时候通过回调的方式来实现普通 Bean 获取容器资源的, Spring 并没有将这些接口设计为普通的 Bean
注入已经没有意义了, 需要设置这些接口为忽略的依赖, 这样这些类型在 Bean 中不会被注入进去, 设置忽略依赖代码如下:

```java
beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

# 设置依赖

既然又忽略的依赖, 当然也设置有几个需要依赖的类型, 如下, 这些类型可以直接在 Bean 中注入使用, 当检测到有相同的类型的时候, 就会注入:

```java
beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
beanFactory.registerResolvableDependency(ResourceLoader.class, this);
beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```

如下示例, 如果调用 BeanTest 的 test 方法, 将会输出 test 结果:

```java
@Slf4j
@Component
public class BeanTest {
    private final BeanFactory beanFactory;
    private final ApplicationContext applicationContext;

    @Autowired
    public BeanTest(BeanFactory beanFactory, ApplicationContext applicationContext) {
        this.beanFactory = beanFactory;
        this.applicationContext = applicationContext;
    }

    public void test() {
        BeanTest beanTest = beanFactory.getBean("beanTest", BeanTest.class);
        beanTest.print();
    }

    public void print() {
        log.info("test");
    }
}
```

# 配置默认的环境 Bean

最后配置了几个和环境变量和属性相关的单实例 Bean, 这些 Bean 可以通过 beanFactory.getBean(String beanName) 的方式获取到, 源码如下:

```java
if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
}

if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
}

if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
    beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
}
```

这里要注意的是, 注意这几个 Bean 的类型, 名为 ENVIRONMENT_BEAN_NAME 的 Bean 注册到容器的是 ConfigurableEnvironment 类型的 Bean , 如下直接使用注解的方式也可以注入进去, 如下:

```java
@Autowired
private Environment environment;
```

但是名为 SYSTEM_PROPERTIES_BEAN_NAME 和 SYSTEM_ENVIRONMENT_BEAN_NAME 是 Map 类型的, 我异想天开的想使用 Map 类型直接注入, 但是这样注入的并非其中的任何一个 Bean, 注入的是是当前所有注册到容器中 bean 的集合, 所有在注册到容器的中的 Bean 都可以在这个 Map 中找到

```java
@Autowired
private Map<String, Object> systemProperties;
```