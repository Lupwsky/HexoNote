---
title: SpringBoot-集成Shiro（二）
date: 2018-03-30 12:04:35
categories:  Spring
---

在上一篇笔记 SpringBoot-集成Shiro（一）中, 已经在 SpringBoot 中顺利的集成和测试了 Shiro 的使用, 这一篇介绍一下在 Thymeleaf 模板引擎里面使用 Shiro 权限标签和使用 ajax 请求时没有权限或者没有认证返回 JSON 数据而不是跳转到指定的页面。因为这些配置已经实际的项目中成功的使用了, 因此笔记里面只说明配置和使用的过程, 就不详细的测试了。

# Thymeleaf 模板引擎里面使用 Shiro 标签

首先需要引入相关的依赖包, 在 pom.xml 中添加如下历来文件：

``` xml
<!-- Thymeleaf 支持 Shiro 标签 -->
<dependency>
    <groupId>com.github.theborakompanioni</groupId>
    <artifactId>thymeleaf-extras-shiro</artifactId>
    <version>1.2.1</version>
</dependency>
```

<!-- more -->

接着需要在 ShiroConfig 文件中添加如下的 Bean：

``` java
/**
 * 模板文件使用 Shiro 标签时使用
 */
@Bean
public ShiroDialect shiroDialect() {
return new ShiroDialect();
}
```

然后在 html 页面里面使用 Shiro 标签, 如下：

``` html
<div id="content">
    <span shiro:hasRole="admin">模板文件</span>
    <button id="test" shiro:hasPermission="option">测试</button>
</div>
```

在登录成功后, 这个账号如果是 [admin] 角色和有 [option] 权限的时候, span 便签和 button 标签包含的内容才会显示出来。

# 登录认证和权限认证失败返回 JSON 数据

Shiro 登录认证事变和权限认证失败会跳转到指定的页面, 在前后端分离的项目中我们希望的是失败后不是跳转到指定的页面而是返回 JSON 数据, 让前端根据返回的值做出相应的处理。实现这种功能需要使用自定义的过滤器来实现, 主要需要实现三个过滤器, AdviceFilter过滤器和普通过滤器的功能类似, 在请求时判断用户是否登录了, 如果没有登录, 返回 JSON 数据。PermissionsAuthorizationFilter 过滤器的 onAccessDenied 方法在权限认证失败的时候调用, 可以在这里返回 JSON 数据, RolesAuthorizationFilter 和 PermissionsAuthorizationFilter 类似,  相关的代码如下：

继承 AdviceFilter 的 ShiroLoginFilter：

``` java
@Log4j2
public class ShiroLoginFilter extends AdviceFilter {
    @Override
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        String accountId = (String) httpServletRequest.getSession().getAttribute("accountId");
        // 没有登录并且访问的路径不是 "/login" 就返回 token 过期的 JSON 数据
        if (null == accountId &&
                !StringUtils.contains(httpServletRequest.getRequestURI(), "/login") &&
                !StringUtils.contains(httpServletRequest.getRequestURI(), "/register")) {
            log.warn("requestURI : " +  httpServletRequest.getRequestURI());
            ResultEntity resultEntity = new ResultEntity().setTokenOutTime();
            httpServletResponse.setCharacterEncoding("UTF-8");
            httpServletResponse.setContentType("application/json");
            httpServletResponse.getWriter().write(JSONObject.toJSONString(resultEntity.getResult()));
            return false;
        }
        return true;
    }
}
```

继承 PermissionsAuthorizationFilter 的 ShiroPermissionFilter：

``` java
public class ShiroPermissionFilter extends PermissionsAuthorizationFilter {
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws IOException {
        HttpServletRequest httpServletRequest = (HttpServletRequest) request;
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        ResultEntity resultEntity = new ResultEntity().setNoPermissions();
        httpServletResponse.setCharacterEncoding("UTF-8");
        httpServletResponse.setContentType("application/json");
        httpServletResponse.getWriter().write(JSONObject.toJSONString(resultEntity.getResult()));
        return false;
    }
}
```

继承 RolesAuthorizationFilter 的 RolesAuthorizationFilter：

``` java
public class ShiroRoleFilter extends RolesAuthorizationFilter {
    @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws IOException {
        HttpServletResponse httpServletResponse = (HttpServletResponse) response;
        ResultEntity resultEntity = new ResultEntity().setNoPermissions();
        httpServletResponse.setCharacterEncoding("UTF-8");
        httpServletResponse.setContentType("application/json");
        httpServletResponse.getWriter().write(JSONObject.toJSONString(resultEntity.getResult()));
        return false;
    }
}
```

然后在 ShiroConfig 类中去设置这个过滤器：

``` java
/**
 * 未登录前拦截, 直接返回过期 JSON 数据, 不跳转到登录页面
 */
@Bean(name = "shiroLoginFilter")
public ShiroLoginFilter shiroLoginFilter(){
    return new ShiroLoginFilter();
}

/**
 * 权限认证失败后拦截, 直接返回没有权限 JSON 数据, 不跳转到没有权限页面
 */
@Bean(name = "shiroPermissionFilter")
public ShiroPermissionFilter shiroPermissionFilter(){
    return new ShiroPermissionFilter();
}

/**
 * 角色权限认证失败后拦截, 直接返回没有权限 JSON 数据, 不跳转到没有权限页面
 */
@Bean(name = "shiroRoleFilter")
public ShiroRoleFilter shiroRoleFilter(){
    return new ShiroRoleFilter();
}

@Bean
public ShiroFilterFactoryBean shiroFilter(@Qualifier("securityManager") SecurityManager securityManager) {
    ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();

    // 必须设置 SecurityManager
    shiroFilterFactoryBean.setSecurityManager(securityManager);

    // 设置自定义的过滤器
    Map<String, Filter> filterMap = new LinkedHashMap<>();
    filterMap.put("shiroLoginFilter", shiroLoginFilter());
    filterMap.put("shiroPermissionFilter", shiroPermissionFilter());
    filterMap.put("shiroRoleFilter", shiroRoleFilter());
    shiroFilterFactoryBean.setFilters(filterMap);

    Map<String, String> map = new LinkedHashMap<>();
    // 配置退出的过滤器
    map.put("/logout", "logout");
    // 该 URL 不需要认证 (登录) 即可访问
    map.put("/login/message/system", "anon");
    map.put("/register/message/system/account", "anon");
    map.put("/app/login", "anon");

    // 静态资源, img, css, js
    map.put("/css/**", "anon");
    map.put("/img/**", "anon");
    map.put("/js/**", "anon");
    // 除去以上之外, 所有 URL 都需要登录认证才能访问, 这个需要放在最后
    map.put("/**", "authc");

    // 没有认证时将会调用这个接口, 或者打开这个页面
    shiroFilterFactoryBean.setLoginUrl("/login");
    // 未授权时的跳转页面和调用的接口
    shiroFilterFactoryBean.setUnauthorizedUrl("/unauthorized");

    shiroFilterFactoryBean.setFilterChainDefinitionMap(map);
    return shiroFilterFactoryBean;
}
```

这样就可以在没有认证和没有权限的时候不会跳转到我们指定的页面去而是返回 JSON 数据了, 我们也可以在自定的过滤器里面机上判断, 如果是 ajax 请求就返回 JSON 数据, 不是就重定向到我们指定的页面去, 相关的参考代码如下：

``` java
if (null == sysUser && !StringUtils.contains(httpServletRequest.getRequestURI(), "/login")) {
    String requestedWith = httpServletRequest.getHeader("X-Requested-With");
    if (StringUtils.isNotEmpty(requestedWith) && StringUtils.equals(requestedWith, "XMLHttpRequest")) { //如果是ajax返回指定数据
        ResponseHeader responseHeader = new ResponseHeader();
        responseHeader.setResponse(ResponseHeader.SC_MOVED_TEMPORARILY, null);
        httpServletResponse.setCharacterEncoding("UTF-8");
        httpServletResponse.setContentType("application/json");
        httpServletResponse.getWriter().write(JSONObject.toJSONString(responseHeader));
        return false;
    } else { //不是ajax进行重定向处理
        httpServletResponse.sendRedirect("/login/local");
        return false;
    }
}
```