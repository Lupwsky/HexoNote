---
title: Java-JavaAgent技术的基础使用
date: 2019-05-27 14:38:05
categories: Java
---

Java Agent 技术可以在加载类的时候进行拦截并对字节码进行修改, Java Agent 技术可以实现动态的修改代码和替换类的定义而不影响原有程序的逻辑, 例如实现对代码的监控, 捕获代码的执行时间, 添加事务控制等 AOP 功能, 实际的实现方式主要使用在 JDK 1.5 的时候添加的 java.lang.instrument 包中的 API  来实现, 修改字节码通常使用 Javassist 和 ASM 技术实现, 基础的知识和概念在这里就不去记录了, 网上有很多, 主要记录下测试时的使用示例

# 实现 Agent 类

首先创建一个普通的 Java 工程, 使用 Maven 管理 jar 包, Java Agent 以 jar 包的形式部署在 JVM 中, 不能单独启动, 需要依赖于其他的应用一起启动, 根据不同的启动时机, 实现 Agent 类需要实现以下两种方法, 参考 [谈谈 Java Intrumentation 和相关应用](http://fanyilun.me/2017/07/18/%E8%B0%88%E8%B0%88Java%20Intrumentation%E5%92%8C%E7%9B%B8%E5%85%B3%E5%BA%94%E7%94%A8/)

```java
/**
 * 以 VM 参数的形式载入, 其 jar 包的 MANIFEST.MF 文件需要配置属性 Premain-Class
 */
public static void premain(String agentArgs, Instrumentation inst);

/**
 * 以 Attach 的方式载入, 在 Java 程序启动后执行, 其 jar 包的 MANIFEST.MF 文件需要配置属性 Agent-Class
 */
public static void agentmain(String agentArgs, Instrumentation inst);
```

<!-- more -->

这里按照第一种方式举例说明, 首先创建一个 InstrumentationPremainTest 类, 实现 premain() 方法, 如下

```java
@Slf4j
public class InstrumentationPremainTest {

    /**
     * 每一个类加载的时候都会回调此方法, 再通过 ClassFileTransformer 的 transform 方法进行拦截处理
     */
    public static void premain(String agentArgs, Instrumentation inst) {
        inst.addTransformer(new ClassFileTransformerTest());
    }
}
```

# 对加载的类进行拦截处理

对加载的类进行拦截需要实现 ClassFileTransformer 的 transform 方法, 这里实现对 `com.example` (创建另外一个测试项目的时候包名和这里一样即可, 或者更改这里检测包名的代码也是可以的) 包下面的所有类的方法进行拦截, 并使用 Javassist 技术修改字节码, 再方法体的前后各输出一句日志, 具体实现如下:

```java
@Slf4j
public class ClassFileTransformerTest implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined,
                            ProtectionDomain protectionDomain,
                            byte[] classfileBuffer) throws IllegalClassFormatException {
        if (!className.startsWith("com/example")) {
            return null;
        } else {
            log.info("className = {}", className);

            String fullClassName = className.replace("/", ".");
            log.info("fullClassName = {}", fullClassName);

            ClassPool classPool = ClassPool.getDefault();
            log.info("classPool = {}", "classPool");
            CtClass ctClass = null;
            try {
                ctClass = classPool.get(fullClassName);
            } catch (NotFoundException e) {
                // Spring 框架很有可能对我们的类进行扩展生成代理类, 这种情况下会出现找不到对应的 class 文件, 会出现 NotFoundException
                // 例如 com/example/javaagent/JavaagentApplication$$EnhancerBySpringCGLIB$$b02db345
                // com.example.javaagent.JavaagentApplication$$EnhancerBySpringCGLIB$$b02db345
                // 可以直接读取 classfileBuffer, 直接通过类的字节码创建CtClass对象, 或者简单点直接通过判断是否包含有 $ 符号直接判定
                try (ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(classfileBuffer)) {
                    ctClass = classPool.makeClass(byteArrayInputStream);
                } catch (IOException ex) {
                    log.error("exception = {}, error = {}", "IOException", ex);
                }
            }

            // 接口类不做处理
            if (ctClass == null || ctClass.isInterface()) {
                return null;
            }

            // 在原方法体前面和后面添加代码
            try {
                Object[] annotations = ctClass.getAnnotations();
                log.info("className = {}, annotations = {}", className, Arrays.toString(annotations));
                CtMethod[] methods = ctClass.getDeclaredMethods();
                for (CtMethod ctMethod : methods) {
                    log.info("className = {}, methodName = {}", className, ctMethod.getName());
                    ctMethod.insertBefore("System.out.println(\"" + className + " transform begin...\");");
                    ctMethod.insertAfter("System.out.println(\"" + className + " transform end...\");");
                }
                return ctClass.toBytecode();
            } catch (CannotCompileException | IOException e) {
                log.error("exception = {}, error = {}", "CannotCompileException", e);
                return null;
            } catch (ClassNotFoundException e) {
                log.warn("Slf4j 类不存在");
                return null;
            } finally {
                // 清除缓存
                ctClass.detach();
            }
        }
    }
}
```

# 添加的依赖

测试中使用的依赖

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.20</version>
</dependency>

<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.25</version>
</dependency>

<dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.25.0-GA</version>
</dependency>
```

# 打包依赖

由于 Java Agent 需要打包成 jar 包, 使用了 maven-assembly-plugin 插件进行打包, 同时需要在项目的 MANIFEST.MF 需要指定 Premain-Class 的路径, 这个文件可以自己手动去添加, 这里统一使用 maven-assembly-plugin 插件在打包的时候去生成, 同时这个项目依赖的第三方 jar 要一起打包进去 Java Agent jar 包才能正确的运行, 需要为插件添加 `<descriptorRefs><descriptorRef>jar-with-dependencies</descriptorRef></descriptorRefs>` 配置, 将依赖的 jar 包添加进去, 全部插件的配置如下

```xml
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-assembly-plugin</artifactId>
        <configuration>
            <!-- get all project dependencies -->
            <descriptorRefs>
                <descriptorRef>jar-with-dependencies</descriptorRef>
            </descriptorRefs>

            <!-- MainClass in mainfest make a executable jar -->
            <archive>
                <manifestEntries>
                    <Project-name>${project.name}</Project-name>
                    <Project-version>${project.version}</Project-version>
                    <Premain-Class>com.example.javaagent.InstrumentationPremainTest</Premain-Class>
                    <Can-Redefine-Classes>true</Can-Redefine-Classes>
                    <Can-Retransform-Classes>true</Can-Retransform-Classes>
                </manifestEntries>
            </archive>
        </configuration>
        <executions>
            <!-- bind to the packaging phase -->
            <execution>
                <id>make-assembly</id>
                <phase>package</phase>
                <goals>
                    <goal>single</goal>
                </goals>
            </execution>
        </executions>
    </plugin>
</plugins>
```

# 查看 JAR 包中的 MANIFEST.MF 文件

打包好成 jar 包后, 可以去查看 MANIFEST.MF 文件是否写入了 Premain-Class 信息, 如下

```text
Manifest-Version: 1.0
Archiver-Version: Plexus Archiver
Created-By: Apache Maven
Built-By: lpw
Build-Jdk: 1.8.0_101
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: com.example.javaagent.InstrumentationPremainTest
Project-name: javaagentpackage
Project-version: 1.0
```

# 开始测试

创建另外一个基于 Spring 框架的 JavaWeb 项目, 为了测试, 包名设置为 `com.example.javaagent`, 在 JVM 启动参数后面添加 `-javaanget:/path/xxx.jar` 指定使用 Java Agent 生成的 jar 包, 在 windows 上测试的添加的路径如下

```text
-javaagent:D:/Work/hello-grpc/javaagentpackage/target/javaagentpackage-jar-with-dependencies.jar
```

创建一个测试类, 如下

```java
@Slf4j
@RestController
public class JavaAgentController {
    @GetMapping("/java/agent/test")
    public void javaAgentTest() {
        log.info("java agent test...");
    }
}
```

启动项目, 开始访问 `/java/agent/test`, 在控制台就能看到如下日志, 说明 Java Agent 对依赖项目中的类的修改是生效的, 输出如下

```text
com/example/javaagent/JavaAgentController transform begin...
[nio-8080-exec-1] c.example.javaagent.JavaAgentController  : java agent test...
com/example/javaagent/JavaAgentController transform end...
```

# Java Agent 应用范围

Java Agent 技术应用范围有很多, 常用于系统的监控等诊断工具, 如 Greys (以 Attach 方式载入), 阿里开源的 TProfiler (以 VM 参数的形式载入) 和 Arthas (基于 Greys 二次开发, 以 Attach 方式载入) 和 MyPerf4j (以 VM 参数的形式载入) 等诊断工具, 也应用于热部署领域, 如 JRebel, Spring Loader 等

# 参考资料

* [Java Instrumentation](https://blog.csdn.net/productshop/article/details/50623626)
* [谈谈 Java Intrumentation 和相关应用](http://fanyilun.me/2017/07/18/%E8%B0%88%E8%B0%88Java%20Intrumentation%E5%92%8C%E7%9B%B8%E5%85%B3%E5%BA%94%E7%94%A8/)
* [基于 Java Agent 与 Javassist 实现零侵入 AOP](https://ouyblog.com/2019/03/%E5%9F%BA%E4%BA%8EJava-Agent%E4%B8%8EJavassist%E5%AE%9E%E7%8E%B0%E9%9B%B6%E4%BE%B5%E5%85%A5AOP)
* [Javassist 使用指南(一)](http://erhu.party/2016/11/17/javassist-turorial-1/)
* [Javassist 使用指南(二)](http://erhu.party/2016/11/23/javassist-turorial-2/)
* [Javassist 使用指南(三)](http://erhu.party/2016/11/23/javassist-tutorial-3/)