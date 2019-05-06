---
title: Spring-源码阅读DefaultSingletonBeanRegistry
date: 2019-04-19 23:14:59
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# DefaultSingletonBeanRegistry

DefaultSingletonBeanRegistry 单实例 bean 注册的默认实现, 实现了单实例 bean 的注册, 销毁等功能 源码如下: 

```java
public class DefaultSingletonBeanRegistry 
  extends SimpleAliasRegistry implements SingletonBeanRegistry {

    // 已经创建的单实例 bean 的 beanName 和 instance 的映射关系集
    private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

    // 正在创建的的单实例 bean 的 beanName 和 ObjectFactory instance 的映射关系集
    private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

    // 正在创建的单实例 bean 的 beanName 和 instance 的映射关系集, 用于解决循环依赖
    private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);

    // 已经成功创建的单实例 bean 的 beanName 集合
    private final Set<String> registeredSingletons = new LinkedHashSet<>(256);

    // 正在创建的单实例 bean 的 beanBean 集合
    private final Set<String> singletonsCurrentlyInCreation 
      = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    // 在检测某个 bean 是否正在创建时需要排除的在检测外的 bean 的 beanName 集合
    private final Set<String> inCreationCheckExclusions 
      = Collections.newSetFromMap(new ConcurrentHashMap<>(16));

    // 这个记录方法调用链出现的异常
    @Nullable
    private Set<Exception> suppressedExceptions;

    //表示当前所有的单例是否正在被销毁
    private boolean singletonsCurrentlyInDestruction = false;

    // 已经销毁的单实例 bean 的 beanName 和 instance 的映射关系集
    private final Map<String, Object> disposableBeans = new LinkedHashMap<>();

    /**
     * Map between containing bean names: bean name to Set of bean names that the
     * bean contains.
     */
    private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);

    /**
     * Map between dependent bean names: bean name to Set of dependent bean names.
     */
    private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);

    // bean 实例化时需要依赖的 bean 的 beanName 集合映射关系集
    private final Map<String, Set<String>> dependenciesForBeanMap
      = new ConcurrentHashMap<>(64);

    @Override
    public void registerSingleton(String beanName, Object singletonObject) 
      throws IllegalStateException {
        Assert.notNull(beanName, "Bean name must not be null");
        Assert.notNull(singletonObject, "Singleton object must not be null");
        synchronized (this.singletonObjects) {
            // 是否已经创建, 如果有创建抛出异常
            Object oldObject = this.singletonObjects.get(beanName);
            if (oldObject != null) {
                throw new IllegalStateException("Could not register object [" + 
                                                singletonObject + "] under bean name '"
                        + beanName + "': there is already object [" + oldObject + "] bound");
            }
            addSingleton(beanName, singletonObject);
        }
    }

    protected void addSingleton(String beanName, Object singletonObject) {
        synchronized (this.singletonObjects) {
            // 加入到已创建集合
            this.singletonObjects.put(beanName, singletonObject);
            // 从正在创建的缓存集合中移除
            this.singletonFactories.remove(beanName);
            // 从正在创建的集合中移除
            this.earlySingletonObjects.remove(beanName);
            // beanName 添加到已注册的 beanName 集合中
            this.registeredSingletons.add(beanName);
        }
    }

    protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(singletonFactory, "Singleton factory must not be null");
        synchronized (this.singletonObjects) {
            if (!this.singletonObjects.containsKey(beanName)) {
                this.singletonFactories.put(beanName, singletonFactory);
                this.earlySingletonObjects.remove(beanName);
                this.registeredSingletons.add(beanName);
            }
        }
    }

    @Override
    @Nullable
    public Object getSingleton(String beanName) {
        return getSingleton(beanName, true);
    }

    // allowEarlyReference: 当 singletonObjects 和 earlySingletonObjects 都没有取到时,
    // 是否到 singletonFactories 缓存中去取
    @Nullable
    protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        // 先从 singletonObjects 中获取
        Object singletonObject = this.singletonObjects.get(beanName);
        // 如果不存在判断是否正在创建
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                // 如果正在创建, 从 earlySingletonObjects 获取 
                singletonObject = this.earlySingletonObjects.get(beanName);
                //  如果 earlySingletonObjects 没有取到, 就去 singletonFactories 去取
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return singletonObject;
    }

    // 返回一个单实例 bean, 如果这个 bean 不存在, 就会创建并注册
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        synchronized (this.singletonObjects) {
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                if (this.singletonsCurrentlyInDestruction) {
                    throw new BeanCreationNotAllowedException(beanName,
                            "Singleton bean creation not allowed while singletons" + 
                            " of this factory are in destruction " + 
                            "(Do not request a bean from a BeanFactory" + 
                            " in a destroy method implementation!)");
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Creating shared instance of singleton bean '" +
                                 beanName + "'");
                }

                // 主要判断 bean 是否正在创建, 如果正在创建, 抛出异常
                beforeSingletonCreation(beanName);
                boolean newSingleton = false;
                boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
                if (recordSuppressedExceptions) {
                    this.suppressedExceptions = new LinkedHashSet<>();
                }
                try {
                    // 从 ObjectFactory 的 getObject 获取实例
                    singletonObject = singletonFactory.getObject();
                    newSingleton = true;
                } catch (IllegalStateException ex) {
                    // Has the singleton object implicitly appeared in the meantime ->
                    // if yes, proceed with it since the exception indicates that state.
                    singletonObject = this.singletonObjects.get(beanName);
                    if (singletonObject == null) {
                        throw ex;
                    }
                } catch (BeanCreationException ex) {
                    if (recordSuppressedExceptions) {
                        for (Exception suppressedException : this.suppressedExceptions) {
                            ex.addRelatedCause(suppressedException);
                        }
                    }
                    throw ex;
                } finally {
                    if (recordSuppressedExceptions) {
                        this.suppressedExceptions = null;
                    } 
                    // 创建单实例 bean 失败处理
                    afterSingletonCreation(beanName);
                }

                // 如果是一个新的单实例 bean, 添加单实例 bean
                if (newSingleton) {
                    addSingleton(beanName, singletonObject);
                }
            }
            return singletonObject;
        }
    }

    // 记录发生的异常栈
    protected void onSuppressedException(Exception ex) {
        synchronized (this.singletonObjects) {
            if (this.suppressedExceptions != null) {
                this.suppressedExceptions.add(ex);
            }
        }
    }
  
    // 移除指定的单实例 bean
    protected void removeSingleton(String beanName) {
        synchronized (this.singletonObjects) {
            this.singletonObjects.remove(beanName);
            this.singletonFactories.remove(beanName);
            this.earlySingletonObjects.remove(beanName);
            this.registeredSingletons.remove(beanName);
        }
    }

    // 是否包含指定的 bean
    @Override
    public boolean containsSingleton(String beanName) {
        return this.singletonObjects.containsKey(beanName);
    }

    // 返回所有的单实例名称
    @Override
    public String[] getSingletonNames() {
        synchronized (this.singletonObjects) {
            return StringUtils.toStringArray(this.registeredSingletons);
        }
    }

    // 获取单实例的数量
    @Override
    public int getSingletonCount() {
        synchronized (this.singletonObjects) {
            return this.registeredSingletons.size();
        }
    }

    // 指定 bean 是否在检测是否正在创建时是否需要排除
    public void setCurrentlyInCreation(String beanName, boolean inCreation) {
        Assert.notNull(beanName, "Bean name must not be null");
        if (!inCreation) {
            this.inCreationCheckExclusions.add(beanName);
        } else {
            this.inCreationCheckExclusions.remove(beanName);
        }
    }

    // 判断指定的 bean 是否正在创建
    public boolean isCurrentlyInCreation(String beanName) {
        Assert.notNull(beanName, "Bean name must not be null");
        return (!this.inCreationCheckExclusions.contains(beanName) &&
                isActuallyInCreation(beanName));
    }

    protected boolean isActuallyInCreation(String beanName) {
        return isSingletonCurrentlyInCreation(beanName);
    }

    // 判断单实例 bean 是否正在创建
    public boolean isSingletonCurrentlyInCreation(String beanName) {
        return this.singletonsCurrentlyInCreation.contains(beanName);
    }

    // 创建单实例 bean 的前置处理, 主要判断 bean 是否正在创建
    // 如果没有创建, 从正在创建的集合中移除, 如果正在创建, 抛出异常
    protected void beforeSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName) && 
            !this.singletonsCurrentlyInCreation.add(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
    }

    // 创建单实例 bean 的前置处理, 从正在创建的集合中移除
    protected void afterSingletonCreation(String beanName) {
        if (!this.inCreationCheckExclusions.contains(beanName)
                && !this.singletonsCurrentlyInCreation.remove(beanName)) {
            throw new IllegalStateException("Singleton '" + beanName +
                                            "' isn't currently in creation");
        }
    }

    // 将 bean 添加到需要销毁的集合中
    public void registerDisposableBean(String beanName, DisposableBean bean) {
        synchronized (this.disposableBeans) {
            this.disposableBeans.put(beanName, bean);
        }
    }

    /**
     * Register a containment relationship between two beans, e.g. between an inner
     * bean and its containing outer bean.
     * <p>
     * Also registers the containing bean as dependent on the contained bean in
     * terms of destruction order.
     * 
     * @param containedBeanName  the name of the contained (inner) bean
     * @param containingBeanName the name of the containing (outer) bean
     * @see #registerDependentBean
     */
    // ??? 暂时没看懂这个 containedBeanMap 的作用
    public void registerContainedBean(String containedBeanName, 
                                      String containingBeanName) {
        synchronized (this.containedBeanMap) {
            Set<String> containedBeans 
              = this.containedBeanMap.computeIfAbsent(containingBeanName,
                    k -> new LinkedHashSet<>(8));
            if (!containedBeans.add(containedBeanName)) {
                return;
            }
        }
        registerDependentBean(containedBeanName, containingBeanName);
    }

    // 为指定的 bean 添加一个依赖的 bean
    public void registerDependentBean(String beanName, String dependentBeanName) {
        String canonicalName = canonicalName(beanName);

        synchronized (this.dependentBeanMap) {
            Set<String> dependentBeans = this.dependentBeanMap.computeIfAbsent(canonicalName,
                    k -> new LinkedHashSet<>(8));
            if (!dependentBeans.add(dependentBeanName)) {
                return;
            }
        }

        synchronized (this.dependenciesForBeanMap) {
            Set<String> dependenciesForBean = 
              this.dependenciesForBeanMap.computeIfAbsent(dependentBeanName,
                    k -> new LinkedHashSet<>(8));
            dependenciesForBean.add(canonicalName);
        }
    }

    // 判断一个 bean 是否依赖另一个 bean
    protected boolean isDependent(String beanName, String dependentBeanName) {
        synchronized (this.dependentBeanMap) {
            return isDependent(beanName, dependentBeanName, null);
        }
    }

    private boolean isDependent(String beanName, String dependentBeanName, 
                                @Nullable Set<String> alreadySeen) {
        if (alreadySeen != null && alreadySeen.contains(beanName)) {
            return false;
        }
        String canonicalName = canonicalName(beanName);
        Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);
        if (dependentBeans == null) {
            return false;
        }
        if (dependentBeans.contains(dependentBeanName)) {
            return true;
        }
        for (String transitiveDependency : dependentBeans) {
            if (alreadySeen == null) {
                alreadySeen = new HashSet<>();
            }
            alreadySeen.add(beanName);
            if (isDependent(transitiveDependency, dependentBeanName, alreadySeen)) {
                return true;
            }
        }
        return false;
    }

    // 判断一个 bean 是否有依赖
    protected boolean hasDependentBean(String beanName) {
        return this.dependentBeanMap.containsKey(beanName);
    }

    // 获取一个 bean 的所有依赖的 bean 的 beanName 集合
    public String[] getDependentBeans(String beanName) {
        Set<String> dependentBeans = this.dependentBeanMap.get(beanName);
        if (dependentBeans == null) {
            return new String[0];
        }
        synchronized (this.dependentBeanMap) {
            return StringUtils.toStringArray(dependentBeans);
        }
    }

    // 返回创建这个 bean 时所依赖的的 bean 的 beanName 集合
    public String[] getDependenciesForBean(String beanName) {
        Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(beanName);
        if (dependenciesForBean == null) {
            return new String[0];
        }
        synchronized (this.dependenciesForBeanMap) {
            return StringUtils.toStringArray(dependenciesForBean);
        }
    }

    // 销毁所有的单实例
    public void destroySingletons() {
        if (logger.isTraceEnabled()) {
            logger.trace("Destroying singletons in " + this);
        }
        synchronized (this.singletonObjects) {
            this.singletonsCurrentlyInDestruction = true;
        }

        String[] disposableBeanNames;
        synchronized (this.disposableBeans) {
            disposableBeanNames = StringUtils.toStringArray(this.disposableBeans.keySet());
        }

        // 销毁依赖的 bean
        for (int i = disposableBeanNames.length - 1; i >= 0; i--) {
            destroySingleton(disposableBeanNames[i]);
        }

        this.containedBeanMap.clear();
        this.dependentBeanMap.clear();
        this.dependenciesForBeanMap.clear();
        clearSingletonCache();
    }

    // 清除所有缓存
    protected void clearSingletonCache() {
        synchronized (this.singletonObjects) {
            this.singletonObjects.clear();
            this.singletonFactories.clear();
            this.earlySingletonObjects.clear();
            this.registeredSingletons.clear();
            this.singletonsCurrentlyInDestruction = false;
        }
    }

    // 销毁指定的 bean
    public void destroySingleton(String beanName) {
        // Remove a registered singleton of the given name, if any.
        removeSingleton(beanName);
        DisposableBean disposableBean;
        synchronized (this.disposableBeans) {
            disposableBean = (DisposableBean) this.disposableBeans.remove(beanName);
        }
        destroyBean(beanName, disposableBean);
    }

    // 销毁依赖的 bean
    protected void destroyBean(String beanName, @Nullable DisposableBean bean) {
        // 先销毁依赖的 bean
        Set<String> dependencies;
        synchronized (this.dependentBeanMap) {
            dependencies = this.dependentBeanMap.remove(beanName);
        }
        if (dependencies != null) {
            if (logger.isTraceEnabled()) {
                logger.trace("Retrieved dependent beans for bean '" +
                             beanName + "': " + dependencies);
            }
            for (String dependentBeanName : dependencies) {
                destroySingleton(dependentBeanName);
            }
        }

        // Actually destroy the bean now...
        if (bean != null) {
            try {
                bean.destroy();
            } catch (Throwable ex) {
                if (logger.isInfoEnabled()) {
                    logger.info("Destroy method on bean with name '" + 
                                beanName + "' threw an exception", ex);
                }
            }
        }

        // Trigger destruction of contained beans...
        Set<String> containedBeans;
        synchronized (this.containedBeanMap) {
            // Within full synchronization in order to guarantee a disconnected Set
            containedBeans = this.containedBeanMap.remove(beanName);
        }
        if (containedBeans != null) {
            for (String containedBeanName : containedBeans) {
                destroySingleton(containedBeanName);
            }
        }

        // Remove destroyed bean from other beans' dependencies.
        synchronized (this.dependentBeanMap) {
            for (Iterator<Map.Entry<String, Set<String>>> it 
                 = this.dependentBeanMap.entrySet().iterator(); it.hasNext();) {
                Map.Entry<String, Set<String>> entry = it.next();
                Set<String> dependenciesToClean = entry.getValue();
                dependenciesToClean.remove(beanName);
                if (dependenciesToClean.isEmpty()) {
                    it.remove();
                }
            }
        }

        // 清除缓存
        this.dependenciesForBeanMap.remove(beanName);
    }

    // 返回已创建的实例的缓存
    public final Object getSingletonMutex() {
        return this.singletonObjects;
    }
}
```
