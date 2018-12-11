---
title: Spring-源码阅读之XML验证模式和XmlValidationModeDetector类
date: 2018-11-19 22:18:59
categories:  Spring
---

# XML 的两种验证模式 DTD 和 XSD

XML 文件的验证模式保证了 XML 文件的正确性, 而比较常用的验证模式有两种: DTD 和 XSD, 这里先简单了解下

## DTD

DTD (Document Type Definition) 即文档类型定义, 是一种XML约束模式语言, 是XML文件的验证机制, 属于XML文件组成的一部分, DTD 是一种保证 XML 文档格式正确的有效方法, 可以通过比较 XML 文档和 DTD 文件来看文档是否符合规范, 元素和标签使用是否正确, 要使用 DTD 验证模式的时候需要在 XML 文件的头部声明类似如下, 宏观特点带有一个 DOCTYPE 字符串, Spring 也是判断是否包含 DOCTYPE 字符串来判断是否使用 DTD 验证的:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE note [
  <!ELEMENT note (to,from,heading,body)>
  <!ELEMENT to      (#PCDATA)>
  <!ELEMENT from    (#PCDATA)>
  <!ELEMENT heading (#PCDATA)>
  <!ELEMENT body    (#PCDATA)>
]>
```

<!-- more -->

更多关于 DTD 的知识点见 [DTD 简介 - W3School 在线教程](http://www.w3school.com.cn/dtd/dtd_intro.asp)

## XSD

XML Schema 语言也称作 XML Schema 定义(XML Schema Definition), XML Schema 是基于 XML 的 DTD 替代者, XML Schema 的作用是定义 XML 文档的合法构建模块, 类似 DTD, XML 文档可对 XML Schema 进行引用, 如下是引入了 [http://www.springframework.org/schema/beans/spring-beans.xsd](http://www.springframework.org/schema/beans/spring-beans.xsd) 的一个例子, 此时即使用 XSD 验证,:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
                           http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userInfo" class="com.thread.excutor.UserInfo">
        <property name="name" value="lupengwei"/>
        <property name="email" value="lupengwei@qq.com"/>
    </bean>

</beans>
```

更多关于 XSD 的知识点见 [XML Schema 简介 - W3School 在线教程](http://www.w3school.com.cn/schema/schema_intro.asp)

# Spring 中的 XmlValidationModeDetector 类

XmlValidationModeDetector 类用于检验一个 XML 文件使用的是哪种验证模式, 基本思路是判断是否包含了 DOCTYPE 字符串, 源码解读如下:

```java
public class XmlValidationModeDetector {
    // 没有使用任何验证模式
    public static final int VALIDATION_NONE = 0;

    // 如果不属能辨别一种验证模式会返回这个值, 表示着会自动猜测一种验证模式
    public static final int VALIDATION_AUTO = 1;

    // 如果找到 DOCTYPE 的字符, 表示使用的是 DTD 验证
    public static final int VALIDATION_DTD = 2;

    // 如果没有找到 DOCTYPE 的字符, 表示使用的是 XSD 验证
    public static final int VALIDATION_XSD = 3;

    // 如果使用了 DTD 验证模式, 一定包含了这个字段
    private static final String DOCTYPE = "DOCTYPE";

    // XML 注释开始的标记
    private static final String START_COMMENT = "<!--";

    // XML 注释结束的标记
    private static final String END_COMMENT = "-->";

    // 表示当前解析的数据是不是注释
    private boolean inComment;

    // 检测提供的 XML 文档的验证模式
    public int detectValidationMode(InputStream inputStream) throws IOException {
        try (BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream))) {
            boolean isDtdValidated = false;
            String content;
            while ((content = reader.readLine()) != null) {
                // 判断当前读取的数据是否在注释内
                content = consumeCommentTokens(content);

                // 如果是注释就继续读取下一行的数据
                if (this.inComment || !StringUtils.hasText(content)) {
                    continue;
                }

                // 如果当前解析的数据包含 DOCTYPE 字符串, 说明是 DTD 模式
                if (hasDoctype(content)) {
                    isDtdValidated = true;
                    break;
                }

                // DOCTYPE 是紧跟在 <?xml version="1.0" encoding="UTF-8"?> 语句下面的,
                // 利用这个特点和上面的判断逻辑快速的判断 XML 文件是否使用了 DTD 验证, 这样就不需要将 XML 文件全部遍历一遍了
                if (hasOpeningTag(content)) {
                    break;
                }
            }
            return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
        } catch (CharConversionException ex) {
            // 出现 CharConversionException 异常时返回 VALIDATION_AUTO
            return VALIDATION_AUTO;
        }
    }

    private boolean hasDoctype(String content) {
        return content.contains(DOCTYPE);
    }

    // 如果是注释, 会一直返回 false
    // DOCTYPE 是紧跟在 <?xml version="1.0" encoding="UTF-8"?> 语句下面的
    private boolean hasOpeningTag(String content) {
        if (this.inComment) {
            return false;
        }
        int openTagIndex = content.indexOf('<');
        // 当前数据是 "<" 开头, 并且数据在 XML 中换行了, 含返回 true
        return (openTagIndex > -1 && (content.length() > openTagIndex + 1) &&
                Character.isLetter(content.charAt(openTagIndex + 1)));
    }

    // 判断当前数据是否是在注释里面
    @Nullable
    private String consumeCommentTokens(String line) {
        if (!line.contains(START_COMMENT) && !line.contains(END_COMMENT)) {
            return line;
        }
        String currLine = line;
        while ((currLine = consume(currLine)) != null) {
            if (!this.inComment && !currLine.trim().startsWith(START_COMMENT)) {
                return currLine;
            }
        }
        return null;
    }

    @Nullable
    private String consume(String line) {
        int index = (this.inComment ? endComment(line) : startComment(line));
        return (index == -1 ? null : line.substring(index));
    }

    private int startComment(String line) {
        return commentToken(line, START_COMMENT, true);
    }

    private int endComment(String line) {
        return commentToken(line, END_COMMENT, false);
    }

    private int commentToken(String line, String token, boolean inCommentIfPresent) {
        int index = line.indexOf(token);
        if (index > - 1) {
            this.inComment = inCommentIfPresent;
        }
        return (index == -1 ? index : index + token.length());
    }
}
```