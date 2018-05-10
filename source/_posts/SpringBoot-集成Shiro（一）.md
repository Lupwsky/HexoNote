---
title: SpringBoot-集成Shiro（一）
date: 2018-03-29 14:51:13
categories:  SpringBoot
---

Apache Shiro 是 Java 的一个安全框架, 目前, 使用 Apache Shiro 的人越来越多, 因为它相对 Spring Security 比较简单, 虽然功能没有后者强大。Shiro 可以非常容易的开发出足够好的应用, 其不仅可以用在 JavaSE 环境, 也可以用在 JavaEE 环境, Shiro 可以帮助我们完成：认证、授权、加密、会话管理、与Web集成、缓存等。这篇学习笔记主要简单的说明如何在 SpringBoot 中如何集成 Shiro。

# 添加依赖

在 pom.xml 文件中添加 shiro-spring 依赖, 如下：
<!-- more -->

``` xml
<!-- spring-shiro -->
<dependency>
    <groupId>org.apache.shiro</groupId>
    <artifactId>shiro-spring</artifactId>
    <version>1.4.0</version>
</dependency>
```

# 继承 AuthorizingRealm, 实现登录验证和权限管理

Shiro 不会去维护用户、维护权限, 需要我们自己去设计实现, 然后通过相应的接口注入即可。Shiro 两个重要的功能就是登录认证和权限认证, 通常我们只需要继承 AuthorizingRealm 类, 实现 doGetAuthenticationInfo (登录认证) 和 doGetAuthenticationInfo (权限认证) 即可。

``` java
@Component
@Log4j2
public class CommonRealm extends AuthorizingRealm {

    /**
     * 登录认证
     *
     * @param authenticationToken authenticationToken
     * @return authenticationInfo
     * @throws AuthenticationException authenticationException
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        // 这里为了测试没有通过数据来判断用户是否存在、登录是否成功, 直接返回登录成功
        String username = (String) authenticationToken.getPrincipal();
        String password = new String((char[])authenticationToken.getCredentials());
        log.warn("username : " + username);
        log.warn("password : " + password);
        return new SimpleAuthenticationInfo(username, password, getName());
    }


    /**
     * 权限认证
     *
     * @param principalCollection principalCollection
     * @return authorizationInfo
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        String username = (String) principalCollection.getPrimaryPrincipal();
        log.warn("username2 : " + username);
        SimpleAuthorizationInfo info = new SimpleAuthorizationInfo();
        // 添加角色
        info.addRole("admin");
        // 添加权限
        info.addStringPermission("perm_a");
        info.addStringPermission("perm_b");
        return info;
    }
}
```

# 配置 Shiro

``` java
@Configuration
public class ShiroConfig  {

    @Bean(name = "commonRealm")
    public CommonRealm getCommonRealm() {
        return new CommonRealm();
    }

    @Bean(name = "securityManager")
    public SecurityManager securityManager(@Qualifier("commonRealm") CommonRealm commonRealm) {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(commonRealm);
        return securityManager;
    }

    @Bean
    public ShiroFilterFactoryBean shiroFilter(@Qualifier("securityManager") SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();

        // 必须设置 SecurityManager
        shiroFilterFactoryBean.setSecurityManager(securityManager);

        // 拦截器
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

        // 没有认证时将会打开这个页面
        shiroFilterFactoryBean.setLoginUrl("/login");

        // 没有权限时跳转页面和调用的接口
        shiroFilterFactoryBean.setUnauthorizedUrl("/unauthorized");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(map);
        return shiroFilterFactoryBean;
    }
}
```

# 使用和测试

在登录接口添加 Shiro 登录相关的代码, 登录成功后可以获取 Session 用于保存用户数据。

``` java
// Shiro 登录
Subject currentUser = SecurityUtils.getSubject();
if (!currentUser.isAuthenticated()) {
    UsernamePasswordToken token = new UsernamePasswordToken(username, password);
    currentUser.login(token);
}

// 使用 Session 保存用户信息
Session session = currentUser.getSession();
session.setAttribute("accountId", userInfo.getString("id"));
session.setTimeout(24 * 3600 * 1000);
```

经过上面的步骤就可以简单的使用 Shiro 来进行登录认证和权限管理了, 让我们来简单的测试一下, 创建一个访问的页面的 URL, 使用注解的方式表明这个访问 URL 需要 add 权限, 代码如下。

``` java
@Controller
public class ViewRouterController {

    @RequiresPermissions("add")
    @RequestMapping(value = "/tdx/template")
    public String indexPage(){
        return "template";
    }
}
```

接着创建两个页面, 一个是登录页面, 一个是没有权限时的页面, 分别对应配置 Shiro 里面设置的没有登录时的跳转页面和没有权限时跳转时的页面。

没有登录时的跳转页面的代码：

``` html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org" xmlns:shiro="http://www.pollix.at/thymeleaf/shiro">
<head>
    <meta http-equiv="X-UA-Compatible" content="IE=Edge, chrome=1">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>
    <meta charset="UTF-8"/>
    <title>模板文件</title>
    <script type="text/javascript" th:src="@{/js/jquery-3.3.1.min.js}"></script>
    <script type="text/javascript" th:src="@{/js/common-method.js}"></script>
</head>

<body>
<div id="content">
    <input id="username" title="账号" value="15172457742"/>
    <input id="password" title="密码" value="123456"/>
    <button id="login">登录</button>
</div>
</body>

<script>
    $(document).ready(function () {
        $("#login").on("click", function () {
            var url = "/login/message/system";
            var username = $("#username").val();
            var password = $("#password").val();
            var params = {"username" : username, "password" : password};
            // common-method.js 里面的 obtainDataFromServer 是对 ajax 请求的一个简单封装
            // 这里可以直接使用 ajax 请求。
            CommonMethod.obtainDataFromServer(url, params, "json", success, null, null, null);
        });

        function success(data) {
            alert("登录成功");
            console.log(data);
            location.href = "/tdx/template"
        }
    });
</script>
</html>
```

没有权限时跳转的页面的代码：

``` html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta http-equiv="X-UA-Compatible" content="IE=Edge, chrome=1">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no"/>
    <meta charset="UTF-8"/>
    <title>模板文件</title>
</head>

<body>
<div id="content">
    没有权限
</div>
</body>
</html>
```

当我们通过访问 `http://localhost:8182/tdx/template` 访问这个页面的时候, 由于我们没有登录, 会直接跳转到登录页面。当我们登录后再访问这个链接, 就可以跳转到这个页面了。

** 但是要注意, 这里是不对的, 我们访问的这个 URL 需要 [add] 权限才能访问, 我们在写 Relam 时, 并没有给 [add] 权限给任何的用户, 但是却可以访问, 后台也没有生效和出现异常, 为什么呢。我们需要在配置 Shiro 时开启 Shiro 的注解支持, 在 ShiroConfig 类里面添加如下代码。**

``` java
/**
 * 开启了 Shiro 注解支持
 */
@Bean
public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor(SecurityManager securityManager){
    AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
    authorizationAttributeSourceAdvisor.setSecurityManager(securityManager);
    return authorizationAttributeSourceAdvisor;
}

@Bean
@ConditionalOnMissingBean
public DefaultAdvisorAutoProxyCreator defaultAdvisorAutoProxyCreator() {
    DefaultAdvisorAutoProxyCreator defaultAAP = new DefaultAdvisorAutoProxyCreator();
    defaultAAP.setProxyTargetClass(true);
    return defaultAAP;
}
```

开启了 Shiro 注解支持后, 再次访问页面, 会发现后台出现了异常, 提示当前用户没有 [add] 权限。

``` xml
org.apache.shiro.authz.UnauthorizedException: Subject does not have permission [add]
```

** 这里也需要注意, 没有权限, 虽然出现了异常, 但是却没有跳转到指定的 unauthorized 页面去。看看源码后, 配置的角色拦截器必须是 满足     filterinstanceof AuthorizationFilter 无权页面配置才会生效, 定义的filter必须满足 filter instanceof AuthorizationFilter, 只有 perms, roles, ssl, rest, port 才是属于 AuthorizationFilter, 而 anon, authcBasic, auchc, user 是 AuthenticationFilter, 所以 unauthorizedUrl设置后页面不跳转；shiro 注解模式下, 登录失败或者是没有权限都是抛出异常, 并且默认的没有对异常做处理, 因此解决方法要么就使用 perms, roles, ssl, rest, port, 或者是按照 springmvc 的处理方式, 专门配置一个异常处理, 例如设置一个全局的异常进行捕捉, 当然是用自定义的过滤器也是可以实现的, 是用过滤器的方式见下一篇文章**

Shiro 源码如下：

``` java
private void applyUnauthorizedUrlIfNecessary(Filter filter) {
    String unauthorizedUrl = getUnauthorizedUrl();
    if (StringUtils.hasText(unauthorizedUrl) && (filter instanceof AuthorizationFilter</span></strong></em>)) {
        AuthorizationFilter authzFilter = (AuthorizationFilter) filter;
        //only apply the unauthorizedUrl if they haven't explicitly configured one already:
        String existingUnauthorizedUrl = authzFilter.getUnauthorizedUrl();
        if (existingUnauthorizedUrl == null) {
            authzFilter.setUnauthorizedUrl(unauthorizedUrl);
        }
    }
}
```

我们可以添加一个全局的异常捕获, 通过 JSON 数据通知前台没有权限：

``` java
@ControllerAdvice
@Log4j2
public class GlobalDefaultExceptionHandler {

    /**
     * 没有权限时的异常
     *
     * @param ex AuthorizationException 异常
     * @return 接口返回的数据
     */
    @ExceptionHandler(value = AuthorizationException.class)
    @ResponseBody
    public Map<String, Object> authorizationException(AuthorizationException ex) {
        log.error(ex.getMessage());
        return new ResultEntity().setNoPermissions().getResult();
    }
}
```