# 08 - Spring / Spring Boot 3 / MyBatis 核心源码面试题

> 本章覆盖：Spring IoC 与 3 种 DI 方式、BeanFactory / FactoryBean / ApplicationContext、Bean 作用域与线程安全、容器 `refresh()` 流程与 Bean 完整生命周期、三级缓存解决循环依赖源码级走查与 AOP 关联、Spring AOP（JDK vs CGLIB、5.2+ 通知顺序、BeanPostProcessor 钩子）、`@Transactional` 隔离级别与 7 种传播机制、10 大事务失效场景与编程式 `TransactionTemplate`、Spring MVC 流程与 Filter/Interceptor/ControllerAdvice 执行顺序、Spring Boot 3 自动装配（`AutoConfiguration.imports`）、手写 Starter、`proxyBeanMethods` Full/Lite 模式、优雅停机与 Jakarta EE / AOT / GraalVM、MyBatis `#{}` vs `${}`、Mapper 动态代理、4 大核心对象拦截器原理与 Batch 批量优化。

---

## 目录

- [1. Spring IoC 核心与 Bean 生命周期（Q1 - Q10）](#1-spring-ioc-核心与-bean-生命周期q1---q10)
- [2. Spring AOP 与事务管理（Q11 - Q18）](#2-spring-aop-与事务管理q11---q18)
- [3. Spring MVC 核心流程与 Web 进阶（Q19 - Q23）](#3-spring-mvc-核心流程与-web-进阶q19---q23)
- [4. Spring Boot 3 核心原理与云原生演进（Q24 - Q32）](#4-spring-boot-3-核心原理与云原生演进q24---q32)
- [5. MyBatis 核心源码与性能调优（Q33 - Q40）](#5-mybatis-核心源码与性能调优q33---q40)

---

## 1. Spring IoC 核心与 Bean 生命周期（Q1 - Q10）

### Q1：什么是 IoC 和 DI？依赖注入有哪 3 种方式？为什么 Spring 官方强烈推荐构造器注入？

**直接结论**  
- **IoC（Inversion of Control，控制反转）**：将传统由业务代码主动 `new` 对象、组装依赖的**控制权反转交给 Spring 容器**统一管理，彻底解耦组件之间的强绑定。
- **DI（Dependency Injection，依赖注入）**：IoC 的核心实现手段，由容器在运行期动态将依赖对象注入目标 Bean 中，包含 **构造器注入、Setter 注入、字段（Field）注入** 3 种方式。**Spring 4.x+ 官方强烈推荐构造器注入**。

**深度原理对比：为什么构造器注入优于 `@Autowired` 字段注入？**
1. **保证不可变性（`final`）与防 NPE**：依赖可以用 `private final` 修饰，只要对象构造完成，所有必需依赖就必然已经就绪，彻底避免“半成品对象”在测试或运行时抛出 `NullPointerException`。
2. **容器无关，极易做纯单元测试**：字段注入强依赖反射或 Spring 容器；构造器注入在 JUnit 中只需直接 `new OrderService(mockRepo)`，无需启动 Spring 上下文，单测毫秒级跑完。
3. **强制暴露耦合过重与循环依赖**：如果构造器塞了七八个参数，代码异味立现（违反单一职责），提示你重构拆分；同时构造器循环依赖在启动期就会立即报错暴露（Spring Boot 2.6+/3.x 已默认禁止循环依赖）。
4. **字段注入（`@Autowired` 在属性上）的隐患**：IDEA 会直接标黄警告 *Field injection is not recommended*——容易隐藏循环依赖、无法注入 `final` 字段、破坏封装性。

- **面试官连环追问**：
  1. **如果一个类只有一个构造器，需要加 `@Autowired` 吗？**  
     Spring 4.3+ 起，**如果类只有一个构造器，可以省略 `@Autowired`**（配合 Lombok `@RequiredArgsConstructor` + `private final` 字段，既简洁又完美符合官方推荐！）。如果有多个构造器，必须用 `@Autowired` 显式指明容器走哪一个。

---

### Q2：`BeanFactory`、`FactoryBean` 和 `ApplicationContext` 有什么区别？

**直接结论**  
三者名字极像但职责完全不同：**`BeanFactory`** 是 IoC 底层顶层工厂接口；**`ApplicationContext`** 是面向开发者的增强型高级容器；**`FactoryBean`** 则是让开发者自定义复杂对象创建逻辑的**特殊工厂 Bean 接口**。

| 对比维度 | `BeanFactory` | `ApplicationContext` | `FactoryBean<T>` |
| :--- | :--- | :--- | :--- |
| **定位** | Spring 最顶层核心容器接口 | 继承自 `BeanFactory` 的企业级容器 | 一个受容器管理的**特殊 Bean 接口** |
| **加载时机** | **延迟加载**（调用 `getBean()` 时才实例化） | 启动 `refresh()` 时**预实例化所有非懒加载单例 Bean** | `getObject()` 按需返回真实目标对象 |
| **扩展能力** | 仅提供最基础的 DI 与 Bean 获取 | 集成国际化、事件发布（`ApplicationEventPublisher`）、AOP、资源加载、自动注册 `BeanPostProcessor` | 用于屏蔽复杂代理/连接对象构造细节（如 MyBatis `SqlSessionFactoryBean`、`MapperFactoryBean`） |

**手写 `FactoryBean` 核心细节**：
```java
@Component("myClient")
public class MyClientFactoryBean implements FactoryBean<MyHttpClient> {
    @Override
    public MyHttpClient getObject() {
        return new MyHttpClient.Builder().timeout(3000).build(); // 返回真正业务对象
    }
    @Override
    public Class<?> getObjectType() { return MyHttpClient.class; }
}
```
- **高频考点**：`context.getBean("myClient")` 拿到的是 **`getObject()` 返回的 `MyHttpClient` 实例**；如果要获取 `MyClientFactoryBean` 工厂自身，必须加前缀 **`&`**：`context.getBean("&myClient")`！

---

### Q3：Spring Bean 的 6 种作用域是什么？单例 Bean 线程安全吗？如何在单例中安全使用原型 Bean？

**直接结论**  
Spring 定义了 6 种作用域：`singleton`（默认单例，容器级唯一）、`prototype`（原型，每次 `getBean` 新建）、Web 环境专属的 `request`、`session`、`application`、`websocket`。

**深度解析**
1. **单例（Singleton）Bean 是线程安全的吗？**
   - **默认不保证线程安全！** Spring 只是把对象做成单例共享给多线程并发调用：
   - **无状态 Bean**（如 Controller、Service、DAO 只注入其他单例依赖，方法内只用局部变量）：**绝对线程安全**。
   - **有状态 Bean**（在类上定义了可变的成员变量 `private int count;` 或 `HashMap`）：**并发读写必现线程安全问题！**
   - **解法**：① 尽量写无状态 Bean，状态放到局部变量或数据库/Redis；② 必须有状态时，使用 `ThreadLocal`（如 Spring 事务与 `RequestContextHolder` 的做法）、`Atomic*` 原子类或把作用域改为 `prototype`/`request`。
2. **单例 Bean 中注入原型（Prototype）Bean 为什么会失效？怎么解决？**
   - **失效原因**：单例 Bean 在容器启动时只注入一次依赖，之后永远复用这同一个原型实例，失去了“每次调用都新建”的意义！
   - **3 种正确解法**：
     1. **`ObjectProvider<PrototypeBean>`（推荐，最优雅）**：每次业务方法内调用 `objectProvider.getObject()` 实时向容器索取新实例。
     2. **`@Lookup` 方法注入**：在单例类定义 `protected abstract PrototypeBean createBean();` 加上 `@Lookup`，Spring 底层用 CGLIB 动态重写该方法每次走 `getBean()`。
     3. **`@Scope(value = "prototype", proxyMode = ScopedProxyMode.TARGET_CLASS)`**：注入 CGLIB 动态代理壳，每次调用方法时代理内部重新创建真实目标对象。

---

### Q4：Spring 容器启动刷新的核心流程（`AbstractApplicationContext.refresh()`）有哪些关键步骤？

**直接结论**  
Spring 容器启动的核心骨架全浓缩在 `AbstractApplicationContext.refresh()` 的 **12 个核心模板方法**中（面试能清晰串出核心主线即过关）：

```text
1. prepareRefresh()                 -> 初始化环境变量、校验必填属性、准备早期事件监听器集合
2. obtainFreshBeanFactory()         -> 解析 XML/注解配置，将所有类扫描转化为 BeanDefinition 注册进 DefaultListableBeanFactory
3. prepareBeanFactory()             -> 配置类加载器、SpEL 解析器、注册忽略的 Aware 接口与内置环境 Bean
4. postProcessBeanFactory()         -> 留给子类（如 Web 容器）扩展 BeanFactory
5. invokeBeanFactoryPostProcessors()-> 【关键】执行 BeanDefinitionRegistryPostProcessor & BeanFactoryPostProcessor
                                       (Spring Boot 自动装配 @Configuration 解析、@ComponentScan 扫描、占位符替换都在这！)
6. registerBeanPostProcessors()     -> 【关键】实例化并按 PriorityOrdered/Ordered 顺序注册所有 BeanPostProcessor (仅注册，暂不拦截业务)
7. initMessageSource()              -> 初始化国际化资源
8. initApplicationEventMulticaster()-> 初始化事件广播器
9. onRefresh()                      -> 【关键】Spring Boot 在此启动内嵌 Tomcat / Jetty Web 容器！
10. registerListeners()             -> 注册监听器并派发早期积压事件
11. finishBeanFactoryInitialization()->【核心】实例化所有剩余非懒加载的单例 Bean (触发完整的 Bean 生命周期与 AOP 代理！)
12. finishRefresh()                 -> 发布 ContextRefreshedEvent 事件、启动 Lifecycle 生命周期组件
```

- **面试官连环追问**：
  1. **`BeanFactoryPostProcessor` 和 `BeanPostProcessor` 分别在第几步执行？**  
     前者在第 5 步执行，此时业务 Bean 还没实例化，操作的是 `BeanDefinition` 元数据；后者在第 6 步注册，并在第 11 步每个 Bean 实例化和初始化前后回调生效。

---

### Q5：详细描述 Spring Bean 的完整生命周期？（源码级必考）

**直接结论**  
Spring Bean 的生命周期本质上围绕 `AbstractAutowireCapableBeanFactory.doCreateBean()` 展开，严格分为 **实例化 → 属性填充 → 初始化 → 销毁** 四大阶段，中间穿插 `InstantiationAwareBeanPostProcessor`、`Aware` 接口和 `BeanPostProcessor` 扩展点。

```text
[阶段 1: 实例化前 & 实例化]
  ① InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation() (有机会直接返回代理短路后续流程)
  ② createBeanInstance(): 反射调用构造器创建原始对象 (此时属性全为 null)
  ③ MergedBeanDefinitionPostProcessor: 收集 @Autowired / @Value / @PostConstruct 元数据
  ④ 【三级缓存曝光】: 若允许循环依赖，调用 addSingletonFactory() 放入三级缓存 ObjectFactory

[阶段 2: 属性填充]
  ⑤ InstantiationAwareBeanPostProcessor.postProcessAfterInstantiation()
  ⑥ InstantiationAwareBeanPostProcessor.postProcessProperties() -> 处理 @Autowired / @Resource / @Value 注入
  ⑦ populateBean(): 完成全部属性赋值

[阶段 3: 初始化 (initializeBean)]
  ⑧ Aware 接口回调: BeanNameAware -> BeanClassLoaderAware -> BeanFactoryAware
  ⑨ BeanPostProcessor.postProcessBeforeInitialization()
       └── (内置 ApplicationContextAwareProcessor 回调 EnvironmentAware / ApplicationContextAware 等)
       └── (内置 InitDestroyAnnotationBeanPostProcessor 执行 @PostConstruct 方法！)
  ⑩ InitializingBean.afterPropertiesSet() -> 执行自定义 @Bean(initMethod = "...")
  ⑪ BeanPostProcessor.postProcessAfterInitialization()
       └── 【核心】AnnotationAwareAspectJAutoProxyCreator 在此生成 AOP 动态代理对象！

[阶段 4: 就绪与销毁]
  ⑫ 放入一级缓存 singletonObjects 单例池供业务使用
  ⑬ 容器关闭 (close): @PreDestroy -> DisposableBean.destroy() -> 自定义 destroy-method
```

- **面试官连环追问**：
  1. **`@PostConstruct`、`InitializingBean.afterPropertiesSet()`、`init-method` 三者执行顺序是什么？**  
     **`@PostConstruct`（在 `postProcessBeforeInitialization` 内执行） → `afterPropertiesSet()` → `init-method`**。记忆口诀：“注解最先，接口其次，XML/属性配置最后”。
  2. **AOP 代理一定是在第 11 步 `postProcessAfterInitialization` 生成的吗？**  
     正常情况下是的；但如果该 Bean 发生了循环依赖，它的 AOP 代理会提前在三级缓存 `ObjectFactory.getObject()` 中生成，且生成后会在第 11 步通过 `earlyProxyReferences` 集合检查跳过重复代理！

---

### Q6：Spring 是如何解决循环依赖的？三级缓存源码级走查？

**直接结论**  
Spring 通过 **`DefaultSingletonBeanRegistry` 中的三级缓存**，提前暴露“已实例化但尚未初始化”的半成品引用，从而解决了 **“单例（Singleton）+ 非构造器注入（Setter/字段注入）”** 的循环依赖。

**三级缓存源码结构**：
```java
// 一级缓存：存放完全初始化好、可直接使用的成熟单例 Bean（包括已生成的代理对象）
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

// 二级缓存：存放提前暴露出来的原始半成品 Bean 或提前生成的早期代理对象
private final Map<String, Object> earlySingletonObjects = new ConcurrentHashMap<>(16);

// 三级缓存：存放创建 Bean 早期引用的 ObjectFactory（Lambda 工厂，按需决定返回原始对象还是 AOP 代理）
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
```

**`A ↔ B` 循环依赖完整走查（假设 A 被 AOP 代理）**：
1. 容器调用 `getBean("a")`：依次查一级、二级、三级缓存均无，标记 A 正在创建中（`singletonsCurrentlyInCreation.add("a")`）。
2. A 完成实例化（原始对象 `aRaw`），**将 Lambda 表达式 `() -> getEarlyBeanReference("a", mbd, aRaw)` 存入三级缓存 `singletonFactories`**。
3. A 执行 `populateBean` 填充属性，发现需要 B → 触发 `getBean("b")`。
4. B 完成实例化并把自己也放入三级缓存，B 填充属性发现需要 A → 触发 `getBean("a")`！
5. **关键转折点**：B 查缓存找 A 时，一级没有，二级没有，在**三级缓存**找到了 A 的 `ObjectFactory`！于是调用 `getObject()`（即 `getEarlyBeanReference`），**提前创建了 A 的 AOP 代理对象 `aProxy`**，把 `aProxy` 存入**二级缓存 `earlySingletonObjects`** 并从三级缓存移除！
6. B 顺利拿到 `aProxy` 完成属性注入与初始化，放入**一级缓存**。
7. 控制权回到 A：A 拿到完整的 B，接着走完自己的初始化；到第 11 步 AOP 后置处理器时，发现 `earlyProxyReferences` 已有记录 A 提前代理过，直接跳过不再二次代理；最后 A 从**二级缓存**取出最终的 `aProxy` 放入**一级缓存**，并清空二级缓存。

---

### Q7：为什么循环依赖必须用三级缓存？二级缓存为什么在 AOP 下不行？Spring Boot 2.6+/3.x 为何默认禁止？

**直接结论**  
- **如果没有 AOP 代理，两级缓存确实就够了！**
- **引入三级缓存的核心原因，是为了在不破坏 Spring Bean 生命周期规范的前提下，优雅处理“带 AOP 代理的循环依赖”**。

**深度对比：为什么只有二级缓存会出问题？**
1. 假设不用三级缓存，只用二级缓存，你只有两种选择：
   - **选择 A（实例化完直接放原始对象进二级缓存）**：那么 B 注入的永远是 A 的**原始对象**；等 A 初始化完做 AOP 代理时，生成了一个新的 `aProxy`，最终容器中是 `aProxy` 而 B 持有的是 `aRaw`，**不仅 B 调用 A 时事务/切面全部失效，Spring 还会直接抛出 `BeanCurrentlyInCreationException`（引用不一致校验失败）**！
   - **选择 B（不管有没有循环依赖，所有 Bean 实例化完立刻在 `doCreateBean` 开头就先创建好 AOP 代理塞进二级缓存）**：这严重违反了 Spring 生命周期的设计原则——正常 AOP 代理本该在所有属性填充和初始化完成后的 `postProcessAfterInitialization` 中创建。
2. **三级缓存的精妙之处**：放一个 `ObjectFactory` 函数式钩子进去，**只有真正发生循环依赖、别人来找你要引用时，才被逼无奈提前执行 `getEarlyBeanReference` 生成代理**；如果根本没有循环依赖，这个函数永远不会被调用，完美恪守生命周期！

**为什么 Spring Boot 2.6+ / 3.x 默认禁止循环依赖？如何解决构造器/Prototype 循环依赖？**
- **默认禁止原因**：循环依赖本身是架构职责划分不清的坏味道，且在 `@Async`、某些复杂 AOP 代理场景下，如果代理后置处理器不走 `SmartInstantiationAwareBeanPostProcessor` 仍会报引用不一致错误（如 `@Async` 经典报错 `Bean with name 'a' has been injected into other beans in its raw version`）。Spring 官方干脆设 `spring.main.allow-circular-references=false` 倒逼解耦。
- **解法**：
  1. **架构重构**（提取公共第三层 Service C，或使用 Spring 事件解耦）。
  2. **使用 `@Lazy` 延迟注入**（连**构造器循环依赖**都能解决！因为注入的是一个延迟加载代理壳，真正调用方法时才去 `getBean`）。
  3. **使用 `ObjectProvider<B>`** 按需获取。
  4. **Prototype（多例）循环依赖**：因为多例 Bean 不缓存，每次都是 `new`，Spring 绝对无法解决（除非改单例或自己 `ObjectProvider` 手动管理）。

---

### Q8：`BeanPostProcessor` 与 `BeanFactoryPostProcessor` 的区别与典型源码应用？

**直接结论**  
两者都是 Spring 最重要的核心扩展接口（SPI），最大区别在于**干预的生命周期阶段与操作对象完全不同**：

| 对比项 | `BeanFactoryPostProcessor` (及其子接口 `BeanDefinitionRegistryPostProcessor`) | `BeanPostProcessor` |
| :--- | :--- | :--- |
| **执行时机** | 容器 `refresh()` 第 5 步，**所有普通 Bean 尚未实例化之前** | 容器 `refresh()` 第 11 步，**每个 Bean 实例化并初始化前后** |
| **操作对象** | **`BeanDefinition`（Bean 的设计图纸/元数据）**、动态注册新 `BeanDefinition` | **已经创建出来的 `Bean` 实例对象** |
| **经典源码实现** | `ConfigurationClassPostProcessor`（解析 `@Configuration`/`@ComponentScan`/`@Import`）、`PropertySourcesPlaceholderConfigurer`（解析 `${...}` 占位符） | `AutowiredAnnotationBeanPostProcessor`（注入 `@Autowired`）、`AnnotationAwareAspectJAutoProxyCreator`（生成 AOP 代理） |

- **实战避坑**：不要在 `BeanFactoryPostProcessor` 里调用 `beanFactory.getBean(...)`，否则会提前触发目标 Bean 实例化，导致该 Bean 错过后续第 6 步才注册的 `BeanPostProcessor`（例如它的 `@Autowired` 和 `@Transactional` 全部失效）！

---

### Q9：`@Autowired` 与 `@Resource` 的区别？多实现类注入冲突如何解决？

**直接结论**  
- **`@Autowired`** 是 Spring 原生注解（由 `AutowiredAnnotationBeanPostProcessor` 解析），默认**先按类型（`byType`）**匹配，找到多个候选再按**字段名/参数名（`byName`）**筛选；支持标注在构造器、方法、字段、参数上，支持 `required = false`。
- **`@Resource`** 是 JSR-250 标准注解（JDK/Jakarta 提供，Spring Boot 3 迁至 `jakarta.annotation.Resource`，由 `CommonAnnotationBeanPostProcessor` 解析），默认**先按名称（`byName`）**匹配（若未指定 `name` 属性则取字段名/setter 名），找不到再按类型（`byType`）兜底；**不支持标注在构造器上**。

**同一接口有多个实现类时的 4 种解法**：
1. **`@Primary`**：标记在某个默认主实现类上，优先注入它。
2. **`@Qualifier("beanName")`**：配合 `@Autowired` 精确指定目标 Bean 名称。
3. **直接按集合注入（策略模式神技）**：`@Autowired private Map<String, PayStrategy> strategyMap;` 或 `List<PayStrategy>`，Spring 会自动把所有实现类按 BeanName 为 Key 装配进 Map，彻底消灭 `if-else`！

---

### Q10：Spring 事件驱动模型原理是什么？`@TransactionalEventListener` 解决什么痛点？

**直接结论**  
Spring 事件模型基于**观察者模式**，由事件（`ApplicationEvent` 或普通 POJO）、发布者（`ApplicationEventPublisher`）和广播器（`ApplicationEventMulticaster`）构成。

**核心痛点与 `@TransactionalEventListener`**：
- **普通 `@EventListener` 的坑**：默认是**同步执行**且**在当前事务内执行**！如果下单事务里发布了“发短信/发 MQ 事件”，监听器刚把 MQ 发出去了，结果后面下单 SQL 抛异常事务回滚了——**用户收到短信/下游收到订单消息，但本地库里订单根本没生成！**
- **正解**：使用 **`@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)`**（默认相位就是 `AFTER_COMMIT`）：Spring 会将监听回调挂到 `TransactionSynchronizationManager` 中，**确保只有当前事务真正 `commit` 成功后，才触发执行监听器逻辑**（如发通知/投递非事务 MQ）。
- **进阶坑**：在 `AFTER_COMMIT` 阶段执行数据库写操作如果还加 `@Transactional`，注意必须用 **`Propagation.REQUIRES_NEW`**，因为原事务已经提交结束了，共用旧事务状态会导致新写操作不开启真实新事务而无法提交！

---

## 2. Spring AOP 与事务管理（Q11 - Q18）

### Q11：什么是 Spring AOP？JDK 动态代理与 CGLIB 动态代理的区别？Spring Boot 2.x/3.x 默认用哪个？

**直接结论**  
Spring AOP（面向切面编程）通过**运行时动态代理**将日志、权限、事务、限流等横切逻辑织入业务方法。底层主要采用 **JDK 动态代理** 和 **CGLIB 字节码增强**。

| 对比维度 | JDK 动态代理 | CGLIB 动态代理（Spring 内嵌 `spring-core` 的 ASM） |
| :--- | :--- | :--- |
| **实现原理** | 实现目标类的**接口**（`java.lang.reflect.Proxy` 生成 `$ProxyN` 代理类） | **继承目标类**生成子类（重写非 `final` 方法，FastClass 机制索引调用避免反射） |
| **前提限制** | 目标类**必须至少实现一个接口** | 目标类不能是 `final` 类，目标方法不能是 `final` 或 `private`/`static` |
| **Spring Boot 默认策略** | Spring Framework 默认：有接口用 JDK，无接口用 CGLIB | **Spring Boot 2.x / 3.x 默认强制 `proxy-target-class=true`，全部使用 CGLIB！** |

- **为什么 Spring Boot 2.x/3.x 默认改用 CGLIB？**  
  因为如果用 JDK 动态代理，当你的类实现了某个接口 `UserService`，一旦有人在注入时直接写了 `@Autowired UserServiceImpl`（注入实现类而非接口），启动就会因为 JDK 代理类与实现类无继承关系而直接报错 `BeanNotOfRequiredTypeException`！统一用 CGLIB 子类继承则完全兼容按接口或按实现类注入。
- **面试官连环追问**：
  1. **JDK 17+ 运行 CGLIB 会遇到模块化限制吗？**  
     Spring 6 / Spring Boot 3 使用自带重新打包的 `org.springframework.cglib`（基于最新 ASM 9.x），完全适配 JDK 17/21 模块化特性且不再依赖老版本无参构造要求（借助 Objenesis 直接绕过构造器分配对象）。

---

### Q12：Spring AOP 的核心术语与通知（Advice）执行顺序（Spring 5.2+ 变化）？

**直接结论**  
- **核心概念**：**Aspect（切面）** = **Pointcut（切点，匹配哪些方法）** + **Advice（通知，增强逻辑）**；**JoinPoint（连接点）** 是程序可被拦截的点（Spring AOP 只支持**方法级**连接点）；**Weaving（织入）** 是生成代理的过程。
- **Spring 5.2+ 通知执行顺序（重大面试细节）**：
  - **正常流程**：`@Around(前)` → `@Before` → **目标方法执行** → `@AfterReturning` → `@After` → `@Around(后)`
  - **异常流程**：`@Around(前)` → `@Before` → **目标方法抛异常** → `@AfterThrowing` → `@After`
  - **注意变化**：在 Spring 5.2 之前，是 `@After` 先于 `@AfterReturning` / `@AfterThrowing` 执行；**Spring 5.2.7+ 修正为类似 Java `try-catch-finally` 的直觉顺序**：`@AfterReturning`/`@AfterThrowing` 先执行，最后执行 `@After`（相当于 `finally`）！
- **多切面嵌套执行顺序**：使用 `@Order(数值)` 控制，**数值越小优先级越高（越在洋葱最外层）**：`Order(1) Before` → `Order(2) Before` → 目标方法 → `Order(2) After` → `Order(1) After`。

---

### Q13：Spring AOP 源码级实现原理：代理对象何时创建？拦截链如何执行？

**直接结论**  
Spring AOP 的总开关 `@EnableAspectJAutoProxy` 最终向容器注册了核心后置处理器 **`AnnotationAwareAspectJAutoProxyCreator`**（实现了 `SmartInstantiationAwareBeanPostProcessor`）。

**核心源码两阶段**：
1. **代理创建阶段（`wrapIfNecessary()`）**：
   - 正常在 Bean 初始化完成后的 `postProcessAfterInitialization()` 触发（若发生循环依赖则提前至 `getEarlyBeanReference()`）。
   - 调用 `findEligibleAdvisors()` 扫描容器中所有 `@Aspect` 切面并按 `@Order` 排序，匹配当前 Bean 是否有任意方法命中切点表达式（`Pointcut.matches`）。
   - 如果命中至少一个 `Advisor`，则交由 `ProxyFactory`（根据配置与接口情况选择 `JdkDynamicAopProxy` 或 `ObjenesisCglibAopProxy`）生成代理对象替换原 Bean 返回。
2. **方法拦截执行阶段（责任链模式 `ReflectiveMethodInvocation`）**：
   - 客户端调用代理方法时，进入 `DynamicAdvisedInterceptor.intercept()`（CGLIB）或 `JdkDynamicAopProxy.invoke()`（JDK）。
   - 将匹配该方法的所有 `Advisor` 统一适配成 `MethodInterceptor` 拦截器链。
   - 构建 `ReflectiveMethodInvocation` 对象并调用 **`proceed()`**：内部通过游标 `currentInterceptorIndex` 递归推进整条拦截器链，依次执行各通知的前置逻辑 → 目标方法反射调用 → 各通知的后置/异常逻辑。

---

### Q14：Spring `@Transactional` 声明式事务的底层实现原理是什么？

**直接结论**  
Spring 声明式事务本质上就是 **Spring AOP 的经典应用**：通过 `ProxyTransactionManagementConfiguration` 注册事务切面，核心拦截器 **`TransactionInterceptor`** 环绕目标方法，利用 **`PlatformTransactionManager`** 与基于 `ThreadLocal` 的 **`TransactionSynchronizationManager`** 协同管控数据库连接的获取、提交与回滚。

**源码核心步骤**：
1. **开启事务（`createTransactionIfNecessary`）**：
   - `TransactionInterceptor.invoke()` 调用 `DataSourceTransactionManager.doBegin()`。
   - 从 `DataSource` 拿一个物理 `Connection`，关闭自动提交（`con.setAutoCommit(false)`），设置隔离级别，并**将此 `ConnectionHolder` 绑定到当前线程的 `ThreadLocal`（`TransactionSynchronizationManager.resources`）中**！
   - 后续在同一个线程内执行的 MyBatis/JDBC SQL 都会从这个 `ThreadLocal` 取出**同一个数据库连接**，从而保证多条 SQL 处于同一个物理数据库事务中！
2. **提交与回滚**：
   - 目标方法正常返回 → 调用 `commitTransactionAfterReturning()` 触发 `con.commit()`。
   - 抛出异常 → 进入 `completeTransactionAfterThrowing()`，判断异常是否符合回滚规则（默认仅 `RuntimeException` 或 `Error`），符合则 `con.rollback()`，**注意：如果不符合回滚规则（如抛出 `IOException` 受检异常），依然会 `commit` 提交之前成功的 SQL！**
   - `finally` 阶段清理 `ThreadLocal` 绑定并归还连接给连接池，恢复 `autoCommit = true`。

---

### Q15：`@Transactional` 的隔离级别与 7 种事务传播行为详解？（高频必考）

**直接结论**  
- **隔离级别（`Isolation`）**：`DEFAULT`（使用数据库默认，MySQL InnoDB 为可重复读 RR，Oracle 为读已提交 RC）、`READ_UNCOMMITTED`、`READ_COMMITTED`、`REPEATABLE_READ`、`SERIALIZABLE`。
- **7 种传播行为（`Propagation`）**：决定“当一个事务方法 B 被另一个事务方法 A 调用时，事务应该如何传递与挂起”。

| 传播行为 | 说明（假设外层 A 调用内层 B） | 核心场景与关键坑点 |
| :--- | :--- | :--- |
| **`REQUIRED`（默认）** | A 有事务则**加入同一个事务**；A 无事务则自己新建 | 只要任意一处抛异常标记 `rollbackOnly`，整个事务全部一起回滚（哪怕 A `try-catch` 了 B 的异常，提交时照样报 **`UnexpectedRollbackException`**！） |
| **`REQUIRES_NEW`** | **挂起 A 的事务**，B 永远**开启一个全新独立事务** | **写操作日志 / 审计记录**：即使主业务 A 失败回滚，B 的日志依然独立提交不回滚！（注意：两连接同时占着，严防死锁与连接池耗尽） |
| **`NESTED`** | 若 A 有事务，则在当前物理事务内借助 **`JDBC Savepoint` 创建嵌套子事务** | B 失败只回滚到 Savepoint，**不影响 A 继续提交**（前提是 A 捕获了 B 的异常）；若 A 最终整体回滚，B 也会随 A 一起回滚！（`REQUIRES_NEW` 则不会随 A 回滚） |
| **`SUPPORTS`** | A 有事务就加入；A 没事务就非事务运行 | 纯查询方法 |
| **`NOT_SUPPORTED`** | 永远以非事务运行；若 A 有事务则**挂起 A** | 调外部耗时 HTTP/RPC 时避免占用 DB 事务连接 |
| **`MANDATORY`** | 必须在已有事务中运行；若 A 无事务直接**抛异常** | 强制要求调用方必须开启事务的内部组件 |
| **`NEVER`** | 必须在无事务中运行；若 A 有事务直接**抛异常** | 禁止被事务包裹的方法 |

- **面试官连环追问**：
  1. **为什么外层 A `try-catch` 住了内层 B（`REQUIRED`）抛出的异常，最后还是整体回滚并报 `UnexpectedRollbackException`？**  
     因为 A 和 B 共用同一个物理事务。B 抛异常走出拦截器时，Spring 已经在共享的 `TransactionStatus` 上打下了全局标记 `rollbackOnly = true`。虽然 A 吞掉了异常以为能继续提交，但最后 `commit()` 检查到 `rollbackOnly == true` 就会强制回滚并抛出此异常！**解法**：如果希望 B 失败不影响 A，B 应该改为 `REQUIRES_NEW` 或 `NESTED`！

---

### Q16：`@Transactional` 事务失效的 10 大经典场景及修复方案？（必考）

**直接结论**  
面试回答这类题，务必结构化归类为 **“代理拦截失效”、“异常处理与配置失效”、“底层环境与并发失效”** 三大类，张口就来 8~10 条：

1. **同类内部自调用（最高频）**：`methodA()` 直接调用 `this.methodB()`，`this` 是原始目标对象而非 Spring 代理对象，完全绕过了 `TransactionInterceptor`！  
   *修复*：拆出独立 Service；或注入自身代理 `self`；或 `@EnableAspectJAutoProxy(exposeProxy = true)` 后用 `((XxxService) AopContext.currentProxy()).methodB()`。
2. **方法访问权限不是 `public`**：`AbstractFallbackTransactionAttributeSource` 源码硬性检查 `Modifier.isPublic(method.getModifiers())`，`private`/`protected`/包级方法直接返回 `null` 忽略（注：`@Transactional` 加在接口上时用 CGLIB 也可能失效，建议一律加在实现类 `public` 方法上）。
3. **方法是 `final` 或 `static`**：CGLIB 通过生成子类重写方法实现增强，`final`/`static` 方法无法被子类重写，切面无法织入！
4. **异常被 `try-catch` 吞掉且未重新抛出**：事务切面抓不到异常，误以为执行成功直接提交。  
   *修复*：记录日志后继续 `throw`；或在 `catch` 块手动调用 `TransactionAspectSupport.currentTransactionStatus().setRollbackOnly()`。
5. **抛出受检异常（Checked Exception）或自定义非 RuntimeException**：Spring 默认只回滚 `RuntimeException` 和 `Error`！抛出 `IOException`、`SQLException`、`Exception` 均**不回滚**！  
   *修复*：养成团队铁律，一律显式写 **`@Transactional(rollbackFor = Exception.class)`**。
6. **类未被 Spring 管理**：忘了加 `@Service` / `@Component`。
7. **传播行为配置错误**：误设为 `Propagation.NOT_SUPPORTED` / `NEVER` / `SUPPORTS`（无外层事务时）。
8. **多线程异步调用**：事务数据库连接绑定在主线程 `ThreadLocal` 中，新开子线程（或 `@Async`）拿不到主线程连接，会独立提交或根本无事务，主线程回滚管不到子线程！
9. **底层存储引擎不支持事务**：如 MySQL 表建成了 `MyISAM`（必须用 `InnoDB`）。
10. **多数据源未指定对应的 `TransactionManager`**：切换了副库数据源，事务管理器却默认绑定主库，导致副库不回滚。

---

### Q17：编程式事务 `TransactionTemplate` 为什么在生产中往往优于 `@Transactional`？

**直接结论**  
`@Transactional` 粒度是**整个方法级别**。如果方法里夹杂着 Redis 操作、外部 HTTP 接口调用、文件上传等耗时操作，会导致**数据库连接被长期霸占不释放（大事务问题）**，极易拖垮连接池、引发死锁和主从延迟！

**生产级大事务优化实战（`TransactionTemplate`）**：
```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final TransactionTemplate transactionTemplate;

    public void submitOrder(OrderReq req) {
        // 1. 事务外执行：参数校验、查 Redis 缓存、调风控/优惠券 RPC 接口 (绝不占用 DB 连接！)
        RiskResult risk = riskRpcClient.check(req);
        if (!risk.isPassed()) throw new BizException("风控拦截");

        // 2. 精准包裹极小粒度的 DB 写事务，毫秒级提交释放连接！
        transactionTemplate.executeWithoutResult(status -> {
            orderMapper.insert(buildOrder(req));
            stockMapper.deduct(req.getSkuId(), req.getCount());
        });

        // 3. 事务提交后再发送 MQ 通知 / 删缓存
        mqProducer.sendOrderCreated(req.getOrderId());
    }
}
```
- **优势**：精确控制事务边界到几行纯 DB 代码；彻底杜绝自调用失效问题；无需担心受检异常漏配 `rollbackFor`（可在 `status.setRollbackOnly()` 精细掌控）。

---

### Q18：`@Async` 异步注解的常见坑点（事务、ThreadLocal 上下文、默认线程池 OOM）？

**直接结论**  
Spring `@Async` 也是基于 **AOP 动态代理**实现的，同样存在**同类自调用失效**问题；除此之外，生产上还有三大致命深坑：
1. **默认线程池坑（`SimpleAsyncTaskExecutor`）**：如果不自定义 `TaskExecutor` Bean，Spring 默认会用 `SimpleAsyncTaskExecutor`——它**根本不复用线程，每个请求直接 `new Thread()`**，高并发下瞬间创建上万线程导致 `OutOfMemoryError: unable to create new native thread`！**生产务必自定义有界 `ThreadPoolTaskExecutor` 并配置拒绝策略**。
2. **丢失请求上下文与 MDC 链路追踪**：切换线程后 `ThreadLocal`（如用户信息、TraceId）会丢失，必须给自定义线程池配置 **`TaskDecorator`** 在提交任务前拷贝父线程 `MDC` / 上下文，执行完 `finally` 清理。
3. **无返回值异步异常静默丢失**：返回 `void` 的 `@Async` 方法抛异常主线程抓不到，需实现 `AsyncUncaughtExceptionHandler` 统一告警兜底。

---

## 3. Spring MVC 核心流程与 Web 进阶（Q19 - Q23）

### Q19：Spring MVC 处理 HTTP 请求的完整核心流程（`DispatcherServlet` 源码走查）？

**直接结论**  
所有请求都统一流经前端控制器 **`DispatcherServlet.doDispatch()`**，由它调度 **`HandlerMapping`、`HandlerAdapter`、`HttpMessageConverter` / `ViewResolver`** 三大核心组件完成闭环：

```text
Client HTTP 请求
   │
   ▼
1. DispatcherServlet.doDispatch()
   ├──► 2. HandlerMapping (如 RequestMappingHandlerMapping): 根据 URL/Method 找到 HandlerExecutionChain
   │       (包含目标 Controller 方法封装 HandlerMethod + 匹配的 HandlerInterceptor 链)
   ├──► 3. HandlerAdapter (如 RequestMappingHandlerAdapter): 执行参数解析与返回值处理
   │       ├──► 4. 依次正序执行所有 Interceptor.preHandle() (若某一个返回 false 则直接中断并倒序触发 afterCompletion)
   │       ├──► 5. 参数解析 (HandlerMethodArgumentResolver: @RequestBody/@PathVariable/@RequestParam) + @Validated 校验
   │       ├──► 6. 反射调用 Controller 方法
   │       └──► 7. 返回值处理 (HandlerMethodReturnValueHandler):
   │               - 若有 @ResponseBody/@RestController -> HttpMessageConverter (Jackson) 直接序列化 JSON 写回 Response
   │               - 传统页面 -> 返回 ModelAndView，经 ViewResolver 解析模板渲染
   ├──► 8. 倒序执行 Interceptor.postHandle() (若前后端分离直接写完 Response 则对响应体修改意义有限)
   ├──► 9. 若中间抛异常 -> HandlerExceptionResolver (@RestControllerAdvice + @ExceptionHandler) 统一兜底转化响应
   └──► 10. 无论是否异常 -> 倒序执行 Interceptor.afterCompletion() 清理资源 (如 ThreadLocal)
```

---

### Q20：`Filter`（过滤器）、`HandlerInterceptor`（拦截器）和 AOP / `@ControllerAdvice` 的执行顺序与区别？

**直接结论**  
请求穿过容器的顺序遵循严格的**洋葱模型**：**Tomcat Filter → `DispatcherServlet` → `HandlerInterceptor` → Spring AOP / `@ControllerAdvice`**。

```text
Request ──► [Filter 前] ──► [DispatcherServlet] ──► [Interceptor.preHandle] ──► [AOP @Around/@Before] ──► Controller
                                                                                                             │
Response ◄── [Filter 后] ◄── [afterCompletion] ◄── [Interceptor.postHandle] ◄── [@ControllerAdvice 异常/AOP后] ◄──┘
```

| 对比维度 | `Filter` (Servlet 规范) | `HandlerInterceptor` (Spring MVC) | Spring AOP (`@Aspect`) |
| :--- | :--- | :--- | :--- |
| **所属层与依赖** | Servlet 容器（Tomcat），脱离 Spring 也能用 | Spring MVC 容器，**能直接 `@Autowired` Spring Bean** | Spring Core，作用于任意 Spring Bean 方法 |
| **拦截粒度** | 粗粒度 URL 匹配，拿不到具体执行的 Controller 方法签名 | 中粒度，能拿到 **`HandlerMethod`** 对象（类/方法/注解元数据） | **最细粒度**，精确到类/方法/入参/返回值/异常（不止 Web 层） |
| **典型场景** | 全局编码、CORS 跨域、XSS 过滤、请求体包装缓存 | 登录态 JWT 校验、接口权限、接口耗时监控 | Service 事务、分布式锁、操作日志、防重放 |

- **实战避坑**：Filter 中抛出的异常，**`@RestControllerAdvice` 抓不到**！因为 `@ControllerAdvice` 工作在 `DispatcherServlet` 内部的 `HandlerExceptionResolver` 层，Filter 在它外层。如果要让 Filter 异常也统一返回 JSON，必须在 Filter 内部捕获并借助 `HandlerExceptionResolver` 转发，或者直接用 Response 写出 JSON。

---

### Q21：`@RestControllerAdvice` + `@ExceptionHandler` 全局异常处理与 `@Validated` 参数校验机制？

**直接结论**  
Spring MVC 通过 **`ExceptionHandlerExceptionResolver`** 在 `doDispatch` 的 `catch` 阶段扫描带有 `@RestControllerAdvice` 注解的全局类，按异常继承链**精准匹配最具体（继承距离最近）的 `@ExceptionHandler`** 返回统一 JSON 结构（如 `Result.fail(code, msg)`）。

**`@Valid` vs `@Validated` 核心区别**：
- **`@Valid`**（Jakarta Bean Validation `jakarta.validation.Valid`）：支持**嵌套对象级联校验**（如 `OrderReq` 里的 `List<ItemReq>` 字段加 `@Valid` 才会递归校验子项），但**不支持分组校验**。
- **`@Validated`**（Spring 提供）：支持**分组校验**（如 `CreateGroup` vs `UpdateGroup` 校验 ID 是否必传），支持标注在类上触发 Spring AOP 方法级参数校验；**不能直接标在内部嵌套字段上做级联校验**（需在字段上用 `@Valid` 配合外层 `@Validated`）。
- **校验失败抛出的异常**：`@RequestBody` JSON 校验失败抛 `MethodArgumentNotValidException`；表单/Query 参数校验失败抛 `ConstraintViolationException` 或 `BindException`，均需在 `@ExceptionHandler` 中统一捕获。

---

### Q22：`HttpMessageConverter` 工作原理与 Jackson 常见生产坑？

**直接结论**  
`HttpMessageConverter` 负责在 HTTP 请求体字节流（`HttpInputMessage`）与 Java 对象之间进行双向转换（前后端分离默认使用 `MappingJackson2HttpMessageConverter`）。

**三大高频生产坑与解法**：
1. **雪花算法 19 位 `Long` ID 传给前端 JS 精度丢失**：JavaScript `Number` 最大安全整数仅 $2^{53} - 1$（16 位），19 位雪花 ID 后三位全变 `0`！  
   *解法*：全局配置 Jackson 把 `Long` / `long` 序列化为 `String`（或关键字段加 `@JsonSerialize(using = ToStringSerializer.class)`）。
2. **`LocalDateTime` 序列化为数组 `[2026, 11, 18, 17, 45]` 或带 `T` 格式**：  
   *解法*：关闭 `SerializationFeature.WRITE_DATES_AS_TIMESTAMPS`，并配置全局 `JavaTimeModule` 统一格式化为 `"yyyy-MM-dd HH:mm:ss"`。
3. **修改 `ObjectMapper` 配置不生效**：手动 `new ObjectMapper()` 覆盖或继承了 `WebMvcConfigurationSupport` 会导致 Spring Boot 自动装配的默认配置失效！应该通过 **`Jackson2ObjectMapperBuilderCustomizer`** Bean 或 `extendMessageConverters` 增量定制。

---

### Q23：Spring MVC 如何解决跨域（CORS）？Filter 与 Interceptor 配置跨域为什么会冲突？

**直接结论**  
浏览器同源策略下，非简单请求（如带 `Authorization` Header 或 `Content-Type: application/json`）会先发送 **`OPTIONS` 预检请求**。

**生产坑点与正解**：
- **为什么用 `WebMvcConfigurer.addCorsMappings()` 配好了跨域，加上登录拦截器（`HandlerInterceptor`）后还是报 CORS 错？**  
  因为预检请求 `OPTIONS` **不会携带 Token**！如果登录拦截器直接把 `OPTIONS` 拦截返回 401，请求根本走不到 Spring MVC 写 CORS 响应头就被干掉了！
- **终极正解**：使用 **`CorsFilter`**（优先级高于所有业务 Filter 和 Interceptor），或者在网关层（Spring Cloud Gateway / Nginx）统一处理跨域（注意**网关和应用层不要同时配重复的 `Access-Control-Allow-Origin`**，否则响应头出现两个值浏览器照样报错！）。同时记得当 `allowCredentials = true` 允许带 Cookie 时，Spring Boot 2.4+/3.x 禁止设置 `allowedOrigins("*")` 必须改用 **`allowedOriginPatterns("*")`**。

---

## 4. Spring Boot 3 核心原理与云原生演进（Q24 - Q32）

### Q24：`@SpringBootApplication` 注解拆解与 Spring Boot 3 自动装配完整源码流程？

**直接结论**  
Spring Boot 自动装配的本质是：**扫描所有 Jar 包约定的声明文件 → 过滤条件注解（`@Conditional*`）→ 将符合条件的配置类动态导入 IoC 容器**。

**深度源码拆解**：
1. **`@SpringBootApplication` 三大核心元注解**：
   - `@SpringBootConfiguration`：本质就是 `@Configuration`，标记主启动类为配置源。
   - `@ComponentScan`：扫描启动类所在包及其所有子包下的 `@Component`/`@Service`/`@Controller`（**注意：这也是为什么主启动类要放在最外层根包下**）。
   - **`@EnableAutoConfiguration`**（自动装配核心）：内部包含 `@AutoConfigurationPackage`（记录根包路径供 JPA/MyBatis 扫描）和 **`@Import(AutoConfigurationImportSelector.class)`**。
2. **Spring Boot 2.7 / 3.x 自动装配读取路径变迁（高频考点）**：
   - **Spring Boot 2.7 之前**：`SpringFactoriesLoader` 读取所有 Jar 包下的 `META-INF/spring.factories`。
   - **Spring Boot 2.7 过渡 & Spring Boot 3.x 彻底移除旧方式**：改用 `ImportCandidates.load(AutoConfiguration.class, ...)` 读取 **`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`**（每行一个 `@AutoConfiguration` 类全限定名，更简洁且利于 AOT 静态分析）。
3. **条件过滤按需生效**：
   - 拿到一百多个自动配置类后，通过 **`@ConditionalOnClass`**（类路径是否有 Redis/DataSource）、**`@ConditionalOnMissingBean`**（允许用户自定义 Bean 覆盖默认配置）、**`@ConditionalOnProperty`** 等精确裁剪，借助 `spring-autoconfigure-metadata.properties` 提前快速跳过不匹配类，加速启动。

---

### Q25：手把手编写一个自定义 Spring Boot 3 Starter（完整核心步骤）？

**直接结论**  
自定义 Starter 命名规范：官方为 `spring-boot-starter-*`，第三方/公司内部为 **`*-spring-boot-starter`**（如 `sms-spring-boot-starter`），核心只需 4 步：

1. **定义配置属性类（`@ConfigurationProperties`）**：
   ```java
   @Data
   @ConfigurationProperties(prefix = "custom.sms")
   public class SmsProperties {
       private boolean enabled = true;
       private String accessKey;
   }
   ```
2. **编写核心业务服务类 `SmsClient`**（普通 POJO，不加 `@Component`）。
3. **编写自动配置类（Spring Boot 3 推荐使用 `@AutoConfiguration`）**：
   ```java
   @AutoConfiguration
   @EnableConfigurationProperties(SmsProperties.class)
   @ConditionalOnClass(SmsClient.class)
   @ConditionalOnProperty(prefix = "custom.sms", name = "enabled", havingValue = "true", matchIfMissing = true)
   public class SmsAutoConfiguration {
       @Bean
       @ConditionalOnMissingBean // 关键：允许业务方自己定义 Bean 覆盖默认实现
       public SmsClient smsClient(SmsProperties props) {
           return new SmsClient(props.getAccessKey());
       }
   }
   ```
4. **注册装配清单**：在 `src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件中写入一行全限定类名 `com.example.sms.SmsAutoConfiguration`（千万别忘了，Spring Boot 3 不再自动读 `spring.factories` 里的 `EnableAutoConfiguration`！）。

---

### Q26：`@Configuration(proxyBeanMethods = true/false)` Full 模式与 Lite 模式的区别？

**直接结论**  
这是 Spring 5.2 / Spring Boot 2.2+ 引入的重要性能优化参数，控制配置类是否生成 **CGLIB 代理子类**来保障方法间相互调用的单例语义。

| 对比项 | `proxyBeanMethods = true` (**Full 模式，默认**) | `proxyBeanMethods = false` (**Lite 模式**) |
| :--- | :--- | :--- |
| **底层代理** | 为配置类生成 CGLIB 代理（启动慢一点，内存多一点） | **不生成代理**，直接以普通配置类处理 |
| **方法互调行为** | 配置类内 `@Bean` 方法 A 直接调用另一个 `@Bean` 方法 `b()` 时，**会拦截返回容器中的单例 Bean**（不重复 `new`） | 方法 A 直接调用方法 `b()` 相当于普通 Java 方法调用，**每次都会 `new` 出新实例**！ |
| **适用场景** | 配置类内部 `@Bean` 之间存在**直接的方法互调依赖** | `@Bean` 之间通过**方法参数注入**（推荐写法）而无互调，**Spring Boot 3 所有内置 `@AutoConfiguration` 默认全都是 `false`！** |

---

### Q27：Spring Boot 配置文件加载顺序与外部化配置优先级是怎样的？

**直接结论**  
Spring Boot 外部化配置遵循 **“高优先级覆盖低优先级”**，且越贴近运行部署环境（命令行、OS 环境变量）优先级越高，方便容器化 K8s 注入参数。

**核心优先级排序（从高到低精简记法）**：
1. **命令行参数**：`java -jar app.jar --server.port=9090`（最高，线上临时救急必用）。
2. **Java 系统属性**：`-Dserver.port=9090`（`System.getProperties()`）。
3. **操作系统环境变量**：`SERVER_PORT=9090`（Docker/K8s `env` 注入）。
4. **外部/包外配置文件**（jar 同级目录 `config/application-{profile}.yml` > 同级 `application-{profile}.yml` > 包外 `application.yml`）。
5. **包内（classpath）配置文件**：`application-{profile}.yml` > `application.yml`。
6. **`@PropertySource`** 注解导入的配置 > `SpringApplication.setDefaultProperties`。

- **Spring Boot 2.4+ / 3.x 新特性**：支持在 YAML 中使用 **`spring.config.import: optional:nacos:app.yaml`** 直接无缝拉取 Nacos/Config Server 配置，不再强依赖臃肿的 `spring-cloud-starter-bootstrap`！

---

### Q28：Spring Boot 内嵌 Tomcat 是如何启动的？为什么不需要打成 WAR 包？

**直接结论**  
传统做法是把代码打成 WAR 丢进独立 Tomcat 的 `webapps` 目录；Spring Boot 则把 Tomcat（`tomcat-embed-core`）当成普通 Jar 依赖引入，在 **`refresh()` 第 9 步 `onRefresh()`** 中以代码方式实例化并启动内嵌 Tomcat。

**启动原理与调优要点**：
1. 引入 `spring-boot-starter-web` 后，主类默认创建的上下文是 **`AnnotationConfigServletWebServerApplicationContext`**。
2. 启动至 `onRefresh()` 时调用 `createWebServer()`，通过 `ServletWebServerFactory`（默认 `TomcatServletWebServerFactory`）创建 `Tomcat` 实例、绑定端口、将 Spring MVC 的 `DispatcherServlet` 注册为 Servlet 并启动 Connector 监听端口。
3. **Tomcat 生产核心调优参数**：
   - `server.tomcat.threads.max`（默认 **200**）：最大工作线程数。IO 密集型可适当调至 400~800，CPU 密集型不要太大。
   - `server.tomcat.max-connections`（默认 **8192**）：最大同时保持 TCP 连接数。
   - `server.tomcat.accept-count`（默认 **100**）：连接满后操作系统全连接队列等待长度（超出直接拒接）。

---

### Q29：Spring Boot Actuator 生产监控与 K8s Graceful Shutdown（优雅停机）如何落地？

**直接结论**  
微服务发版滚动重启时，如果直接 `kill -9`，正在处理一半的扣款/下单请求会瞬间中断报 502 错误、甚至导致数据不一致！生产必须落地 **Actuator 探针 + K8s `preStop` + Spring Boot 优雅停机**。

**生产标准配置与全链路时序**：
```yaml
server:
  shutdown: graceful # 开启优雅停机 (默认 immediate 直接中断!)
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s # 最多等待存量在途请求处理 30 秒，超时强制退出
management:
  endpoint:
    health:
      probes:
        enabled: true # 开启 K8s 专属 /actuator/health/liveness & readiness 探针
```
- **K8s 滚动发版防断流时序**：
  1. K8s 触发 Pod 销毁，同时做两件事：① 把 Pod 从 Service Endpoints 摘除；② 执行 `preStop: sleep 10s`（**关键！**因为 K8s 摘除 Endpoint 同步到所有节点 `kube-proxy`/Gateway 注册中心需要几秒时间差，`sleep 10s` 确保新流量彻底不再打过来！）。
  2. 随后向 JVM 发送 **`SIGTERM`（`kill -15`，严禁 `kill -9`）**。
  3. Spring Boot 收到 `SIGTERM`，内嵌 Tomcat 立刻拒绝接收新请求，同时放行**已进入的所有在途请求继续执行完（最多等 30s）**，最后关闭数据库连接池、MQ 消费者并优雅退出。

---

### Q30：Spring Boot 3 相比 2.x 有哪些重大升级？（2025-2026 高频新特性）

**直接结论**  
Spring Boot 3.0（基于 Spring Framework 6）是一次跨时代的基线大升级，面试官主要考察你是否跟进现代 Java 技术栈：

1. **JDK 基线跃迁至 Java 17+**（完美支持 **Java 21 虚拟线程 Virtual Threads**：只需一行配置 `spring.threads.virtual.enabled=true`，Tomcat 和 `@Async` 即可切换为百万级轻量虚拟线程！）。
2. **全面迁移至 Jakarta EE 9/10**：所有 `javax.servlet.*`、`javax.persistence.*`、`javax.validation.*`、`javax.annotation.*` 全部更名为 **`jakarta.*`**（老第三方库未适配升级是迁移最大阻碍）。
3. **支持 AOT 编译与 GraalVM Native Image**（毫秒级冷启动，内存降 5~10 倍）。
4. **声明式 HTTP 客户端 `@HttpExchange`**：Spring 6 原生内置类似 OpenFeign 的接口式 HTTP 客户端（`HttpServiceProxyFactory`），配合 `RestClient`（同步流式替代老旧 `RestTemplate`）。
5. **统一可观测性（Micrometer Observation）**：将指标（Metrics）与链路追踪（Tracing，替代 Spring Cloud Sleuth）统一为 `Observation` API，原生对接 OpenTelemetry 与 Prometheus。
6. **标准化错误响应 RFC 7807 `ProblemDetail`**：开箱即用的标准 JSON 错误格式。

---

### Q31：什么是 Spring Boot 3 AOT 编译与 GraalVM Native Image？有哪些限制？

**直接结论**  
传统 Spring 应用靠 JVM **运行时大量反射、CGLIB 动态代理、Classpath 扫描**实现灵活装配，代价是**启动慢（几秒~几十秒）、内存高（几百 MB）**，在 Serverless / K8s 弹性扩容场景很吃亏。

**原理与限制**：
- **AOT（Ahead-Of-Time，提前编译）**：在 Maven/Gradle 构建期就预先运行 Spring 应用分析所有 `@Configuration`、条件注解、Bean 依赖图，**提前生成确定的 Java 源码注册代码**（替代运行期慢吞吞的反射扫描）。
- **GraalVM Native Image**：将 AOT 生成的字节码 + 裁剪后的极简运行库直接静态编译为**原生机器码可执行文件（不依赖 JVM）**，实现 **50ms 级冷启动、内存占用降至 30~50MB**！
- **限制（为什么普通业务还不敢全面用）**：
  1. **动态性受限**：运行时动态反射、JDK 动态代理、JNI、资源文件读取都必须提供静态 Hint 配置（`RuntimeHintsRegistrar`），很多老旧第三方 Jar 包不兼容。
  2. **构建极慢且吃内存**：编译一次 Native Image 动辄耗时几分钟、吃掉 8~16GB 构建内存。
  3. **峰值吞吐略逊于 JIT**：没有 JVM JIT 运行期热点优化（Profile-Guided Optimization），长期跑满高负载的传统大服务仍推荐跑标准 JVM。

---

### Q32：Spring 中用到了哪些经典设计模式？（精准对应到源码类名）

**直接结论**  
回答设计模式切忌只背名字，一定要**把模式名字精准对应到 Spring 源码真实类名**：

| 设计模式 | 在 Spring 源码中的真实体现 |
| :--- | :--- |
| **工厂模式** | `BeanFactory`、`ApplicationContext`、`FactoryBean`（创建复杂对象） |
| **单例模式** | `DefaultSingletonBeanRegistry`（利用一级缓存 `ConcurrentHashMap` 注册表实现容器级单例） |
| **代理模式** | Spring AOP 底层 `JdkDynamicAopProxy`、`CglibAopProxy` |
| **模板方法模式** | `JdbcTemplate`、`RedisTemplate`、`TransactionTemplate`、`AbstractApplicationContext.refresh()`（定义骨架，`onRefresh()` 留给子类重写） |
| **观察者模式** | `ApplicationEvent`、`ApplicationListener`、`SimpleApplicationEventMulticaster` |
| **适配器模式** | `HandlerAdapter`（将不同形态的 Controller 统一适配给 `DispatcherServlet` 调用）、`AdvisorAdapter`（把各种通知统一适配成 `MethodInterceptor`） |
| **责任链模式** | Spring AOP 拦截器链 `ReflectiveMethodInvocation.proceed()`、Spring MVC `HandlerExecutionChain` 多拦截器执行 |
| **装饰器模式** | `BeanWrapper`、`HttpServletRequestWrapper`（如缓存可重复读请求体 `ContentCachingRequestWrapper`） |

---

## 5. MyBatis 核心源码与性能调优（Q33 - Q40）

### Q33：MyBatis 中 `#{}` 和 `${}` 的底层区别是什么？如何安全实现动态表名或 `IN` 查询？

**直接结论**  
- **`#{}`（预编译占位符，99% 场景必须用它）**：MyBatis 解析为 **`?`** 占位符，底层通过 JDBC `PreparedStatement.setXxx()` 设值并自动进行类型转义加引号，**100% 彻底杜绝 SQL 注入**，还能复用数据库预编译执行计划缓存！
- **`${}`（直接字符串替换拼接）**：由 `TextSqlNode` 原样拼接进 SQL 字符串中，**存在致命 SQL 注入漏洞**！

**何时不得不用 `${}` 及安全规范**：
- 表名、列名、`ORDER BY ${sortColumn} ${sortDir}` 等**SQL 语法结构部分**（JDBC `?` 占位符只能替代字面量值，不能替代标识符！）。
- **安全铁律**：凡是使用 `${}` 的地方，**绝对禁止直接接收前端用户输入**，必须在 Java 代码层通过 **枚举/白名单集合校验**（如 `Set.of("create_time", "price").contains(col)`），否则必被安全扫描打回！
- **多值 `IN` 查询**：必须用 **`<foreach collection="ids" item="id" open="(" separator="," close=")">#{id}</foreach>`**（千万别图省事用 `IN (${idsStr})`！）。

---

### Q34：MyBatis Mapper 接口没有实现类，为什么能被 `@Autowired` 注入并执行 SQL？

**直接结论**  
因为 MyBatis 借助 **JDK 动态代理 + Spring `FactoryBean` + `BeanDefinitionRegistryPostProcessor`**，在容器启动时悄无声息地把每个 Mapper 接口替换成了一个代理工厂 Bean！

**源码完整链路**：
1. **`@MapperScan` 扫描注册**：通过 `@Import(MapperScannerRegistrar.class)` 注册 `MapperScannerConfigurer`（实现了 `BeanDefinitionRegistryPostProcessor`）。
2. **偷梁换柱改图纸**：扫描指定包下的所有 Mapper 接口，把它们原本无法实例化的接口 `BeanDefinition` 的 `beanClass` 全部修改为 **`MapperFactoryBean.class`**，并把接口类型作为构造参数传入。
3. **`FactoryBean` 生成代理**：当业务 Service `@Autowired UserMapper` 时，Spring 调用 `MapperFactoryBean.getObject()` → 内部调用 `SqlSession.getMapper(UserMapper.class)` → **`MapperProxyFactory.newInstance()`** 使用 **`Proxy.newProxyInstance`** 创建实现 `UserMapper` 接口的代理对象，InvocationHandler 就是 **`MapperProxy`**！
4. **方法拦截执行 SQL**：调用 `userMapper.selectById(1L)` 时，直接进入 `MapperProxy.invoke()`（跳过 `Object` 的 `toString`/`hashCode` 与接口 `default` 方法），将全限定类名 + 方法名（`com.xx.UserMapper.selectById`）作为 `StatementId`，从全局 `Configuration.mappedStatements` 找到对应 SQL 执行！

---

### Q35：MyBatis 的核心组件与一条 SQL 的完整执行流程是怎样的？

**直接结论**  
MyBatis 执行一条 SQL 围绕四大核心执行组件协同流转：**`Executor`（调度器） → `StatementHandler`（JDBC 声明处理器） → `ParameterHandler`（入参赋值） → `ResultSetHandler`（结果集映射封装）**。

```text
1. MapperProxy 拦截接口方法调用 -> 委托给 DefaultSqlSession
2. SqlSession 委托给 Executor (先查二级缓存 CachingExecutor -> 再查 BaseExecutor 一级缓存)
3. 缓存未命中 -> Executor.doQuery/doUpdate() 创建 RoutingStatementHandler (默认 PreparedStatementHandler)
4. StatementHandler 调用 ParameterHandler.setParameters() -> 通过 TypeHandler 把 Java 入参绑定到 '?' 占位符
5. StatementHandler 调用 JDBC PreparedStatement.execute() 向数据库发 SQL
6. ResultSetHandler 处理 JDBC 返回的 ResultSet -> 通过 TypeHandler 和反射/构造器映射成 Java Bean / List 返回
```

---

### Q36：MyBatis 的三种 `Executor` 有什么区别？千万级数据批量插入如何做最优调优？

**直接结论**  
MyBatis 内置三种执行器（可在 `openSession(ExecutorType)` 或全局配置指定）：
1. **`SIMPLE`（默认）**：每次执行更新或查询都创建一个新的 JDBC `Statement` 对象，用完立即关闭。
2. **`REUSE`**：在当前 `SqlSession` 内**复用相同 SQL 的 `PreparedStatement` 对象**（缓存进 Map），减少反复预编译开销。
3. **`BATCH`**：专用于批量写操作（`INSERT`/`UPDATE`/`DELETE`），将所有 SQL 攒入 JDBC 批处理缓冲区，直到 `flushStatements()` 或 `commit()` 才一次性发给数据库执行。

**生产千万级批量插入最佳实践（比 `<foreach>` 拼接快 10 倍以上）**：
- **为什么不推荐用超大 `<foreach>` 拼一条包含几万条记录的巨型 `INSERT`？**  
  ① 拼接出的 SQL 动辄几十 MB，容易直接触发 MySQL **`max_allowed_packet`** 报错断开；② MyBatis OGNL 动态解析上万个节点极度消耗 CPU 甚至引发 OOM！
- **最优解法**：**`ExecutorType.BATCH` 批处理 + MySQL 连接串开启 `rewriteBatchedStatements=true`**！
  ```java
  // 1. JDBC URL 必须加上: ?rewriteBatchedStatements=true (让 MySQL 驱动在底层把批量指令自动重写为 multi-values 极速插入!)
  // 2. 分批次每 1000 条刷一次批处理，防止单次事务过大撑爆 undo log 与内存
  try (SqlSession batchSession = sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {
      OrderMapper mapper = batchSession.getMapper(OrderMapper.class);
      for (int i = 0; i < list.size(); i++) {
          mapper.insert(list.get(i));
          if ((i + 1) % 1000 == 0 || i == list.size() - 1) {
              batchSession.flushStatements();
              batchSession.clearCache(); // 清理一级缓存防 OOM！
          }
      }
      batchSession.commit();
  }
  ```

---

### Q37：MyBatis 一级缓存与二级缓存的底层实现？为什么生产严禁开启二级缓存？

**直接结论**  
- **一级缓存（`SqlSession` 级别，默认强制开启）**：底层是 `BaseExecutor` 内部持有的一个 `PerpetualCache`（简单 `HashMap`）。只在同一个 `SqlSession` 内生效；一旦执行 `insert/update/delete`、`commit`、`rollback` 都会立刻清空整个一级缓存。
- **二级缓存（`Mapper / Namespace` 级别，默认关闭）**：由装饰器 **`CachingExecutor`** 包装 `BaseExecutor`，多个 `SqlSession` 共享同一份缓存（需实体类实现 `Serializable`，且 `SqlSession` 提交或关闭后才会刷入二级缓存）。

**两个高频灵魂拷问**：
1. **为什么 MyBatis 整合 Spring 后，如果 Service 方法没加 `@Transactional`，一级缓存就“失效”了？**  
   因为 `mybatis-spring` 包装了 `SqlSessionTemplate`。没有 Spring 事务时，**每一次调用 Mapper 方法都会新打开一个 `SqlSession` 并在方法结束立即关闭**，导致连续两次相同查询根本不在同一个 `SqlSession` 里，一级缓存形同虚设！只有加上 `@Transactional` 开启事务，Spring 才会把同一个 `SqlSession` 绑定到 `ThreadLocal` 中复用。
2. **为什么生产中严禁开启 MyBatis 二级缓存？**  
   - **多表联查脏数据（致命）**：二级缓存按 `Namespace` 隔离。比如 `OrderMapper` 里做了一个关联 `User` 表的 Join 查询并缓存了结果；此时 `UserMapper` 更新了用户名，它只会清空 `UserMapper` 的缓存，**根本不会清空 `OrderMapper` 的缓存，导致读到永久脏数据**！
   - **分布式多实例不一致**：它是单机 JVM 本地内存，微服务多节点部署时节点 A 改库不会通知节点 B 失效。分布式缓存应统一交给 **Redis**。

---

### Q38：MyBatis 插件（Interceptor / PageHelper）底层原理是什么？如何手写一个拦截器？

**直接结论**  
MyBatis 插件机制是基于 **责任链模式 + JDK 动态代理**，专门拦截四大核心对象（**`Executor`、`StatementHandler`、`ParameterHandler`、`ResultSetHandler`**）的方法。

**源码原理（层层动态代理包装）**：
- 在创建上述四大对象时（如 `Configuration.newExecutor()`），都会调用 **`interceptorChain.pluginAll(target)`**。
- 遍历所有注册的 `Interceptor`，依次执行 `Plugin.wrap(target, this)`。如果目标对象是该拦截器 `@Intercepts` 签名指定拦截的类，就为其生成一层 JDK 动态代理包裹起来！**配置多个插件时，会形成多层套娃动态代理链**。
- **`PageHelper` 物理分页原理**：在 `Executor.query()`（或 `StatementHandler.prepare()`）前拦截执行，从 `ThreadLocal` 取出 `PageHelper.startPage(1, 10)` 设置的分页参数，自动改写原始 SQL 先查一次 `SELECT COUNT(0)` 总数，再拼接 `LIMIT 0, 10` 改写原 SQL 执行并封装返回 `PageInfo`（**注意**：`startPage` 后紧跟的**第一条**查询才有效，且若中途抛异常未消费记得 `PageHelper.clearPage()` 防线程池复用导致后续查询误分页！）。

**手写 SQL 耗时监控拦截器核心骨架**：
```java
@Intercepts({
    @Signature(type = StatementHandler.class, method = "query", args = {Statement.class, ResultHandler.class}),
    @Signature(type = StatementHandler.class, method = "update", args = {Statement.class})
})
public class SqlCostInterceptor implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            return invocation.proceed(); // 放行调用链
        } finally {
            long cost = System.currentTimeMillis() - start;
            if (cost > 500) {
                StatementHandler handler = (StatementHandler) invocation.getTarget();
                log.warn("Slow SQL [{}ms]: {}", cost, handler.getBoundSql().getSql());
            }
        }
    }
    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this); // 按 @Signature 决定是否生成代理
    }
}
```

---

### Q39：MyBatis 动态 SQL（`<if>` / `<where>` / `<foreach>`）底层是如何解析的？

**直接结论**  
MyBatis 在解析 XML 或 `@SelectProvider` 时，会将所有动态标签解析成一棵由 **`SqlNode`** 组成的语法树（如 `IfSqlNode`、`WhereSqlNode`、`ForEachSqlNode`、`MixedSqlNode`）。

**源码两种 `SqlSource` 对比**：
- **无动态标签且无 `${}` 的纯静态 SQL**：解析阶段直接构建 **`RawSqlSource`**，在应用启动时就一次性把 `#{}` 替换成 `?` 生成最终静态 SQL，运行时**零解析开销**。
- **包含 `<if>`/`<where>`/`<foreach>` 或 `${}` 的动态 SQL**：构建 **`DynamicSqlSource`**，每次执行 SQL 时根据入参通过 **OGNL 表达式引擎** 动态求值各 `SqlNode` 拼接出最终带 `?` 的 SQL 文本，再生成 `BoundSql` 交给 `StatementHandler`。
- **细节坑**：`<if test="status != null and status != ''">` 中，如果 `status` 是整数 **`0`**，在 OGNL 里 **`0 != ''` 会被判定为 `false`（因为 `0` 和空串都转成数字 0 相等）**！导致传入状态 `0` 时 `<if>` 条件直接丢失！**铁律：非字符串数字字段千万不要写 `!= ''`！**

---

### Q40：MyBatis-Plus 常见生产面试陷阱有哪些？与原生 MyBatis 如何配合？

**直接结论**  
MyBatis-Plus（MP）通过启动时反射解析实体类自动注入单表 CRUD `MappedStatement` 极大提升了开发效率，但在高并发生产环境下有 **5 大经典深坑**：

1. **逻辑删除（`@TableLogic`）与唯一索引冲突**：  
   如果表对 `username` 建了 `UNIQUE` 索引，逻辑删除把 `deleted` 从 `0` 改为 `1` 后，用户再重新注册同名 `username` 会直接触发数据库 `DuplicateKeyException`！  
   *解法*：删除时把 `deleted` 字段设为**主键 ID / 删时时间戳**（`0` 表示未删除，`id` 表示已删除），建联合唯一索引 `UNIQUE KEY(username, deleted)`！
2. **乐观锁插件（`@Version`）更新必须先查后改且只对内置方法有效**：  
   如果忘记把当前查出来的旧 `version` 塞进更新实体里，或自己手写了 XML SQL，乐观锁自动 `SET version = version + 1 WHERE version = old` 会直接失效；且并发失败时 `updateById` 只返回 `false`（影响行数为 0），**不会自动抛异常**，业务需自己判断返回值重试或报错。
3. **MP `ServiceImpl.saveBatch()` 默认不是真 Batch 还是有坑？**：  
   MP 底层其实已用了 `ExecutorType.BATCH`，但**如果 MySQL JDBC 连接串忘了配 `rewriteBatchedStatements=true`**，依然是一条条网络往返执行，插入速度极慢！
4. **多租户插件（`TenantLineInnerInterceptor`）误杀系统表/关联表**：  
   开启自动追加 `AND tenant_id = ?` 时，若某个全局字典表或子查询/Join 漏配 `ignoreTable`，会直接导致查询报错或查不到数据。
5. **复杂多表联查 & 深度调优**：MP 只适合单表与简单 Lambda 条件构造（防止写死字段名硬编码）；**凡涉及 3 表以上 Join、复杂聚合统计、极限性能 Hint SQL，必须老老实实回到 XML 手写原生 MyBatis SQL**！
