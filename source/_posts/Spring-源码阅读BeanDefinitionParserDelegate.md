---
title: Spring-源码阅读BeanDefinitionParserDelegate
date: 2019-04-14 21:24:35
categories: Spring
---

# BeanDefinitionParserDelegate

BeanDefinitionParserDelegate 是用于将定义了 bean 的 XML 文件解析成 BeanDefinition  类的委托类, 在看 XmlBeanFactory 源码时, 需要将 XML 配置的 bena 进行解析, 主要调用的是 BeanDefinitionDocumentReader 的 registerBeanDefinitions 方法, 如下:

```java
public int registerBeanDefinitions(Document doc, Resource resource) 
    throws BeanDefinitionStoreException {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}=
```

<!-- more -->

实际上又调用了 BeanDefinitionDocumentReader 的 registerBeanDefinitions 方法, 如下:

```java
@Override
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    his.readerContext = readerContext;
    doRegisterBeanDefinitions(doc.getDocumentElement());
}

protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Skipped XML bean definition file due to specified profiles ["
                            + profileSpec +
                            "] not matching: "
                            + getReaderContext().getResource());
                    }
                    return;
                }
        }  
    }
    preProcessXml(root);
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);
    this.delegate = parent;
}

protected BeanDefinitionParserDelegate createDelegate( XmlReaderContext readerContext,
Element root, BeanDefinitionParserDelegate parentDelegate) {
    BeanDefinitionParserDelegate delegate
        = new BeanDefinitionParserDelegate(readerContext);
    delegate.initDefaults(root, parentDelegate);
    return delegate;
}
```

# initDefaults

可以看到首先创建 BeanDefinitionParserDelegate 示例后接着调用了 initDefaults 方法, 主要调用 populateDefaults 方法, 源码如下:

```java
public void initDefaults(Element root) {
    initDefaults(root, null);
}

public void initDefaults(Element root, @Nullable BeanDefinitionParserDelegate parent) {
    populateDefaults(this.defaults, (parent != null ? parent.defaults : null), root);
    this.readerContext.fireDefaultsRegistered(this.defaults);
}
```

# populateDefaults

通常我们设置一个 bean 的 lazy-init 是在每个 bean 节点设置的, 对于同一个 XML 文件的所有 bean 节点可以在 beans 节点设置 default-lazy-init 来设置, 相当于定义一个全局的行为, 如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
       http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-lazy-init="true"
       default-init-method="initFunction"
       default-destroy-method="destoryFunction"
       default-autowire="byType"
       default-autowire-candidates="*abc"
       default-merge="false">

    <bean id="userInfo" class="com.spring.study.beans.UserInfo" lazy-init="true">
        <constructor-arg name="name" value="lupengwei"/>
        <constructor-arg name="email" value="lupengwei@163.com"/>
    </bean>
</beans>
```

populateDefaults 方法解析这些值, 解析后的值存放在类型为 DocumentDefaultsDefinition 的 defaults 实例中, 源码如下:

```java
protected void populateDefaults(DocumentDefaultsDefinition defaults,
                                DocumentDefaultsDefinition parentDefaults,
                                Element root) {
    // default-lazy-init
    String lazyInit = root.getAttribute(DEFAULT_LAZY_INIT_ATTRIBUTE);
    if (isDefaultValue(lazyInit)) {
        lazyInit = (parentDefaults != null ? parentDefaults.getLazyInit() : FALSE_VALUE);
    }
    defaults.setLazyInit(lazyInit);

    // default-merge
    String merge = root.getAttribute(DEFAULT_MERGE_ATTRIBUTE);
    if (isDefaultValue(merge)) {
        merge = (parentDefaults != null ? parentDefaults.getMerge() : FALSE_VALUE);
    }
    defaults.setMerge(merge);

    // default-autowire
    String autowire = root.getAttribute(DEFAULT_AUTOWIRE_ATTRIBUTE);
    if (isDefaultValue(autowire)) {
        autowire = (parentDefaults != null ? 
                    parentDefaults.getAutowire() : AUTOWIRE_NO_VALUE);
    }
    defaults.setAutowire(autowire);

    // default-autowire-candidates
    if (root.hasAttribute(DEFAULT_AUTOWIRE_CANDIDATES_ATTRIBUTE)) {
        defaults.setAutowireCandidates(
        root.getAttribute(DEFAULT_AUTOWIRE_CANDIDATES_ATTRIBUTE));
    }
    else if (parentDefaults != null) {
        defaults.setAutowireCandidates(parentDefaults.getAutowireCandidates());
    }

    // default_init_method
    if (root.hasAttribute(DEFAULT_INIT_METHOD_ATTRIBUTE)) {
        defaults.setInitMethod(root.getAttribute(DEFAULT_INIT_METHOD_ATTRIBUTE));
    } else if (parentDefaults != null) {
        defaults.setInitMethod(parentDefaults.getInitMethod());
    }

    // default-destory-method
    if (root.hasAttribute(DEFAULT_DESTROY_METHOD_ATTRIBUTE)) {
        defaults.setDestroyMethod(root.getAttribute(DEFAULT_DESTROY_METHOD_ATTRIBUTE));
    } else if (parentDefaults != null) {
        defaults.setDestroyMethod(parentDefaults.getDestroyMethod());
    }

    defaults.setSource(this.readerContext.extractSource(root));
}
```

# preProcessXml 和 postProcessXml

在 createDelegate 方法调用完成后, 接着调用了 preProcessXml(root),parseBeanDefinitions(root, this.delegate), postProcessXml(root) 方法, 其中 preProcessXml 和 postProcessXml 这两个个方法没有具体的实现, 是为了方便用户扩展使用, 如果需要在 bean 解析前后做一些处理的话, 只需要继承 DefaultBeanDefinitionDocumentReader 并重写这两个方法

```java
protected void preProcessXml(Element root) {}

protected void postProcessXml(Element root) {}
```

# parseBeanDefinitions

parseBeanDefinitions(root, this.delegate) 方法是我们的需要了解的核心方法了, 源码如下:

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    // 对 import 标签的处理
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        importBeanDefinitionResource(ele);
    }
    // 对 alias 标签的处理
    else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        processAliasRegistration(ele);
    }
    // 对 bean 标签的处理
    else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        processBeanDefinition(ele, delegate);
    }
    // 对 beans 标签的处理
    else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
        doRegisterBeanDefinitions(ele);
    }
}
```

这个方法涉及到了对 import, alais, bean, beans 标签的处理

# importBeanDefinitionResource

import 标签可以将 Spring 的多个配置文件整合到一个配置文件然后加载, 可以简化 Spring 中的配置文件, 使用示例如下, 先创建两个 XML 配置文件:

UserBeansImport.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans 
       http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-lazy-init="true">

    <bean id="userInfoImport" class="com.spring.study.beans.UserInfoImport" 
          lazy-init="true" name="userInfoImportAlias1,userInfoImportAlias2">
        <constructor-arg name="name" value="lupengwei"/>
        <constructor-arg name="email" value="lupengwei@163.com"/>
    </bean>
</beans>
```

UserBeans.xml

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd"
       default-lazy-init="true">

    <import resource="UserBeansImport.xml"/>

    <bean id="userInfo" class="com.spring.study.beans.UserInfo" 
          lazy-init="true" name="userInfoBeanName1,userInfoBeanName2">
        <constructor-arg name="name" value="lupengwei"/>
        <constructor-arg name="email" value="lupengwei@163.com"/>
    </bean>
</beans>
```

只需要加载 UserBeans.xml 即可将 UserBeansImport.xml 也加载, 测试代码如下:

```java
XmlBeanFactory beanFactory =
    new XmlBeanFactory(new ClassPathResource("/beans/UserBeans.xml"));
UserInfoImport userInfo = (UserInfoImport) beanFactory.getBean("userInfoImport");
```

因为多个 Spring 配置文件会合并到一起, 因此这些配置中的 bean 都是可以互相引用的

在来看 importBeanDefinitionResource 源码, 可以看到, 主要逻辑是读取 import 标签中的 resoure 的值, 定位到资源文件, 接着还是调用 getReaderContext().getReader().loadBeanDefinitions(location, actualResources) 方法来解析 XML 中定义的 bean, 跟踪代码发现之后又进入了的步骤就和文章开始上面的解析逻辑了

```java
protected void importBeanDefinitionResource(Element ele) {
    String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
    if (!StringUtils.hasText(location)) {
        getReaderContext().error("Resource location must not be empty", ele);
        return;
    }

    // Resolve system properties: e.g. "${user.dir}"
    location = getReaderContext().getEnvironment()
        .resolveRequiredPlaceholders(location);

    Set<Resource> actualResources = new LinkedHashSet<>(4);

    // Discover whether the location is an absolute or relative URI
    boolean absoluteLocation = false;
    try {
        absoluteLocation = ResourcePatternUtils.isUrl(location) 
        || ResourceUtils.toURI(location).isAbsolute();
    } catch (URISyntaxException ex) {
        // cannot convert to an URI, considering the location relative
        // unless it is the well-known Spring prefix "classpath*:"
    }

    if (absoluteLocation) {
        try {
        int importCount = getReaderContext().getReader()
            .loadBeanDefinitions(location, actualResources);
        if (logger.isTraceEnabled()) {
            logger.trace("Imported " + importCount + 
                        " bean definitions from URL location [" + location + "]");
        }
        }
        catch (BeanDefinitionStoreException ex) {
        getReaderContext().error(
            "Failed to import bean definitions from URL location [" +
            location + "]", ele, ex);
        }
    } else {
        try {
        int importCount;
        Resource relativeResource = getReaderContext().getResource()
            .createRelative(location);
        // 解析 import 导入的 XML 配置文件
        if (relativeResource.exists()) {
            importCount = getReaderContext().getReader()
            .loadBeanDefinitions(relativeResource);
            actualResources.add(relativeResource);
        } else {
            String baseLocation = getReaderContext().getResource().getURL().toString();
            importCount = getReaderContext().getReader().loadBeanDefinitions(
            StringUtils.applyRelativePath(baseLocation, location), actualResources);
        }
        if (logger.isTraceEnabled()) {
            logger.trace("Imported " + importCount + 
                        " bean definitions from relative location [" + location + "]");
        }
        } catch (IOException ex) {
        getReaderContext().error("Failed to resolve current resource location", 
                                ele, ex);
        } catch (BeanDefinitionStoreException ex) {
        getReaderContext().error(
            "Failed to import bean definitions from relative location [" +
            location + "]", ele, ex);
        }
    }
    Resource[] actResArray = actualResources.toArray(new Resource[0]);
    getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```

# processAliasRegistration

alias 标签为指定的 bean 取一个别名, 使用示例如下:

```java
<bean id="userInfo" class="com.spring.study.beans.UserInfo" 
  lazy-init="true" name="userInfoBeanName1,userInfoBeanName2">
    <constructor-arg name="name" value="lupengwei"/>
    <constructor-arg name="email" value="lupengwei@163.com"/>
</bean>

<alias name="userInfo" alias="userInfoAlias1"/>
```

processAliasRegistration 源码如下, 比较简单, 主要逻辑为调用 SimpleAliasRegistry 的 registerAlias 方法将别名添加 SimpleAliasRegistry 类维护的注册表 aliasMap 中:

```java
protected void processAliasRegistration(Element ele) {
    String name = ele.getAttribute(NAME_ATTRIBUTE);
    String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
    boolean valid = true;
    if (!StringUtils.hasText(name)) {
        getReaderContext().error("Name must not be empty", ele);
        valid = false;
    }
    
    if (!StringUtils.hasText(alias)) {
        getReaderContext().error("Alias must not be empty", ele);
        valid = false;
    }
    
    if (valid) {
        try {
        // 把 alias 注册到 SimpleAliasRegistry.aliasMap 中
        getReaderContext().getRegistry().registerAlias(name, alias);
        }
        catch (Exception ex) {
        getReaderContext().error("Failed to register alias '" + alias +
                                "' for bean with name '" + name + "'", ele, ex);
        }
        getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
    }
}
```

# doRegisterBeanDefinitions

对于 beans 标签就没什么好多的了, 重新调用了 doRegisterBeanDefinitions 方法而已

# processBeanDefinition

对 bean 的标签的解析并将 bean 注册到容器里面

```java
protected void processBeanDefinition(Element ele,
                                     BeanDefinitionParserDelegate delegate) {
    // 解析 bean 标签
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
        // bean 注册带容器
        BeanDefinitionReaderUtils.registerBeanDefinition(
            bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
        getReaderContext().error("Failed to register bean definition with name '" +
                                bdHolder.getBeanName() + "'", ele, ex);
        }
        // 发送注册事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

# parseBeanDefinitionElement

解析 bean 标签, 调用的是, parseBeanDefinitionElement(Element ele) 方法,  返回 BeanDefinitionHolder 实例, BeanDefinitionHolder 是对 BeanDefinition 实例的封装, 将 BeanDefinition 的 beanName 和 alias 属性在 BeanDefinitionHolder 存放了一份, 方便使用, 源码如下:

```java
@Nullable
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    return parseBeanDefinitionElement(ele, null);
}
```

可以看出实际上调用的是 parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) 方法, 该方法第二个参数表示是否将 containingBean 的 baneName 和 alias 应用到解析出来的 BeanDefinition 上, 这传入的是 null:

```java

public BeanDefinitionHolder parseBeanDefinitionElement(
    Element ele, @Nullable BeanDefinition containingBean) {
    // ID_ATTRIBUTE = id, 提取 bean 的 id
    String id = ele.getAttribute(ID_ATTRIBUTE);
    
    // NAME_ATTRIBUTE = name, MULTI_VALUE_ATTRIBUTE_DELIMITERS = ",; "
    // 提取 bean 定义的别名
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
    List<String> aliases = new ArrayList<>();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(
        nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    // 默认 bean 的名称和 id 的定义是一样的, 如果 id 没有定义, 
    // 就使用第一个定义的别名作为 bean 名称, 并移除第一个别名, 因为别名和 bean 名称一样无意义
    String beanName = id;
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
        if (logger.isTraceEnabled()) {
        logger.trace("No XML 'id' specified - using '" + beanName +
                    "' as bean name and " + aliases + " as aliases");
        }
    }

    // 如果不是将 XML 定义的 bean 应用于一个已存在的 BeanDefinition 中
    // 需要检测这些 beanName 和 alais 是否符合命名要求
    // 如果是将 XML 定义的 bean 应用于一个已存在的 BeanDefinition 中
    // 直接应用已存在的 BeanDefinition 的 beanName 和 alais 即可, 不需要检测了
    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }

    // 解析 XML
    AbstractBeanDefinition beanDefinition 
        = parseBeanDefinitionElement(ele, beanName, containingBean);
    
    // 成功解析 bean 并创建了 BeanDefinition
    if (beanDefinition != null) {
        // 解析出来的 bean 没有定义 beanName 
        if (!StringUtils.hasText(beanName)) {
        try {
            // 使用 containingBean 不为 null
            // BeanDefinitionReaderUtils#generateBeanName 方法生成一个 beanName
            // 否则是用全类名作为 beanName
            if (containingBean != null) {
            beanName = BeanDefinitionReaderUtils.generateBeanName(
                beanDefinition, this.readerContext.getRegistry(), true);
            } else {
            beanName = this.readerContext.generateBeanName(beanDefinition);
            String beanClassName = beanDefinition.getBeanClassName();
            if (beanClassName != null &&
                beanName.startsWith(beanClassName) && 
                beanName.length() > beanClassName.length() &&
                !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                aliases.add(beanClassName);
            }
            }
            if (logger.isTraceEnabled()) {
            logger.trace("Neither XML 'id' nor 'name' specified - " +
                        "using generated bean name [" + beanName + "]");
            }
        } catch (Exception ex) {
            error(ex.getMessage(), ele);
            return null;
        }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    return null;
}
```

可以了解到解析方法又做了封装, 调用的是 parseBeanDefinitionElement(ele, beanName, containingBean) 方法, 源码如下:

```java
@Nullable
public AbstractBeanDefinition parseBeanDefinitionElement(Element ele,
                                                         String beanName,
                                                         BeanDefinition containingBean) {

    // 入栈当前 bean 的解析状态
    this.parseState.push(new BeanEntry(beanName));

    // 获取 class 的值, 即 bean 的全类名
    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }

    String parent = null;
    if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
        parent = ele.getAttribute(PARENT_ATTRIBUTE);
    }

    try {
        // 创建 BeanDefinition 实例, 并通过类型来床架 bean 实例
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        // 解析 bean 节点中的属性, 如 init-method, primary 等
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);

        // description 属性
        bd.setDescription(DomUtils.getChildElementValueByTagName(
        ele, DESCRIPTION_ELEMENT));

        // 解析元数据
        parseMetaElements(ele, bd);

        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        // 构造方法参数和值解析
        parseConstructorArgElements(ele, bd);

        // 属性值解析, 包括 ref, set, list 等属性的解析
        parsePropertyElements(ele, bd);

        // qualifier 属性的解析
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));
        return bd;
    } catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    } catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    } catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    } finally {
        this.parseState.pop();
    }
    return null;
}
```