---
title: "单测和打包都通过，Spring 为什么还在启动时寻找无参构造器"
date: 2026-07-24
updated: 2026-07-24
status: published
category: problems
tags:
  - Spring Boot
  - Dependency Injection
  - Testing
  - Deployment
  - Constructor
summary: "复盘一个只在发布启动阶段出现的 Bean 实例化故障：组件增加第二个构造器后，Spring 无法判断注入入口并尝试寻找无参构造器；普通单测和 Maven 打包为何漏过，以及如何用容器级测试补上门禁。"
---

# 单测和打包都通过，Spring 为什么还在启动时寻找无参构造器

## 问题现象

一次 Spring Boot 发布中，前端构建、Java 编译、单元测试和可执行 JAR 打包全部成功。新版本切换后，服务却始终无法通过就绪检查，并在启动日志中出现：

```text
BeanInstantiationException: Failed to instantiate [SomeService]:
No default constructor found

Caused by: NoSuchMethodException: SomeService.<init>()
```

发布脚本随后按设计回滚到旧版本。

这类日志很容易让人误以为“Spring Bean 必须有无参构造器”。实际并不是。单一构造器的 Spring 组件通常不需要写 `@Autowired`，也不需要无参构造器。真正触发问题的是：组件后来增加了第二个构造器，却没有明确告诉容器哪个才是依赖注入入口。

## 代码是怎样变得有歧义的

最初的服务只有一个配置构造器：

```java
@Service
class SourceService {
    SourceService(
            @Value("${app.endpoint:}") String endpoint,
            @Value("${app.bucket:}") String bucket
    ) {
        // ...
    }
}
```

Spring 4.3 之后，如果 Bean 只有一个构造器，可以自动使用它做构造器注入，因此省略 `@Autowired` 没有问题。

为了让单元测试更方便，代码又增加了一个接收现成客户端的构造器：

```java
SourceService(Client client, String bucket) {
    // 测试直接传 mock client
}
```

此时类中有两个都没有标注的构造器。对 Java 编译器来说完全合法，对直接 `new SourceService(...)` 的单元测试也完全合法；但对 Spring 来说，注入入口不再唯一。在没有可选无参构造器的情况下，Bean 实例化最终失败。

关键点是：报错中的“找不到无参构造器”是容器选择策略失败后的结果，不代表正确修法一定是补一个无参构造器。

## 为什么现有测试没有发现

### 1. 普通单元测试绕过了容器

测试如果这样创建对象：

```java
SourceService service = new SourceService(mockClient, "test-bucket");
```

它只验证业务方法和测试构造器可用，没有触发组件扫描、`@Value` 解析或 Spring 构造器选择。测试全部通过，与生产启动失败并不矛盾。

### 2. Mockito 注入不等于 Spring 注入

`@InjectMocks` 由 Mockito 根据自己的规则选择构造器。它能成功创建对象，不代表 Spring BeanFactory 会做出相同选择。依赖注入框架之间的构造器解析规则不能互相替代。

### 3. Maven 打包默认不启动完整应用

`mvn package` 会编译源码、运行配置好的测试并生成 JAR，但不会天然保证：

- 每个 `@Service` 都能被 Spring 实例化；
- 所有 `@Value` 占位符都能解析；
- 条件化 Bean 组合在真实 profile 下成立；
- 应用上下文可以刷新完成。

“JAR 打出来了”只证明构建流水线完成，不等于运行时对象图成立。

### 4. 功能关闭不代表 Bean 不会创建

这次功能在产品层已经默认关闭，但实现类仍是普通单例 `@Service`。Spring 启动时依然会创建它。

```text
功能开关关闭
≠ Controller 不展示入口
≠ Bean 不参与容器启动
```

只有显式使用 `@ConditionalOnProperty`、延迟加载或拆分配置，才可能让关闭状态不创建相关 Bean。是否这样设计要根据系统需求决定，不能靠“反正功能没开放”推断。

## 正确修复方式

### 方案一：明确标注生产注入构造器

当保留多个构造器确有价值时，直接标注容器应使用的构造器：

```java
@Autowired
SourceService(
        @Value("${app.endpoint:}") String endpoint,
        @Value("${app.bucket:}") String bucket
) {
    // ...
}
```

测试便利构造器可以保持包可见，避免成为对外 API。

这个修复的含义不是“到处补 `@Autowired`”，而是只在多构造器消除了自动推断前提时显式声明意图。

### 方案二：恢复单一构造器

更长期的做法通常是让组件只保留一个构造器，把配置收拢到 `@ConfigurationProperties`，外部客户端由独立 `@Bean` 提供：

```java
@Service
class SourceService {
    SourceService(Client client, SourceProperties properties) {
        // ...
    }
}
```

测试直接传 mock `Client` 和测试配置对象即可，不需要为测试额外设计一个构造器。对象创建规则也更接近生产。

### 不推荐：为了消除报错硬加无参构造器

无参构造器可能让 Spring 创建出一个依赖为空、配置未初始化的对象，把启动期的明确失败推迟为运行期空指针或错误请求。除非该类确实支持先构造再完整注入，否则这只是隐藏问题。

## 用最小容器测试锁住回归

修完后，至少增加一个会真实触发 Spring 构造器解析的测试：

```java
@Test
void springCanInstantiateTheService() {
    try (AnnotationConfigApplicationContext context =
                 new AnnotationConfigApplicationContext()) {
        TestPropertyValues.of(
                "app.endpoint=",
                "app.bucket="
        ).applyTo(context);
        context.register(SourceService.class);
        context.refresh();

        assertThat(context.getBean(SourceService.class)).isNotNull();
    }
}
```

这个测试比直接 `new` 多覆盖了：

- Spring 对多构造器的选择；
- `@Value` 占位符解析；
- Bean 生命周期的基本创建过程。

它又比完整 `@SpringBootTest` 更轻，不必连接真实数据库、认证服务和外部 API。若组件依赖自动配置，可以改用 `ApplicationContextRunner`；若风险来自跨模块对象图，则应补一个经过测试 profile 隔离的完整上下文启动测试。

## 发布门禁应该验证什么

这次故障说明构建门禁至少要分成三层：

| 层级 | 解决的问题 | 典型手段 |
| --- | --- | --- |
| 编译与单测 | 方法行为、边界和算法是否正确 | `mvn test`、前端测试 |
| 容器级测试 | Bean 是否可实例化、配置能否绑定 | 最小 Context、`ApplicationContextRunner` |
| 发布就绪 | 打包后的 JAR 能否在目标配置下启动 | systemd 启动、内部 readiness、自动回滚 |

第三层不能被前两层完全替代。反过来，自动回滚也不能替代测试：它保护了服务可用性，却仍然会消耗一次发布窗口。

就绪失败时，发布脚本应保留：

- 新版本目录和版本号；
- `systemctl status`；
- 最近的应用日志；
- 最后一次 HTTP 状态和响应体；
- 实际回滚到的旧版本路径。

这样可以迅速区分 Bean 实例化、配置缺失、数据库校验和健康检查契约错误。

## 可复用的检查清单

当 Spring 组件增加构造器时，逐项确认：

- 组件是否从单构造器变成了多构造器；
- Spring 应使用的构造器是否已经明确；
- 测试是否只在手工 `new` 或 Mockito 环境中创建对象；
- 功能关闭时相关 Bean 是否仍会创建；
- 是否至少有一个真实 ApplicationContext 实例化测试；
- 发布流程是否会在 readiness 失败后输出根异常并安全回滚。

最重要的经验是：**单元测试验证的是你调用到的代码路径，容器启动验证的是对象图。** 当一次修改改变了对象如何被创建，就必须有一项测试真正让容器创建它。
