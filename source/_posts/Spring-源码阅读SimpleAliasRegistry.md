---
title: Spring-源码阅读SimpleAliasRegistry
date: 2019-04-14 21:20:45
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# SimpleAliasRegistry

SimpleAliasRegistry 是 AliasRegistry 的实现类,  源码如下: 

```java
public class SimpleAliasRegistry implements AliasRegistry {
    protected final Log logger = LogFactory.getLog(getClass());

    // 别名注册表, 使用 Map 对保存已经添加的别名, key = alias, value = name
    private final Map<String, String> aliasMap = new ConcurrentHashMap<>(16);

    @Override
    public void registerAlias(String name, String alias) {
        Assert.hasText(name, "'name' must not be empty");
        Assert.hasText(alias, "'alias' must not be empty");
        synchronized (this.aliasMap) {
            if (alias.equals(name)) {
            // 别名和 name 相同, 别名的存在没有任何意义, 直接从注册表中删除
                this.aliasMap.remove(alias);
                if (logger.isDebugEnabled()) {
                    logger.debug("Alias definition '" + 
                       alias + 
                       "' ignored since it points to same name");
                }
            } else {
                String registeredName = this.aliasMap.get(alias);
                if (registeredName != null) {
                   // 注册的别名和旧的一样, 不需要在注册
                    if (registeredName.equals(name)) {
                        return;
                    }

                    // 不支持对别名的重写
                    if (!allowAliasOverriding()) {
                        throw new IllegalStateException("Cannot define alias '" + alias +
                            "' for name '" +
                            name +
                            "': It is already registered for name '" +
                            registeredName + "'.");
                    }
                    if (logger.isDebugEnabled()) {
                        logger.debug("Overriding alias '" + alias +
                            "' definition for registered name '" +
                            registeredName +
                            "' with new target name '" + 
                            name + "'");
                    }
                }

                // 循环检测, 给定的 name 和 alias 不在注册表中存在, 如果存在旧抛出异常 
                checkForAliasCircle(name, alias);
                this.aliasMap.put(alias, name);
                if (logger.isTraceEnabled()) {
                    logger.trace("Alias definition '" + alias +
                        "' registered for name '" +
                        name + "'");
                }
            }
        }
    }

    // 是否允许重写别名
    protected boolean allowAliasOverriding() {
        return true;
    }

    // 循环判断别名是否存在
    public boolean hasAlias(String name, String alias) {
        for (Map.Entry<String, String> entry : this.aliasMap.entrySet()) {
            String registeredName = entry.getValue();
            if (registeredName.equals(name)) {
                String registeredAlias = entry.getKey();
                if (registeredAlias.equals(alias) || hasAlias(registeredAlias, alias)) {
                    return true;
                }
            }
        }
        return false;
    }

    // 删除别名
    @Override
    public void removeAlias(String alias) {
        synchronized (this.aliasMap) {
            String name = this.aliasMap.remove(alias);
            if (name == null) {
                throw new IllegalStateException("No alias '" + alias + "' registered");
            }
        }
    }

    // 判断别名是否存在
    @Override
    public boolean isAlias(String name) {
        return this.aliasMap.containsKey(name);
    }

    // 获取全部别名
    @Override
    public String[] getAliases(String name) {
        List<String> result = new ArrayList<>();
        synchronized (this.aliasMap) {
            retrieveAliases(name, result);
        }
        return StringUtils.toStringArray(result);
    }

    private void retrieveAliases(String name, List<String> result) {
        this.aliasMap.forEach((alias, registeredName) -> {
            if (registeredName.equals(name)) {
                result.add(alias);
                retrieveAliases(alias, result);
            }
        });
    }

    // 使用指定的 StringValueResolver 对别名进行解析, 新别名和名称使用解析出来的值
    public void resolveAliases(StringValueResolver valueResolver) {
        Assert.notNull(valueResolver, "StringValueResolver must not be null");
        synchronized (this.aliasMap) {
            Map<String, String> aliasCopy = new HashMap<>(this.aliasMap);
            aliasCopy.forEach((alias, registeredName) -> {
                String resolvedAlias = valueResolver.resolveStringValue(alias);
                String resolvedName = valueResolver.resolveStringValue(registeredName);
                if (resolvedAlias == null || resolvedName == null 
            || resolvedAlias.equals(resolvedName)) {
                    this.aliasMap.remove(alias);
                }
                else if (!resolvedAlias.equals(alias)) {
                    String existingName = this.aliasMap.get(resolvedAlias);
                    if (existingName != null) {
                        if (existingName.equals(resolvedName)) {
                            this.aliasMap.remove(alias);
                            return;
                        }
                        throw new IllegalStateException("Cannot register resolved alias '" 
                                            + resolvedAlias 
                                            + "' (original: '" 
                                            + alias 
                                            + "') for name '" 
                                            + resolvedName 
                                            + "': It is already registered for name '"
                                            + registeredName + "'.");
                    }
                    checkForAliasCircle(resolvedName, resolvedAlias);
                    this.aliasMap.remove(alias);
                    this.aliasMap.put(resolvedAlias, resolvedName);
                }
                else if (!registeredName.equals(resolvedName)) {
                    this.aliasMap.put(alias, resolvedName);
                }
            });
        }
    }

    protected void checkForAliasCircle(String name, String alias) {
        if (hasAlias(alias, name)) {
            throw new IllegalStateException("Cannot register alias '" + alias +
                    "' for name '" + name + "': Circular reference - '" +
                    name + "' is a direct or indirect alias for '" + alias + "' already");
        }
    }

    public String canonicalName(String name) {
        String canonicalName = name;
        String resolvedName;
        do {
            resolvedName = this.aliasMap.get(canonicalName);
            if (resolvedName != null) {
                canonicalName = resolvedName;
            }
        }
        while (resolvedName != null);
        return canonicalName;
    }
}
```