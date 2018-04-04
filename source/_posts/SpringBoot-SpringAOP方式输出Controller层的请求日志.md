---
title: SpringBoot-SpringAOP方式输出Controller层的请求日志
date: 2018-04-04 18:26:36
categories:  SpringBoot
---

Java Web 后台开发的时候好的日志输出有助于于排查错误, 我在开发时做出的接口和文档，前端查看文档查看接口对接接口, 经常对前端说的是 "你发个请求看看, 我打个断点看看是不是你有参数有没传错"等等, 虽然在项目中使用了日志记录了一些异常, 但是排除接口请求问题的时候真的有时候感觉特别不方便, 如果能对每个请求在调试开发的时候能够将请求相关的信息在日志中输出, 觉得对排查错误有很大的帮助, 基于此在谷歌上搜索相关的资料后, 使用 Spring AOP 可以解决这个问题(也有基于拦截器方法实现的), AOP 相关的知识这里就不讨论了, 看看如何完成这个功能。

# 添加依赖

首先添加依赖, 由于我使用的是 SpringBoot, 添加 spring-boot-starter-aop 依赖, 同时我使用的 fastsson 用于 JSON 数据的处理, 所以也添加了 fastjson 依赖, 还使用用了 Lombok, 日志则是使用的是 Log4j2。
<!-- more -->

``` xml
<!-- SpringBoot AOP -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<!-- fastjson -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.32</version>
</dependency>

<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.20</version>
    <scope>provided</scope>
</dependency>
```

# 配置使用

``` ini
\# Add @EnableAspectJAutoProxy 默认是 true, 这样就不需要在 Application 类里面去添加 @EnableAspectJAutoProxy 注解了
spring.aop.auto=true
\# 需要使用CGLIB来实现AOP的时候，需要配置spring.aop.proxy-target-class=true，不然默认使用的是标准Java的实现
spring.aop.proxy-target-class=true
```

# 定义切片

主要定义了两个 startTimeThreadLocal 和 joinPointThreadLocal 两个变量用于同步的问题, 整个日志的输出使用 @Before 和 @AfterReturning 两个注解实现 (当然也可以直接使用 @Around 注解直接实现, 这样也可以不用关心什么同步的问题了), 至于 AOP 的相关知识这里就不多逼逼了, 剩下的全部直接贴代码了。

``` java
@Aspect
@Component
@Log4j2(topic = "logLog")
public class GloberLogInterceptor {
    private ThreadLocal<Long> startTimeThreadLocal = new ThreadLocal<>();
    private ThreadLocal<JoinPoint> joinPointThreadLocal = new ThreadLocal<>();

    /**
     * 申明一个切点 里面是 execution表达式
     */
    @Pointcut("execution(public * com.mw.tdx.controller..*.*(..))")
        private void controllerAspect() {
    }

    /**
     * Before 在方法执行之前执行
     */
    @Before("controllerAspect()")
    public void doBefore(JoinPoint joinPoint) {
        // 记录开始时间和保存切点
        startTimeThreadLocal.set(System.currentTimeMillis());
        joinPointThreadLocal.set(joinPoint);
    }


    /**
     * 在方法执行完结后打印返回内容
     *
     * @param o 响应的参数
     */
    @AfterReturning(returning = "o", pointcut = "controllerAspect()")
    public void methodAfterReturning(Object o) throws NotFoundException, ClassNotFoundException {
        JoinPoint joinPoint = joinPointThreadLocal.get();

        // 获取被切参数名称及参数值
        String classType = joinPoint.getTarget().getClass().getName();
        Class<?> clazz = Class.forName(classType);
        String clazzName = clazz.getName();
        String methodName = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        Map<String, Object> nameAndArgs = getFieldsName(this.getClass(), clazzName, methodName, args);

        // 获取 HttpServletRequest
        ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
        HttpServletRequest request = attributes.getRequest();

        StringBuilder stringBuilder = new StringBuilder("-------------- 请求内容 ----------------\n");
        stringBuilder.append("请求地址 : ").append(request.getRequestURL().toString()).append("\n")
                .append("请求方式 : ").append(request.getMethod()).append("\n")
                .append("请求方法 : ").append(joinPoint.getSignature().getDeclaringTypeName()).append(".").append(joinPoint.getSignature().getName()).append("\n")
                .append("请求地址 : ").append(request.getRemoteAddr()).append("\n")
                .append("请求参数 : ").append(nameAndArgs).append("\n")
                .append("响应时间 : ").append(System.currentTimeMillis() - startTimeThreadLocal.get()).append("ms").append("\n")
                .append("响应内容 : ").append(JSON.toJSON(o)).append("\n");
        startTimeThreadLocal.remove();
        joinPointThreadLocal.remove();
        log.info(stringBuilder);
    }

    /**
     * 通过反射机制 获取被切参数名以及参数值
     *
     * @param cls        cls
     * @param clazzName  clazzName
     * @param methodName methodName
     * @param args       args
     * @return 参数名和参数值
     * @throws NotFoundException NotFoundException
     */
    private Map<String, Object> getFieldsName(Class cls, String clazzName, String methodName, Object[] args) throws NotFoundException {
        LinkedHashMap<String, Object> map = new LinkedHashMap<>();

        ClassPool pool = ClassPool.getDefault();
        ClassClassPath classPath = new ClassClassPath(cls);
        pool.insertClassPath(classPath);

        CtClass cc = pool.get(clazzName);
        CtMethod cm = cc.getDeclaredMethod(methodName);
        MethodInfo methodInfo = cm.getMethodInfo();
        CodeAttribute codeAttribute = methodInfo.getCodeAttribute();

        LocalVariableAttribute attr = (LocalVariableAttribute) codeAttribute.getAttribute(LocalVariableAttribute.tag);
        if (attr != null) {
            int pos = Modifier.isStatic(cm.getModifiers()) ? 0 : 1;
            for (int i = 0; i < cm.getParameterTypes().length; i++) {
                map.put(attr.variableName(i + pos), args[i]);
            }
        }
        return map;
    }
}
```

上面的代码使用了反射区获取请求参数的相关信息, request 有一个 getParameterMap 方法也是可以获取到参数的, 只不过他获取的参数值是 String[] 类型的, 这样我们可以通过反射可以获取参数。

``` java
/**
    * 在方法执行完结后打印返回内容
    *
    * @param o 响应的参数
    */
@AfterReturning(returning = "o", pointcut = "controllerAspect()")
public void methodAfterReturning(Object o) throws NotFoundException, ClassNotFoundException {
    JoinPoint joinPoint = joinPointThreadLocal.get();

    // 获取被切参数名称及参数值
    String classType = joinPoint.getTarget().getClass().getName();
    Class<?> clazz = Class.forName(classType);
    String clazzName = clazz.getName();
    String methodName = joinPoint.getSignature().getName();
    Object[] args = joinPoint.getArgs();
    Map<String, Object> nameAndArgs = getFieldsName(this.getClass(), clazzName, methodName, args);

    // 获取 HttpServletRequest
    ServletRequestAttributes attributes = (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
    HttpServletRequest request = attributes.getRequest();

    StringBuilder stringBuilder = new StringBuilder("-------------- 请求内容 ----------------\n");
    stringBuilder.append("请求地址 : ").append(request.getRequestURL().toString()).append("\n")
            .append("请求方式 : ").append(request.getMethod()).append("\n")
            .append("请求方法 : ").append(joinPoint.getSignature().getDeclaringTypeName()).append(".").append(joinPoint.getSignature().getName()).append("\n")
            .append("请求地址 : ").append(request.getRemoteAddr()).append("\n")
            .append("请求参数 : ").append(nameAndArgs).append("\n")
            .append("响应时间 : ").append(System.currentTimeMillis() - startTimeThreadLocal.get()).append("ms").append("\n")
            .append("响应内容 : ").append(JSON.toJSON(o)).append("\n");
    startTimeThreadLocal.remove();
    joinPointThreadLocal.remove();
    log.info(stringBuilder);
}
```

发送一个请求, 获取的日志输出的一个例子如下, 可以看到我们想要输出请求信息的详细日志, 终于不用去问前端你传的什么参数或者你是不是漏传了什么参数, 然后还要自己打 debug 调试了。

``` txt
[18:19:53] [INFO] - com.mw.tdx.message.system.common.GloberLogInterceptor.methodAfterReturning(GloberLogInterceptor.java:76) - -------------- 请求内容 ----------------
请求地址 : http://192.168.1.177:8182/tdx/get/element/ui/company/organization/list
请求方式 : POST
请求方法 : com.mw.tdx.message.system.controller.SysValuesController.getElementUiCompanyOrganizationList
请求地址 : 192.168.1.177
请求参数 : {"token":"eyJhbGciOiJIUzI1NiJ9.eyJhY2NvdW50SWQiOiJhYWE0NWExNmRiMjYzNTFhNjc1YWM4ZWMxZGFlMGFjYiIsIm5iZiI6MTUyMjI5NTQyNCwiZXhwIjoxNTIyOTAwLCJpYXQiOjE1MjIyOTU0MjR9.gs8_DQDULv9Y7iGdib-6yqhHn4itzxvJN1jN-PTky1s"}
响应时间 : 106ms
响应内容 : {"msg":"成功!","code":10000,"data":{"dataList":[{"children":[{"children":[{"label":"测试片区001","value":"b094fc87a7f8aecae4391f6df29604ac"},{"label":"测试片区002","value":"30209f351e9b2b250ed275c29216986d"}],"label":"测试分部","value":"e99bfdd474c4ea834151d25fc2d45684"},{"children":[{"label":"金福片区","value":"c69324dacb90137b087b6f0504b22208"},{"label":"清水河片区","value":"e6afb1063888d127b184715c74c13f9f"},{"label":"田心片区","value":"8f86c3b2edf09e8c38a9e089c07797fb"},{"label":"水贝片区","value":"95fbfd68be7db573edd79cdeeacce2ab"},{"label":"碧波片区","value":"99e36dd4d8e6cca5c9cb626739cbcff9"},{"label":"办公室","value":"b44e09f7db985d7c006b62519d545dc4"},{"label":"太白路片区","value":"8c964edcca1f42a17bfac049457caf22"},{"label":"鹏基片区","value":"9d598356aacf6f3e9872753c4542babb"},{"label":"东乐片区","value":"b1ed3b2f7f52376f1297ee64a29bbb24"},{"label":"大望片区","value":"f163d3a8b78e57d10e49026c30bedb89"},{"label":"草埔片区","value":"77d469cac16940f633f09eda2aa9265c"},{"label":"八卦岭片区","value":"c313975214fd7bda36ab766d50dacc85"},{"label":"银湖片区","value":"7c2b7bc174b3c3d85bcaa5c6fa4fb9aa"}],"label":"草埔","value":"2ba3af865de3d3f1fc64bd6c250028eb"}],"label":"韵达快递","value":"13af6d5cb7cbc0db00a879ab08b31a6b"},{"children":[{"children":[{"label":"测试片区002","value":"3ba128e41aa0b4ca6091ccd6127efcc6"},{"label":"测试片区001","value":"fb8c3e6e641a6ce519bc0c00a226cfe8"}],"label":"测试分部001","value":"d771ee31a54a77257e3f271ecbef285c"}],"label":"圆通快递","value":"9f3731983b54bb708dc9b222a67dced6"},{"children":[{"children":[{"label":"片区002","value":"c1fd42347a89be1a5dde1c0657a4605a"},{"label":"片区001","value":"0168fe21fe60de78ffebeef2389174c8"}],"label":"测试分部01","value":"3a0f43abf870dfca030e9dff345868e9"}],"label":"中通快递","value":"5f9964ecf6ab71ea2beb1451f03752b1"},{"children":[{"children":[{"label":"测试片区001","value":"176c0db3cb26258eb6e339107ad1e211"},{"label":"东门区","value":"32405ba0249175ce44eb9c469404e4b0"},{"label":"笋岗区","value":"4ec9e59ac2cfedc363e9b495038a21bc"}],"label":"罗湖区六部","value":"5f375b35ae64fcb0f895f6ecb322bf49"}],"label":"百世快递","value":"942df9ecd9e9c1a0d1182f7337f6e339"}]}}
```

# 参考资料

Spring Boot中使用AOP统一处理Web请求日志 : `http://blog.didispace.com/springbootaoplog/`
spring-boot使用AOP统一处理日志 : `https://my.oschina.net/u/2278977/blog/799786`
SpringBoot AOP统一处理请求日志 : `https://blog.csdn.net/u010412719/article/details/70183180`