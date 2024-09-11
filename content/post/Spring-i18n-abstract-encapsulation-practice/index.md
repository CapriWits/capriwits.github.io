---
title: "Spring i18n 抽象封装实践"
slug: "spring-i18n-abstract-encapsulation-practice"
date: 2024-09-11T15:03:29+08:00
author: CapriWits
image: spring-cover.png
tags:
- Java
- Spring
categories:
- Backend
hidden: false
math: true
comments: true
draft: false
---

<!-- TOC -->
* [Spring i18n 抽象封装实践](#spring-i18n-抽象封装实践)
  * [Spring 国际化基本使用](#spring-国际化基本使用)
  * [抽象国际化功能](#抽象国际化功能)
    * [定义接口](#定义接口)
    * [简单 i18n 消息](#简单-i18n-消息)
    * [复杂 i18n 消息](#复杂-i18n-消息)
    * [i18n 工具类](#i18n-工具类)
    * [使用](#使用)
  * [总结](#总结)
<!-- TOC -->

# Spring i18n 抽象封装实践

![](https://img.shields.io/badge/JDK-1.8-B3CAD8)  
![](https://img.shields.io/badge/spring--web-5.3.31-DBD4C6)

## Spring 国际化基本使用

1. 首先 maven 引入 `spring-web` 的依赖
2. 在 `resources` 目录下，创建一个放国际化文本的文件夹，例如 `/resources/i18n/`，然后修改 `resources/application.yml` 配置文件

```yaml
spring:
  messages:
    basename: i18n/messages
    encoding: UTF-8
    useCodeAsDefaultMessage: true
```

`basename` 指定翻译文本的前缀，可以带相对路径，即 i18n 文件夹。 `messages` 是文本前缀，文本名示例：`messages.properties`（默认文本，`messages_en.properties`（英文，`messages_zh.properties`（简体中文

3. 文本文件格式

有两种，一种直接 k-v 结构翻译，`key = value`，翻译时传 key，直接返回 value 的字符串；

另一种用占位符大括号 `{0} {1} {2}…`

- `messages_en_US.properties`：

```bash
greeting=Hello, {0}!
```

- `messages_zh_CN.properties`：

```bash
greeting=你好，{0}！
```

4. 使用 `MessageSource` 获取 i18n 文本

- 前端传 url 参数 lang，指定语言
- 当用户访问 `/greet?name=John&lang=zh_CN` 时，返回的消息将是 `你好，John！`；当用户访问 `/greet?name=John&lang=en_US` 时，返回的消息将是 `Hello, John!`。

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.Locale;

@RestController
public class GreetingController {

    private final GreetingService greetingService;

    public GreetingController(GreetingService greetingService) {
        this.greetingService = greetingService;
    }

    @GetMapping("/greet")
    public String greet(@RequestParam String name, @RequestParam String lang) {
        Locale locale = new Locale(lang.split("_")[0], lang.split("_")[1]);
        return greetingService.getGreeting(name, locale);
    }
}
```

当然，为了前端方便也可以统一用 HTTP Header，然后后端获取创建 `java.util.Locale` 对象

```java
@RequestHeader(value = "Accept-Language", defaultValue = "zh-CN") String lang
```

- 后端使用 `org.springframework.context.MessageSource#getMessage(java.lang.String, java.lang.Object[], java.util.Locale)` 翻译即可
- 参数分别为：文本 key；渲染参数数组（按顺序，地区对象

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.MessageSource;
import org.springframework.stereotype.Service;

import java.util.Locale;

@Service
public class GreetingService {

    @Autowired
    private MessageSource messageSource;

    public String getGreeting(String name, Locale locale) {
        return messageSource.getMessage("greeting", new Object[]{name}, locale);
    }
}
```

## 抽象国际化功能

以上做法是直接替换掉国际化的内容，如果内容不多，且没有一些嵌套的文本，可以直接替换。

嵌套文本指的是，`key1` 对应 `value` 里面，其中一个占位符的内容也是动态生成，拼接而成的。假如 `{1}` 也是一个国际化文本，对应另一个 key2 的内容，这中嵌套就需要先翻译 `{1}`，再渲染 `value`。

可以写一个工具类，抽象 i18n 渲染功能。

### 定义接口

- `I18nKey` 只有一个方法，获取 key，实现类是一个枚举
- `I18nMessage` 是一条 i18n 信息，除了获取 key，还有获取渲染值列表；类型上界限定为 `I18nKey` 实现类；`getValues()` 由于渲染的值对象不止 `String`，还可能是还是一个 `I18nMessage`，需要递归地渲染，所以获取列表泛型只限制 Object

```java
/**
 * I18n 配置文件 key
 */
@FunctionalInterface
public interface I18nKey {
    String getKey();
}

/**
 * I18n 基本信息, 包括配置文件 key 和对应模板顺序的待渲染值
 *
 * @param <K> I18n 配置文件 key 类型
 * @see I18nKey
 */
public interface I18nMessage<K extends I18nKey> {
    /**
     * 获取 I18n 配置文件 key
     *
     * @return I18n 配置文件 key
     */
    K getKey();

    /**
     * 获取 I18n 配置文件 key 对应模板顺序的待渲染值
     *
     * @return I18n 配置文件 key 对应模板顺序的待渲染值
     */
    List<Object> getValues();
}
```

### 简单 i18n 消息

- 简单 i18n key，对应简单 i18n message，即**简单 k-v 结构**，无渲染参数
- 实现 `I18nKey` 接口，是一个**枚举类**
    - 下面是审批示例，设计三类，审批添加、操作和撤销的 i18n key
    - 成员变量只有 `String key`，需要实现接口方法 `getKey`
    - 为了实现 key 还原枚举对象，实现 `fromKey` 方法

```java
/**
 * 简单 i18n 配置文件 key 枚举, 直接获取 i18n 字符串, 无渲染
 *
 * @see I18nKey
 * @see SimpleI18nMessage
 */
@AllArgsConstructor
@Slf4j
public enum SimpleI18nKey implements I18nKey {
    /**
     * 审批添加操作日志
     */
    APPROVAL_ADD_TYPENAME("opLog.approvalAdd.typeName"),
    APPROVAL_ADD_FIELDS_DESCRIPTION("opLog.approvalAdd.fields.description"),

    /**
     * 操作审核操作日志
     */
    APPROVAL_REVIEW_TYPENAME("opLog.approvalReview.typeName"),
    APPROVAL_REVIEW_FIELDS_UNSOLVED("opLog.approvalReview.fields.unsolved"),
    APPROVAL_REVIEW_FIELDS_APPROVED("opLog.approvalReview.fields.approved"),
    APPROVAL_REVIEW_FIELDS_REJECTED("opLog.approvalReview.fields.rejected"),

    /**
     * 撤销审批操作日志
     */
    APPROVAL_WITHDRAW_TYPENAME("opLog.approvalWithdraw.typeName"),
    APPROVAL_WITHDRAW_FIELDS_AUTOWITHDRAW("opLog.approvalWithdraw.fields.autoWithdraw"),
    ;

    public final String key;

    @Override
    public String getKey() {
        return key;
    }

    /**
     * 根据给定的字符串查找对应的枚举对象, 主要应用在操作日志从数据库取出后还原 Key 枚举对象
     *
     * @param key i18n 配置文件 key 字符串
     * @return 匹配的枚举对象，如果没有找到则返回 {@code null}
     */
    public static SimpleI18nKey fromKey(String key) {
        for (SimpleI18nKey i18nKey : SimpleI18nKey.values()) {
            if (i18nKey.getKey().equals(key)) {
                return i18nKey;
            }
        }
        log.error("I18n key not found: [{}]", key);
        return null;
    }
}
```

- 简单 i18n 消息体
    - 成员变量是泛型中同类型的 `SimpleI18nKey` 枚举对象
    - 实现 `I18nMessage` 接口方法
        - 获取 key，直接返回成员变量
        - 获取渲染参数列表，简单 kv 消息，无渲染参数，返回 `null`

```java
/**
 * 简单 i18n 消息体, 无需渲染(无 args), 根据 key 直接获取 i18n 内容
 *
 * @see SimpleI18nKey
 */
@AllArgsConstructor
public class SimpleI18nMessage implements I18nMessage<SimpleI18nKey> {

    public final SimpleI18nKey key;

    @Override
    public SimpleI18nKey getKey() {
        return key;
    }

    /**
     * 无需渲染(无 args)
     *
     * @return {@code null}
     */
    @Override
    public List<Object> getValues() {
        return null;
    }
}
```

### 复杂 i18n 消息

复杂 i18n 消息，即带有渲染的 i18n 消息，渲染值可能带有 i18n message 需要递归渲染

以添加操作审批日志为例， `I18nKey` 实现类同样是一个枚举，只有 `String key` 一个成员变量；

`I18nMessage` 实现类，泛型 `ApprovalAddAdditionOtherI18nKey` 限制 i18n key 类型，待渲染信息有两个，放在成员变量位置，同 key，由有参构造方法统一设置；可以看到 `description` 变量对应第二个参数 `{1}` 同样是一个 `I18nMessage`，该 i18n message 泛型限制为简单 i18n key；获取渲染参数 `getValues()` 时，按顺序传入 `approvalId` 和 `description`

```java
/**
 * 添加审批操作日志 i18n 配置文件 key
 *
 * @see ApprovalAddAdditionOtherI18nMsg
 */
@AllArgsConstructor
public enum ApprovalAddAdditionOtherI18nKey implements I18nKey {
    ADDITION_OTHER("opLog.approvalAdd.addition.other");

    public final String key;

    @Override
    public String getKey() {
        return key;
    }
}

@AllArgsConstructor
public class ApprovalAddAdditionOtherI18nMsg implements I18nMessage<ApprovalAddAdditionOtherI18nKey> {
    // 发起未知类型审批；审批ID = {0}；{1}
    public final ApprovalAddAdditionOtherI18nKey key;
    public final Integer approvalId;
    public final I18nMessage<SimpleI18nKey> description;

    @Override
    public ApprovalAddAdditionOtherI18nKey getKey() {
        return key;
    }

    @Override
    public List<Object> getValues() {
        return Arrays.asList(approvalId, description);
    }
}
```

### i18n 工具类

```java
/**
 * I18n 工具类 <p>
 * 提供两种方式渲染国际化信息
 * <ol>
 *     <li>根据 key 直接渲染国际化消息</li> {@link #processI18nMessage(String, Locale, Object...)}
 *     <li>根据 {@link I18nMessage} 实现类渲染国际化消息</li> {@link #processI18nMessage(I18nMessage, Locale)}
 * </ol>
 *
 * @see I18nMessage
 * @see I18nKey
 * @see SimpleI18nMessage
 */
@Slf4j
@Component("i18n-utils")
public class I18nUtils {

    private final MessageSource messageSource;

    @Autowired
    public I18nUtils(MessageSource messageSource) {
        this.messageSource = messageSource;
    }

    /**
     * 直接根据 key 获取国际化消息
     *
     * @param key    i18n 配置文件 key
     * @param locale {@link Locale} 语言环境
     * @param args   按模板顺序的参数列表, 可为 {@code null}
     * @return 渲染后国际化消息
     */
    public String processI18nMessage(String key, Locale locale, Object... args) {
        // 为避免渲染模版出现多余分隔符, 如 Integer 增加千位逗号分隔符, 统一 String 进行渲染
        String[] argsArray = Arrays.stream(args).map(String::valueOf).toArray(String[]::new);
        return getMessage(key, locale, argsArray);
    }

    /**
     * 直接根据 key 获取国际化消息, 渲染失败时返回默认消息
     *
     * @param key            i18n 配置文件 key
     * @param defaultMessage 默认消息, 可为 {@code null}
     * @param locale         {@link Locale} 语言环境
     * @param args           按模板顺序的参数列表, 可为 {@code null}
     * @return 渲染后国际化消息
     */
    public String processI18nMessage(String key, @Nullable String defaultMessage, Locale locale, Object... args) {
        // 为避免渲染模版出现多余分隔符, 如 Integer 增加千位逗号分隔符, 统一 String 进行渲染
        String[] argsArray = Arrays.stream(args).map(String::valueOf).toArray(String[]::new);
        return getMessage(key, locale, defaultMessage, argsArray);
    }

    /**
     * 根据 {@link I18nMessage} 实现类和 {@link Locale} 语言环境获取渲染后的国际化消息
     *
     * @param i18nMessage {@link I18nMessage} 实现类, 国际化消息
     * @param locale      {@link Locale} 语言环境
     * @return 渲染后的国际化消息
     */
    public String processI18nMessage(@NonNull I18nMessage<?> i18nMessage, @NonNull Locale locale) {
        return resolveMessage(i18nMessage, locale);
    }

    /**
     * 处理国际化消息
     *
     * @param i18nMessage {@link I18nMessage} 实现类, 国际化消息
     * @param locale      {@link Locale} 语言环境
     * @return 渲染后的国际化消息
     */
    private String resolveMessage(@NonNull I18nMessage<?> i18nMessage, @NonNull Locale locale) {
        // 直接渲染简单国际化消息
        if (i18nMessage instanceof SimpleI18nMessage) {
            return getMessage(i18nMessage.getKey().getKey(), locale, null);
        }
        String[] args = i18nMessage.getValues().stream()
                .map(value -> value instanceof I18nMessage ? resolveMessage((I18nMessage<?>) value, locale) : value)
                .map(String::valueOf)
                .toArray(String[]::new);
        return getMessage(i18nMessage.getKey().getKey(), locale, args);
    }

    /**
     * 获取国际化消息
     *
     * @param key    i18n 配置文件 key
     * @param locale {@link Locale} 语言环境
     * @param args   按模板顺序的参数列表, 可为 {@code null}
     * @return 渲染后国际化消息; 若未找到消息, 返回空字符串
     */
    private String getMessage(String key, Locale locale, String[] args) {
        String i18nMessage = "";
        try {
            i18nMessage = messageSource.getMessage(key, args, locale);
        } catch (NoSuchMessageException e) {
            log.error("Resolve i18n message [{}] error", key, e);
        }
        return i18nMessage;
    }

    /**
     * 获取国际化消息
     *
     * @param key            i18n 配置文件 key
     * @param locale         {@link Locale} 语言环境
     * @param defaultMessage 渲染失败, 默认消息
     * @param args           按模板顺序的参数列表, 可为 {@code null}
     * @return 渲染后国际化消息; 若未找到消息, 返回默认消息
     */
    private String getMessage(String key, Locale locale, String defaultMessage, String[] args) {
        return messageSource.getMessage(key, args, defaultMessage, locale);
    }
}
```

- 渲染的主要类是通过 `org.springframework.context.MessageSource`
- 渲染的参数 `args` 虽然是 `Object` 数组，不限制类型，但是传入 `Integer` 时，渲染千位以上的值时，会自动带上千分位分隔符，类似 `1,000` ，因此渲染千统一将 `args` 转为 `String` ，保持原数值格式
- `org.springframework.context.MessageSource#getMessage(java.lang.String, java.lang.Object[], java.util.Locale)` 找不到 i18n key，默认会抛出 `NoSuchMessageException` 异常，而不是返回 `null`，需要注意
- `org.springframework.context.MessageSource#getMessage(java.lang.String, java.lang.Object[], java.lang.String, java.util.Locale)` 提供 `defaultMessage` 参数，找不到 i18n key 不抛异常，返回默认字符串
- `processI18nMessage(String key, Locale locale, Object... args)` 参数同 `getMessage`，直接根据字符串 key 获取 i18n 消息，不建议使用，因为实际使用有大量魔法值，即 i18n key 是魔法值字符串，统一用枚举管理便于复用，注释和查看引用。
- `processI18nMessage(@NonNull I18nMessage<?> i18nMessage, @NonNull Locale locale)` 渲染 i18n 信息方法，只需要 i18n message 对象和地区对象 `Locale` ；`I18nMessage` 会带有 key 信息，因此只需要一个对象即可
- `resolveMessage(@NonNull I18nMessage<?> i18nMessage, @NonNull Locale locale)` 处理 i18n message 的核心方法，先判断 i18n message 是否为简单 i18n 消息 `SimpleI18nMessage`，如果是，直接 getMessage；如果不是，需要将参数 args 依次获取，判断 `instanceof I18nMessage` ，是 i18n message 递归处理消息即可，直到所有国际化消息都被渲染

### 使用

```java
public String getShowAddition(Locale lang) {
    I18nUtils i18n = BeanContext.getBean(I18nUtils.class);
    SimpleI18nMessage descMsg = new SimpleI18nMessage(SimpleI18nKey.APPROVAL_ADD_FIELDS_DESCRIPTION);
    return i18n.processI18nMessage(new ApprovalAddAdditionOtherI18nMsg(
                    ApprovalAddAdditionOtherI18nKey.ADDITION_OTHER, approvalId, descMsg)
            , lang);
}
```

- `getShowAddition` 传入 `Locale` 地区对象
- 先用 `org.springframework.beans.factory.BeanFactory#getBean(java.lang.Class<T>)` 获取 i18n 工具类
- 创建简单 i18n 消息，指定其 i18n 枚举 key
- 创建复杂 i18n 消息，先指定复杂 i18n key，然后按渲染顺序传入成员变量，其中 description 是上面创建的简单 i18n 消息

## 总结

以上递归渲染的抽象工具类，在复杂的递归渲染场景可以比较方便地使用，语义和可读性都较高。

如只有简单的 i18n 消息渲染场景，直接使用 `MessageSource` 渲染即可，无需过度封装；注意管理好 i18n key 的魔法值。
