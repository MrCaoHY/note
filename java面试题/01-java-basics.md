# 01 - Java 核心基础面试题（深度精讲版）

> **本章涵盖**：面向对象与 JVM 分派机制、基本类型与精度陷阱、String 家族与常量池演进、关键字与内存语义、拷贝/序列化/SPI/枚举、反射/动态代理/泛型/注解、异常体系与 IO/零拷贝、JDK 8 ~ JDK 21 核心新特性（共 40 道高频深度面试题）。
>
> **阅读建议**：每道题均按「**直接结论 -> 底层原理 / 源码 / JVM 机制 -> 代码实战 -> 面试官连环追问 -> 实战避坑**」结构编排，建议结合手写代码与字节码思考。

---

## 目录

- [1. 面向对象与核心语义（Q1 ~ Q7）](#1-面向对象与核心语义q1--q7)
- [2. 基本类型、包装类与数值精度（Q8 ~ Q12）](#2-基本类型包装类与数值精度q8--q12)
- [3. String 家族与关键字语义（Q13 ~ Q19）](#3-string-家族与关键字语义q13--q19)
- [4. 拷贝、序列化、SPI 与枚举（Q20 ~ Q23）](#4-拷贝序列化spi-与枚举q20--q23)
- [5. 反射、动态代理、泛型与注解（Q24 ~ Q30）](#5-反射动态代理泛型与注解q24--q30)
- [6. 异常体系与 IO / 零拷贝（Q31 ~ Q35）](#6-异常体系与-io--零拷贝q31--q35)
- [7. JDK 8 ~ JDK 21 核心新特性（Q36 ~ Q40）](#7-jdk-8--jdk-21-核心新特性q36--q40)

---

## 1. 面向对象与核心语义（Q1 ~ Q7）

### Q1：Java 面向对象的三大特性是什么？多态的底层实现原理是什么？

- **直接结论**：
  - **封装**：隐藏对象内部数据与实现细节，仅对外暴露受控的方法入口（`private` + `getter`/`setter`/领域方法），提升安全性与可维护性。
  - **继承**：子类复用父类的非私有属性和行为并扩展新能力（Java 类单继承，接口多继承多实现），是实现多态的前提。
  - **多态**：同一操作作用于不同实际类型的对象时产生不同的执行结果，分为**编译时多态（静态绑定/重载）**和**运行时多态（动态绑定/重写）**。
- **底层原理（JVM 虚方法表 vtable 与调用指令）**：
  - Java 字节码提供 5 种方法调用指令：
    1. `invokestatic`：调用静态方法（编译期唯一确定，非虚方法）。
    2. `invokespecial`：调用构造器 `<init>`、`private` 方法和 `super.xxx()` 方法（编译期唯一确定，非虚方法）。
    3. `invokevirtual`：调用普通实例方法（**动态绑定**，多态核心；注意 `final` 方法也用 `invokevirtual` 但不可重写）。
    4. `invokeinterface`：调用接口方法（**动态绑定**，运行时通过接口方法表 `itable` 定位）。
    5. `invokedynamic`：运行时动态解析调用点（用于 Lambda 表达式、JDK 9+ 字符串拼接等）。
  - **虚方法表（vtable）查找机制**：
    ```text
    Parent 引用 p ──▶ 堆中 Child 实例
                          ├── 对象头 (Mark Word + Klass Pointer ──▶ Child.class 元数据)
                          └── 实例数据                                │
                                                                     ▼
                                                           Child 的 vtable (虚方法表)
                                                           [0] Object.toString() -> Object 实现
                                                           [1] Parent.eat()      -> Child.eat() (已重写覆盖)
                                                           [2] Child.run()       -> Child.run()
    ```
  - 类加载的**解析阶段**，JVM 会为每个类在元空间建立 `vtable`。如果子类重写了父类的方法，子类 `vtable` 中相同偏移量槽位的指针就会被替换为子类方法的入口地址。执行 `invokevirtual` 时，JVM 通过对象头的类元数据指针找到实际类型的 `vtable`，按固定偏移量直接跳转执行。
- **面试官连环追问**：
  - **追问 1**：构造方法、静态方法、`final` 方法、`private` 方法能被重写吗？能体现多态吗？
    - **回答**：都不能被重写，不体现运行时多态。`private` 对子类不可见；`static` 属于类而非对象，子类定义同名静态方法属于**方法隐藏（Hide）**；`final` 显式禁止重写；构造方法随类绑定，不参与继承。
  - **追问 2**：成员变量有多态性吗？
    - **回答**：**没有**。属性在编译期直接按**声明类型（静态类型）**静态绑定。例如 `Parent p = new Child();`，`p.name` 读取的是 `Parent` 中定义的字段，而 `p.getName()` 调用的是 `Child` 重写后的方法。
- **实战避坑**：
  - **严禁在父类构造方法中调用可被子类重写的方法！** 执行父类构造器时，子类的成员变量尚未初始化（仍为 `0`/`false`/`null`），此时发生动态绑定调用子类方法，极易触发 `NullPointerException` 或读取到未初始化状态。

---

### Q2：方法重载（Overload）与方法重写（Override）的区别？什么是静态分派与动态分派？

- **核心对比**：

| 对比维度 | 方法重载（Overload） | 方法重写（Override） |
| :--- | :--- | :--- |
| **发生范围** | 同一个类或父子类之间 | 父子类 / 接口实现类之间 |
| **方法签名** | 方法名相同，**参数列表（类型、个数、顺序）必须不同** | 方法名和参数列表必须**完全相同** |
| **返回/异常/权限** | 与返回值类型、修饰符、异常列表无关 | 返回值 $\le$ 父类（协变）；异常 $\le$ 父类；访问权限 $\ge$ 父类 |
| **绑定阶段** | **编译时多态（静态分派）** | **运行时多态（动态分派）** |

- **底层原理（静态分派 vs 动态分派）**：
  - **静态分派**：编译期 `javac` 只根据参数的**静态类型（外观类型）**决定选用哪个重载方法版本。
  - **动态分派**：运行期 JVM 根据方法接收者（`this`）的**实际类型**通过 `vtable` 决定执行哪个重写版本。
  - **结论**：Java 是一门**静态多分派、动态单分派**的语言（动态分派不关心传入参数的实际类型，只关心接收者实际类型）。
- **代码示例**：

```java
public class DispatchExample {
    static class Father {
        public void choose(QQ arg) { System.out.println("Father choose QQ"); }
        public void choose(WeChat arg) { System.out.println("Father choose WeChat"); }
    }
    static class Son extends Father {
        @Override
        public void choose(QQ arg) { System.out.println("Son choose QQ"); }
        @Override
        public void choose(WeChat arg) { System.out.println("Son choose WeChat"); }
    }
    static class QQ {}
    static class WeChat {}

    public static void main(String[] args) {
        Father father = new Father();
        Father son = new Son(); // 静态类型 Father，实际类型 Son
        father.choose(new QQ());    // Father choose QQ
        son.choose(new WeChat());   // Son choose WeChat（动态单分派：运行期确定接收者为 Son）
    }
}
```

- **面试官连环追问**：
  - **追问**：如果重载方法没有完全精确匹配的参数类型，编译器按什么优先级进行匹配？
    - **回答**：`精确匹配 -> 基本类型自动放宽（char->int->long->float->double） -> 自动装箱（int->Integer） -> 父类/接口（Integer->Number->Object） -> 可变参数（int...）`。注意：`char` 会先匹配 `int`，而不是先装箱成 `Character`。
- **实战避坑**：
  - 避免同时重载 `remove(int index)` 与 `remove(Object o)` 这种基本类型与包装类冲突的 API。典型陷阱：`List<Integer> list = new ArrayList<>(List.of(10, 20)); list.remove(1);` 删除的是下标 `1`（即 `20`），而非元素 `1`！要删除元素 `1` 必须传 `Integer.valueOf(1)`。

---

### Q3：接口（Interface）与抽象类（Abstract Class）的区别？JDK 8/9 接口新特性冲突怎么解？

- **核心对比**：

| 对比维度 | 抽象类 (`abstract class`) | 接口 (`interface`) |
| :--- | :--- | :--- |
| **设计语义** | **is-a**（是什么），代码复用与模板方法骨架 | **can-do / has-a**（具备何种能力），行为契约与解耦 |
| **继承/实现** | 单继承 | 类可多实现；接口可多继承多个父接口 |
| **成员变量** | 可定义普通变量、`static`、各种访问权限字段 | 默认且只能是 **`public static final`** 常量 |
| **构造方法** | **有**（供子类 `super()` 初始化状态） | **没有** |
| **方法类型** | 抽象方法、普通方法、静态方法 | 抽象方法、JDK 8 `default`/`static`、JDK 9 `private` |

- **面试官连环追问**：
  - **追问 1**：JDK 8 为什么要在接口中引入 `default` 默认方法？JDK 9 为什么又引入 `private` 方法？
    - **回答**：JDK 8 引入 `default` 是为了**二进制向后兼容**。例如给 `Collection` 接口增加 `stream()` 方法时，如果全是抽象方法，全世界所有实现类都将编译失败；用 `default` 提供默认实现即可无缝升级。JDK 9 引入 `private` 方法则是为了抽离多个 `default` 或 `static` 方法之间的公共辅助代码，防止逻辑重复且不对外暴露。
  - **追问 2**：如果一个类实现的两个接口有同名的 `default` 方法（菱形冲突）怎么办？如果父类和接口有同名方法呢？
    - **回答**：
      1. **两接口冲突**：编译报错，实现类**必须显式重写**该方法，内部可通过 `InterfaceA.super.foo()` 指定复用哪一方逻辑。
      2. **父类与接口冲突**：**类优先原则（Class Wins）**——父类的具体实现（哪怕继承自更上层父类）优先级高于接口的 `default` 方法。
- **实战避坑**：
  - 对外暴露领域服务/RPC 接口时一律使用 `interface`；只有当多个紧密相关的子类需要共享**可变成员变量状态**或**模板方法执行骨架**（如 `AbstractQueuedSynchronizer`、`HttpServlet`）时才使用抽象类。

---

### Q4：`==`、`equals()` 与 `hashCode()` 的区别与契约？为什么重写 `equals()` 必须重写 `hashCode()`？

- **直接结论**：
  - **`==`**：比较基本数据类型时比较**数值**；比较引用类型时比较**内存地址**（是否指向堆中同一对象实例）。
  - **`equals()`**：`Object` 默认实现就是 `return (this == obj);`；`String`、`Integer` 等类重写后用于比较**对象内部属性内容**是否逻辑相等。
  - **`hashCode()`**：返回对象的 `int` 散列码，`Object.hashCode()` 是本地方法（HotSpot 默认根据线程局部随机数状态等算法生成并缓存在对象头 Mark Word 中，并非直接返回物理内存地址）。
- **核心契约（`Object` 规范与散列集合前提）**：
  1. 若 `a.equals(b) == true`，则 `a.hashCode()` **必须**等于 `b.hashCode()`。
  2. 若 `a.hashCode() == b.hashCode()`，`a.equals(b)` **不一定**为 `true`（存在哈希碰撞）。
  3. **不重写 `hashCode()` 的致命后果**：假设 `User` 只重写了 `equals()` 比较 `id`。此时 `u1 = new User(1L)` 与 `u2 = new User(1L)` 逻辑相等，但调用默认 `hashCode()` 算出来的散列值不同，放入 `HashMap` / `HashSet` 时会被路由到**两个不同的哈希桶**中，导致 `set.add(u1); set.add(u2);` 去重失败（size=2），且 `map.get(new User(1L))` 返回 `null`！
- **标准规范模板**：

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    User user = (User) o;
    return Objects.equals(id, user.id) && Objects.equals(name, user.name);
}

@Override
public int hashCode() {
    return Objects.hash(id, name); // 底层调用 Arrays.hashCode，使用乘数 31 累乘
}
```

- **面试官连环追问**：
  - **追问 1**：为什么 `Objects.hash` / `String.hashCode` 习惯用 `31` 作为乘数？
    - **回答**：① `31` 是奇素数，能有效减少哈希分布冲突；② 可被 JVM 优化为位运算 `31 * i == (i << 5) - i` 极快；③ 大小适中，不易过早溢出导致高位丢失。
  - **追问 2**：重写 `equals` 时用 `getClass() != o.getClass()` 还是 `o instanceof User`？
    - **回答**：通常推荐 `getClass()` 保证严格的对称性（防止子类扩展新字段后破坏 `parent.equals(child)` 与 `child.equals(parent)` 的对称性）；只有当业务要求父子类之间允许跨类判等时（如代理对象、集合视图 `AbstractList`），才使用 `instanceof` 并将 `equals` 设为 `final`。
- **实战避坑**：
  - **严禁在有双向关联（如 `Order <-> OrderItem`）的实体上直接使用 Lombok `@Data`！** `@Data` 自动生成的 `equals`/`hashCode` 会递归包含所有字段，一旦发生循环引用会直接抛出 **`StackOverflowError`**！应使用 `@EqualsAndHashCode(of = "id")` 或手写核心唯一键。

---

### Q5：Java 是值传递还是引用传递？传递对象与不可变对象时有何表现？

- **直接结论**：**Java 永远只有值传递（Pass by Value），没有引用传递（Pass by Reference）**。
- **底层内存图解**：

```text
调用前：main 栈帧               堆内存
        userRef (0x100) ─────▶ User{ name = "Alice" }

传参时：拷贝一份地址值 0x100 给方法形参 u
        test(User u) 栈帧
        u (0x100) ───────────▶ 同一个 User{ name = "Alice" }
        ├─ 情况 A: u.setName("Bob")       ──▶ 堆中对象变成 "Bob"，外部 userRef 可见！
        └─ 情况 B: u = new User("Cindy")  ──▶ 仅把形参 u 改成 0x200，外部 userRef 仍是 0x100！
```

- **代码示例**：

```java
public class PassByValueTest {
    static void change(int num, String str, StringBuilder sb1, StringBuilder sb2) {
        num = 99;                      // 基本类型副本修改，外部无效
        str = "world";                 // String 不可变，相当于指向新常量池对象，外部无效
        sb1.append("_new");            // 通过地址副本修改堆内原对象属性，外部可见
        sb2 = new StringBuilder("X");  // 修改形参自身指向，外部无效
    }

    public static void main(String[] args) {
        int a = 1;
        String s = "hello";
        StringBuilder sb1 = new StringBuilder("A");
        StringBuilder sb2 = new StringBuilder("B");
        change(a, s, sb1, sb2);
        System.out.println(a + " | " + s + " | " + sb1 + " | " + sb2);
        // 输出：1 | hello | A_new | B
    }
}
```

- **面试官连环追问**：
  - **追问**：为什么说即使形参能修改外部对象的属性，也不能叫“引用传递”？
    - **回答**：真正的“引用传递”（如 C++ 的 `int& x`）传递的是**变量本身的别名/栈内存指针**，在方法内直接执行 `x = newObj` 可以让外部实参变量也指向新对象。而 Java 做不到这一点，无法写一个方法实现 `swap(User a, User b)` 交换外部两个引用的指向。
- **实战避坑**：公共服务或工具方法接收外部 `List` / `Map` 参数时，如果不希望调用方的数据被意外污染，要么在文档明确说明，要么做防御性拷贝 `new ArrayList<>(input)`。

---

### Q6：`Object` 类有哪些常用方法？`finalize()` 为什么被废弃？

- **直接结论**：`Object` 是所有类的顶层根类，共有 11 个方法（9 种核心功能）：
  1. `getClass()`：`final native`，返回运行时实际类的 `Class<?>`。
  2. `hashCode()` / `equals(Object)`：散列码与内容判等。
  3. `toString()`：默认返回 `类名@无符号16进制hashCode`。
  4. `clone()`：`protected native`，浅拷贝，子类必须实现 `Cloneable` 标记接口否则抛 `CloneNotSupportedException`。
  5. `wait()` / `wait(long)` / `wait(long, int)` / `notify()` / `notifyAll()`：`final native`，线程间等待/通知协作（必须在 `synchronized` 同步锁内调用，`wait` 会释放锁）。
  6. `finalize()`：GC 回收前回调（**JDK 9 起已标记 `@Deprecated(forRemoval = true)`**）。
- **为什么 `finalize()` 被彻底废弃？**
  1. **执行时间与是否执行完全不确定**：由低优先级的 `Finalizer` 守护线程异步串行执行，程序退出时可能根本来不及跑。
  2. **严重拖累 GC 性能并诱发 OOM**：重写了 `finalize()` 的对象在分配和回收时都会进入 `F-Queue` 队列多经历一次标记周期；若 `finalize()` 执行耗时过长，会导致整个 `F-Queue` 堆积，引发内存泄漏或 OOM。
  3. **对象复活风险**：在 `finalize()` 中重新将 `this` 赋值给静态变量会让对象死而复生（但一个对象的 `finalize()` 至多只会被 JVM 调用一次）。
- **现代替代方案**：
  - 常规 IO/锁/连接资源：使用 **`try-with-resources`（`AutoCloseable`）**。
  - 底层堆外内存等无感知清理：JDK 9+ 使用 **`java.lang.ref.Cleaner`**（基于虚引用 `PhantomReference`，无法访问到目标对象本身，彻底杜绝复活问题，如 `DirectByteBuffer`）。

---

### Q7：Java 类的初始化与对象创建执行顺序是怎样的？

- **直接结论**：口诀——**“先静态、后实例；先父类、后子类；先字段/代码块、后构造方法”**。
- **完整执行链路（首次创建 `new Child()` 且父子类均未加载）**：
  1. `Parent` 类的静态变量赋值 & 静态代码块 `static {}`（按源码书写先后顺序执行，`<clinit>` 阶段全局仅执行一次）。
  2. `Child` 类的静态变量赋值 & 静态代码块 `static {}`（按源码先后顺序执行，仅一次）。
  3. `Parent` 实例变量赋值 & 实例代码块 `{}`（按源码先后顺序执行）。
  4. `Parent` 构造方法。
  5. `Child` 实例变量赋值 & 实例代码块 `{}`（按源码先后顺序执行）。
  6. `Child` 构造方法。
- **面试官连环追问**：
  - **追问 1**：下面这段经典面试题输出什么？
    ```java
    class Demo {
        static Demo instance = new Demo(); // 1
        static int a = 10;                 // 2
        int b;
        Demo() { b = a; }                  // 3
        public static void main(String[] args) {
            System.out.println(instance.b);
        }
    }
    ```
    - **回答**：输出 **`0`**！因为类加载准备阶段 `a` 先被赋零值 `0`，初始化阶段按从上到下顺序先执行 `instance = new Demo()`，此时执行构造器 `b = a` 读到的是零值 `0`，之后才执行 `a = 10`！
  - **追问 2**：多线程同时首次访问一个未初始化的类，`<clinit>()` 会执行多次吗？
    - **回答**：不会。JVM 会对类的 `<clinit>()` 方法自动加内部初始化锁，保证多线程下有且仅有一个线程执行类初始化，其余线程阻塞等待。
- **实战避坑**：不要在静态变量初始化或 `static {}` 块中开新线程并 `join()` 等待它操作当前类——因为主线程拿着当前类的初始化锁，子线程访问当前类时会等待初始化锁，直接导致**类加载死锁**！

---

## 2. 基本类型、包装类与数值精度（Q8 ~ Q12）

### Q8：Java 有哪些基本数据类型？分别占多少字节？与包装类的核心区别？

- **8 种基本数据类型一览表**：

| 类型 | 包装类 | 字节数 | 默认值 | 取值范围 / 说明 |
| :--- | :--- | :--- | :--- | :--- |
| `byte` | `Byte` | 1 | `0` | $[-128, 127]$（$-2^7 \sim 2^7-1$） |
| `short` | `Short` | 2 | `0` | $[-32768, 32767]$ |
| `int` | `Integer` | 4 | `0` | $[-2^{31}, 2^{31}-1]$（约 $\pm 21.47$ 亿） |
| `long` | `Long` | 8 | `0L` | $[-2^{63}, 2^{63}-1]$ |
| `float` | `Float` | 4 | `0.0f` | 单精度浮点（IEEE 754，有效数字约 6~7 位） |
| `double` | `Double` | 8 | `0.0d` | 双精度浮点（IEEE 754，有效数字约 15~16 位） |
| `char` | `Character` | 2 | `'\u0000'` | $[0, 65535]$（无符号 UTF-16 码元） |
| `boolean` | `Boolean` | **规范未定** | `false` | JVM 规范规定单个 `boolean` 编译期用 `int`（4B）存储，`boolean[]` 用 `byte[]`（1B/元素） |

- **基本类型 vs 包装类**：
  1. **空值语义**：基本类型不能为 `null`，有默认值（如 `0`）；包装类默认为 `null`。
  2. **泛型支持**：集合与泛型只支持包装类（如 `List<Integer>`，不能 `List<int>`）。
  3. **内存开销**：开启指针压缩的 64 位 JVM 中，一个 `Integer` 对象占 **16 字节**（12B 对象头 + 4B `value`），是 `int`（4B）的 4 倍。
- **实战避坑**：《阿里 Java 开发手册》强制规定：**所有 POJO 类（DTO/DO/VO）属性和 RPC 方法返回值、参数必须使用包装类型**（防止数据库或远程调用返回 `null` 时被基本类型零值 `0` 静默掩盖引发资金/状态事故）；局部高频计算变量优先用基本类型避免无谓装箱 GC。

---

### Q9：自动装箱与拆箱原理是什么？包装类缓存池有哪些经典陷阱？

- **直接结论**：自动装箱与拆箱完全是**编译期语法糖**：
  - 装箱：`Integer x = 10;` 编译为 `Integer x = Integer.valueOf(10);`。
  - 拆箱：`int y = x;` 编译为 `int y = x.intValue();`。
- **6 大包装类缓存池（享元模式）**：
  - `Byte`、`Short`、`Integer`、`Long`：默认缓存 **$[-128, 127]$**。其中仅 **`Integer`** 支持通过 JVM 启动参数 **`-XX:AutoBoxCacheMax=<size>`** 调大缓存上限（底层在 `IntegerCache` 静态代码块中读取 `java.lang.Integer.IntegerCache.high`）。
  - `Character`：缓存 **$[0, 127]$**。
  - `Boolean`：缓存 `Boolean.TRUE` 和 `Boolean.FALSE`。
  - `Float` 和 `Double`：**没有缓存池**。
- **经典陷阱代码演示**：

```java
Integer a = 127, b = 127;
System.out.println(a == b);          // true（走 IntegerCache 缓存同一对象）

Integer c = 128, d = 128;
System.out.println(c == d);          // false（超出 127，各自 new Integer(128)）

Integer e = new Integer(127);        // 显式 new（JDK 9 已废弃该构造器）不走缓存池
System.out.println(a == e);          // false
System.out.println(c == 128);        // true（包装类与基本类型 == 会触发自动拆箱，变成数值比较）

// 致命 NPE 陷阱：三目运算符类型对齐拆箱
Boolean flag = true;
Integer i = null;
Integer res = flag ? i : 0;          // 抛出 NullPointerException！因为右侧是 int 0，编译器会把 i 拆箱为 int
```

- **实战避坑**：① 包装类判等永远用 **`Objects.equals(a, b)`**，绝不用 `==`；② 三目运算符两侧类型保持一致（如写成 `flag ? i : Integer.valueOf(0)`）；③ 循环累加变量如 `Long sum = 0L; for (...) sum += i;` 每次循环都会拆箱再装箱创建新 `Long` 对象，务必改为基本类型 `long sum = 0L;`。

---

### Q10：为什么浮点数会丢失精度？`BigDecimal` 底层原理与实战避坑？

- **浮点数为什么会丢失精度？**
  - 计算机使用二进制表示小数，遵循 **IEEE 754** 标准（符号位 + 指数位 + 尾数位）。许多十进制有限小数（如 `0.1`）转换成二进制是无限循环小数（`0.0001100110011...`），有限尾数位截断后就会产生微小误差（例如 `0.1 + 0.2 = 0.30000000000000004`）。
- **`BigDecimal` 底层实现原理**：
  - 核心由 **`BigInteger intVal`（无标度大整数）** 与 **`int scale`（小数点右移位数，即标度）** 组成（部分较小值还会直接缓存为 `long intCompact` 避免 `BigInteger` 开销）。
  - 原理：把小数放大 $10^{\text{scale}}$ 倍变成**精确的十进制整数**再做加减乘除，彻底规避二进制浮点表示误差。
- **五大实战避坑（金融/电商业务必考）**：
  1. **禁止使用 `new BigDecimal(double)`**：
     - `new BigDecimal(0.1)` 会把 `double` 本身不精确的二进制值带进去（实际值为 `0.100000000000000005551115...`）。
     - **正确写法**：使用 **`new BigDecimal("0.1")`** 或 **`BigDecimal.valueOf(0.1)`**（`valueOf` 内部先调用 `Double.toString(0.1)` 转为字符串）。
  2. **比较大小禁止使用 `equals()`**：
     - `BigDecimal.equals()` **既比较数值又比较 `scale`**！`new BigDecimal("1.0").equals(new BigDecimal("1.00"))` 返回 **`false`**！
     - **正确写法**：使用 **`a.compareTo(b) == 0`**（仅比较数值大小）。
  3. **除法必须指定精度和舍入模式**：
     - 直接调用 `a.divide(b)` 遇上 `1 / 3` 这种无限循环小数会直接抛出 **`ArithmeticException: Non-terminating decimal expansion`**！
     - **正确写法**：`a.divide(b, 2, RoundingMode.HALF_UP)`（银行家舍入法可用 `RoundingMode.HALF_EVEN`）。
  4. **防科学计数法输出**：
     - `new BigDecimal("0.0000001").toString()` 会输出 `"1E-7"`，前端展示或拼接字符串需用 **`toPlainString()`**。
  5. **不可变性忘接收返回值**：`a.add(b)` 返回的是新对象，原 `a` 不变，必须 `a = a.add(b);`。

---

### Q11：`short s1 = 1; s1 = s1 + 1;` 和 `s1 += 1;` 的区别？

- **直接结论**：`s1 = s1 + 1;` **编译报错**；`s1 += 1;` **编译通过并正常运行**。
- **底层原理解析**：
  - 在 Java 算术运算中，小于 `int` 的类型（`byte`、`short`、`char`）都会自动进行**二元数值提升（Binary Numeric Promotion）**转为 `int` 再计算。因此 `s1 + 1` 的结果类型是 `int`，直接赋值给 `short` 属于窄化转换，编译器拒绝隐式丢失精度，必须写 `s1 = (short) (s1 + 1)`。
  - 而复合赋值运算符 `+=`（包括 `-=`、`*=`、`/=`）在 Java 语言规范（JLS §15.26.2）中明确规定：**内置了隐式强转语义**，`E1 += E2` 等价于 `E1 = (TypeOfE1) ((E1) + (E2))`。
- **实战避坑**：虽然 `+=` 能过编译，但如果超过类型上限（如 `short s = 32767; s += 1;`），隐式强转会静默截断高位变成 `-32768`，不会抛任何异常！大数累加需防溢出时应使用 `Math.addExact()`。

---

### Q12：Java 中 `char` 占几个字节？能否存下一个汉字或 Emoji 表情？

- **直接结论**：Java 中 `char` 固定占 **2 个字节**（16 bit），内部采用 **UTF-16** 编码表示一个码元（Code Unit）。
- **能否存储汉字或 Emoji？**
  - **常用汉字（基本多文种平面 BMP，`U+0000 ~ U+FFFF`）**：占 2 字节，一个 `char` **可以**存下。
  - **生僻字（扩展区汉字）和绝大多数 Emoji（增补平面，`U+10000 ~ U+10FFFF`，如 🚀）**：在 UTF-16 中需要 **4 个字节（即 1 对代理对 Surrogate Pair，占用 2 个 `char`）**，单个 `char` **无法存储**！
- **实战避坑**：
  - `"🚀".length()` 返回的是 **`2`** 而不是 `1`！如果对包含 Emoji 的用户昵称直接调用 `str.substring(0, 1)`，会把 4 字节代理对生生劈成一半，导致前端展示出**乱码字符 ``**，存入 MySQL `utf8mb4` 时甚至可能报错！
  - **正确姿势**：使用 `str.codePointCount(0, str.length())` 获取真实字符个数，按码点偏移量 `str.offsetByCodePoints(0, count)` 安全截取。

---

## 3. String 家族与关键字语义（Q13 ~ Q19）

### Q13：`String` 为什么被设计为不可变的？JDK 8 到 JDK 9 底层做了什么优化？

- **直接结论**：`String` 一旦创建，其内部字符序列内容永不可更改（任何修改操作都会返回新 `String`）。
- **不可变的 4 个源码保障**：
  1. **数组 `final` + `private`**：JDK 8 为 `private final char value[]`；JDK 9+ 改为 `private final byte[] value` + `private final byte coder`。
  2. **类本身 `final`**：`public final class String`，防止子类继承后重写方法破坏不可变契约。
  3. **构造器防御性拷贝**：外部传入 `char[]`/`byte[]` 时，构造器内部执行 `Arrays.copyOf()` 复制一份，外部改原数组不影响 `String`。
  4. **无暴露修改接口**：`substring()`、`replace()` 等方法均新建 `String` 返回。
- **为什么要设计成不可变？**
  1. **支持字符串常量池**：相同字面量可以安全共享同一对象，节省海量内存；如果可变，改一个就会连带改掉所有引用处。
  2. **天然线程安全**：多线程无需加锁同步即可安全共享。
  3. **缓存 HashCode**：`private int hash;` 首次计算后直接缓存，使 `String` 成为 `HashMap` 效率最高的 Key。
  4. **安全性保障**：文件路径、网络连接 URL、数据库凭证、反射类名在安全校验通过后不可能被中途篡改。
- **JDK 9 Compact Strings（紧凑字符串）优化**：
  - Oracle 统计大量生产应用堆内存发现 `String` 占比常达 25% 以上，且绝大多数字符串仅包含英文、数字等单字节字符。
  - JDK 9 将 `char[]`（每字符 2B）改为 **`byte[]` + 编码标志 `coder`**（`LATIN1 = 0` 占 1 字节/字符；`UTF16 = 1` 占 2 字节/字符），纯英文字符串内存占用**直接减半**，大幅减轻 GC 压力！

---

### Q14：`String`、`StringBuilder`、`StringBuffer` 的区别与源码扩容机制？

- **核心对比**：

| 对比维度 | `String` | `StringBuilder` | `StringBuffer` |
| :--- | :--- | :--- | :--- |
| **底层可变性** | 不可变 | 可变（继承 `AbstractStringBuilder`） | 可变（继承 `AbstractStringBuilder`） |
| **线程安全** | 安全（不可变） | **不安全** | **安全**（几乎所有公开方法加 `synchronized`） |
| **性能表现** | 频繁拼接极慢（产生大量垃圾对象） | **最高** | 略低于 `StringBuilder`（锁开销，但单线程可被 JIT 锁消除） |
| **适用场景** | 常量、少次拼接 | 单线程/方法局部变量大量动态拼接（**首选**） | 多线程并发共享修改同一字符串缓冲区 |

- **源码扩容机制（`AbstractStringBuilder.ensureCapacityInternal` -> `newCapacity`）**：
  - 无参构造默认初始容量为 **`16`**（`new byte[16]`）。
  - 当 `append()` 导致 `count + len > value.length` 时触发扩容：
    1. 计算 **`newCapacity = (oldCapacity << 1) + 2`**，即 **旧容量的 2 倍 + 2**。
    2. 若 `newCapacity` 仍小于所需最小容量 `minCapacity`，则直接取 `minCapacity`。
    3. 若超过 `SOFT_MAX_ARRAY_LENGTH`（`Integer.MAX_VALUE - 8`），进入 `hugeCapacity` 兜底处理（最大 `Integer.MAX_VALUE`，再大抛 `OutOfMemoryError`）。
    4. 调用 `Arrays.copyOf(value, newCapacity)` 完成数据迁移。
- **实战避坑**：在拼接大 JSON、大 SQL 或导出 CSV 文本时，根据业务估算长度显式指定构造容量 `new StringBuilder(1024)`，避免从 16 -> 34 -> 70 -> 142... 一路频繁扩容拷贝数组。

---

### Q15：`String s = new String("abc")` 创建了几个对象？字符串 `+` 拼接底层怎么优化的？

- **`new String("abc")` 创建几个对象？**
  - **情况 1（常量池中尚无 `"abc"`）**：创建 **2 个对象**。① 类加载或执行到字节码 `ldc` 时，在字符串常量池创建一个 `"abc"` 对象；② `new String(...)` 在堆中再创建一个 `String` 对象（注意：构造器源码 `this.value = original.value;` 会直接复用常量池 `"abc"` 内部的字符数组，不会再拷贝一份数组！）。
  - **情况 2（常量池中已有 `"abc"`）**：仅在堆中创建 **1 个**新 `String` 对象。
- **字符串 `+` 拼接的编译器与 JVM 优化演进**：
  1. **纯字面量相加**：`String s = "a" + "b" + "c";` 在编译期直接被**常量折叠**优化为 `String s = "abc";`，运行期零开销。
  2. **JDK 8 变量相加**：`String s = a + b;` 编译器隐式改写为 `new StringBuilder().append(a).append(b).toString()`。
     - **循环拼接陷阱**：如果在 `for` 循环中写 `res += i;`，字节码会在**每次循环体内部都新建一个 `StringBuilder`** 并 `toString()`，1 万次循环产生上万个垃圾对象！
  3. **JDK 9+ 变量相加（JEP 280）**：改用 **`invokedynamic`** 指令调用 **`StringConcatFactory.makeConcatWithConstants()`**！运行期动态计算所有拼接片段总长度及编码类型（Latin1 还是 UTF16），一次性分配精确大小的 `byte[]` 并直接填入内容生成 `String`，**彻底省去 `StringBuilder` 对象创建及中间扩容开销**（但循环体内的 `+=` 仍会每次生成新 `String`，循环拼接依然必须手动在循环外创建 `StringBuilder`）。

---

### Q16：`String.intern()` 在 JDK 6、JDK 7/8、JDK 9+ 的底层原理与经典笔试题？

- **直接结论**：`intern()` 是一个 `native` 方法，作用是“把字符串驻留到字符串常量池（`StringTable`，底层为 C++ 实现的定长桶哈希表）中并返回池中唯一引用”。
- **JDK 6 vs JDK 7/8+ 的核心差异**：
  - **JDK 6**：`StringTable` 在**永久代（PermGen）**，与堆隔离。调用 `s.intern()` 若池中无该字符串，会**把堆里的字符串对象完整深拷贝一份**放入永久代常量池，并返回永久代地址。
  - **JDK 7 / 8+**：`StringTable` 被移入**堆（Heap）**中！调用 `s.intern()` 若池中无该字符串，**不再复制对象，而是直接把堆中该对象的引用地址登记到 `StringTable` 哈希表里**并返回该引用！
- **经典笔试题深度解析（JDK 7/8+ 环境）**：

```java
// 题目 1：
String s1 = new String("a") + new String("b"); // 常量池产生 "a", "b"；堆中拼接产生 "ab"（常量池此时无 "ab"！）
String s2 = s1.intern();                       // 将堆中 s1 的引用直接登记进常量池
System.out.println(s1 == "ab");                // true（字面量 "ab" 查常量池拿到的就是 s1 引用）
System.out.println(s1 == s2);                  // true

// 题目 2（陷阱变式）：
String s3 = new String("1") + new String("1"); // 堆中产生 "11"
String s4 = "11";                              // 字面量先在常量池创建了 "11"
s3.intern();                                   // 常量池已有，什么都不做返回 s4
System.out.println(s3 == s4);                  // false！

// 题目 3（终极灵魂拷问）：
String s5 = new StringBuilder("ja").append("va").toString();
System.out.println(s5.intern() == s5);         // false！因为 JVM 启动初始化 sun.misc.Version 时早已把 "java" 放进常量池了！
```

- **实战避坑**：不要对海量随机无规律字符串（如订单流水号、UUID）调用 `intern()`！`StringTable` 桶数固定（JDK 8 默认 `-XX:StringTableSize=60013`），滥用会导致严重哈希碰撞拉长链表，并使 Young GC / Full GC 扫描 `StringTable` 耗时激增引发停顿（STW）。

---

### Q17：`final`、`finally`、`finalize` 的区别？`final` 在 JMM 中的可见性语义？

- **三者区别**：
  - **`final`**：修饰类（不可被继承，如 `String`/`Integer`）、修饰方法（不可被子类重写）、修饰变量（基本类型值不可变；引用类型指向的地址不可改，但对象内部属性依然可变）。
  - **`finally`**：异常处理收尾块，通常用于释放资源，无论是否发生异常都会执行（除非遇到 `System.exit(0)` 或线程被强杀/断电）。
  - **`finalize`**：`Object` 方法（JDK 9 已废弃），GC 回收前的清理钩子。
- **`final` 的并发安全与 JMM（Java 内存模型）重排序规则**：
  - 很多人只知道 `volatile` 防重排序，却忽略了 **`final` 也有极强的 JMM 并发语义（安全发布）**！
  - JMM 规定两项 `final` 域重排序规则：
    1. **写 `final` 域重排序规则**：在构造器内对一个 `final` 域的写入，与随后把被构造对象的引用赋值给一个引用变量，这两个操作之间**禁止重排序**（编译器会在构造器 `return` 前插入 **`StoreStore` 内存屏障**）。
    2. **读 `final` 域重排序规则**：初次读一个包含 `final` 域的对象引用，与随后初次读这个 `final` 域，禁止重排序（插入 `LoadLoad` 屏障）。
  - **工程价值**：只要对象在构造器内没有发生 `this` 引用逸出，任何其他线程一旦拿到该对象的引用，看到的 `final` 字段**必然是已经初始化完毕的值**，无需 `volatile` 或锁同步（`String` 的多线程安全就部分依赖构造器无 `this` 逸出 + `final` 域保障）。
- **实战避坑**：在 Lambda 或匿名内部类中引用外部局部变量时，该变量必须声明为 `final` 或 **effectively final（事实不可变，即赋值后从未再改过）**——因为局部变量的生命周期随栈帧销毁而结束，Lambda 捕获的其实是外部变量值的副本。

---

### Q18：`static` 关键字深入：静态变量存在哪？静态内部类 vs 非静态内部类的内存泄漏陷阱？

- **静态变量到底存放在 JVM 哪块内存？**
  - **JDK 7 及以前**：随类信息存放在**永久代（方法区）**。
  - **JDK 8 及以后**：静态变量（`static` 修饰的字段及引用本身）随同该类的 **`java.lang.Class` 对象**一起存放在**堆（Heap）**中；而类的结构元数据（常量池、方法字节码、字段描述等）存放在本地内存的**元空间（Metaspace）**。
- **静态内部类 vs 非静态（成员）内部类的核心区别**：

| 对比维度 | 静态内部类 (`static class`) | 非静态内部类 (`class`) |
| :--- | :--- | :--- |
| **外部类引用** | **无**隐式外部类引用，独立存在 | **隐式持有外部类实例引用（字节码中合成 `final Outer this$0`）** |
| **创建方式** | `new Outer.StaticInner()` | 必须依赖外部实例：`outer.new Inner()` |
| **访问成员** | 只能访问外部类的 `static` 成员 | 可访问外部类的所有成员（含私有实例字段） |
| **定义静态成员** | 可自由定义静态字段/方法 | JDK 16 之前禁止定义静态成员（编译期常量除外） |

- **实战避坑（极其隐蔽的内存泄漏源）**：
  - 在异步回调、线程池提交任务、或向全局事件总线注册监听器时，如果使用了**非静态内部类或匿名内部类**（注意：非静态方法中的匿名内部类同样持有 `this$0`！），它会把外部大对象（如 Spring Service、Controller 上下文、甚至包含大数组的业务处理器）死死拽住，导致任务不结束外部对象就无法被 GC 回收！**能用 `static class` 或无状态静态 Lambda 时，绝不用普通内部类**。

---

### Q19：`transient` 关键字的作用是什么？对 `static` 字段和 JSON 框架有效吗？

- **直接结论**：`transient` 只能修饰成员变量，表示在执行 **Java 原生序列化（`ObjectOutputStream`）** 时忽略该字段，反序列化恢复时该字段会被赋予类型默认值（`0`/`false`/`null`）。
- **面试官连环追问**：
  - **追问 1**：`static` 静态变量需要用 `transient` 修饰吗？
    - **回答**：不需要。Java 原生序列化保存的是**对象堆实例的状态**，而 `static` 变量属于**类全局状态**，原本就不会被序列化。反序列化后读到的也是当前 JVM 中该类的静态变量值。
  - **追问 2**：`ArrayList` 源码中 `transient Object[] elementData;` 加了 `transient`，为什么序列化后元素不会丢失？
    - **回答**：`elementData` 数组通常有大量未装满的 `null` 空位（如容量 1000 只存了 10 个元素），直接全量序列化会严重浪费带宽与磁盘。`ArrayList` 自己定义了私有的 `writeObject(ObjectOutputStream s)` 和 `readObject(ObjectInputStream s)` 方法，在序列化时仅遍历写出真实存在的 `size` 个元素！`HashMap` 的 `table` 数组也是同样优化思路。
  - **追问 3**：`transient` 对 Jackson、Fastjson 等 JSON 序列化有效吗？
    - **回答**：默认行为不一致！Fastjson 默认忽略 `transient` 字段；而 **Jackson 默认不会忽略 `transient` 字段**（除非配置 `MapperFeature.PROPAGATE_TRANSIENT_MARKER`）。生产中屏蔽 JSON 字段请统一使用 **`@JsonIgnore`**，不要混用 `transient`。

---

## 4. 拷贝、序列化、SPI 与枚举（Q20 ~ Q23）

### Q20：深拷贝与浅拷贝的区别？生产中如何实现深拷贝？

- **核心区别**：
  - **浅拷贝**：创建一个新对象，基本类型拷贝数值，**引用类型字段仅拷贝引用地址**（新旧对象共享同一个子对象，改一个影响另一个）。`Object.clone()` 默认就是浅拷贝。
  - **深拷贝**：不仅创建新外层对象，其内部嵌套的所有引用类型子对象也会层层递归创建出全新副本，新旧对象完全隔离。
- **生产级实现方案对比**：
  1. **重写 `clone()` 层层克隆**：必须让整条引用链上的类全实现 `Cloneable` 并重写 `clone()`，维护成本极高，极易遗漏（不推荐）。
  2. **序列化/反序列化（`Serializable` 字节流 / Jackson JSON / Kryo）**：代码通用性强；缺点是原生 `Serializable` 慢，JSON 对复杂泛型、日期、循环引用需额外适配。
  3. **拷贝构造器 / `MapStruct` 编译期代码生成（最推荐）**：`MapStruct` 在编译期生成纯 `get`/`set` 拷贝代码，**零反射、性能与手写代码完全一致且编译期类型校验**，是业务系统 DO/DTO/VO 拷贝的绝对首选。
- **实战避坑**：Spring 的 `BeanUtils.copyProperties(src, target)` 与 Apache `BeanUtils.copyProperties` **全都是浅拷贝**！且 Apache 版本内部包含大量反射与日志校验，性能极其低下（《阿里规约》明文禁止使用 Apache `BeanUtils`）。

---

### Q21：Java 序列化原理、`serialVersionUID` 作用与破坏单例的防范？

- **核心原理**：实现 `java.io.Serializable` 标记接口（或 `Externalizable` 手动控制读写）后，`ObjectOutputStream` 按类描述符元数据写出字段流，接收端通过 `ObjectInputStream` 还原。
- **`serialVersionUID` 的作用**：
  - 序列化版本指纹。反序列化时会校验字节流中的 `serialVersionUID` 与本地类定义的是否一致；若不一致则直接抛出 **`InvalidClassException`**。
  - **未显式声明的风险**：如果不手动写死 `private static final long serialVersionUID = 1L;`，JVM 会在编译期根据类名、字段、方法签名自动计算一个 hash；只要后续需求迭代**增删哪怕一个字段或方法**，UID 就会改变，导致缓存（如 Redis）或消息队列里的旧版本字节流全部反序列化失败！
- **反序列化如何破坏单例及如何防范？**
  - **破坏原因**：`ObjectInputStream.readObject()` 底层通过反射调用无参构造机制**绕过 `getInstance()` 直接在堆上分配一个新实例**，导致 `instance != deserializedInstance`。
  - **防范手段**：在单例类中定义如下私有方法：
    ```java
    private Object readResolve() {
        return INSTANCE; // readObject 结束前会自动反射检查并调用此方法替换返回值
    }
    ```
    （或者直接使用**枚举单例**，天然防反序列化和防反射破坏！）
- **面试官连环追问**：
  - **追问**：为什么分布式 RPC 和缓存架构不建议用 Java 原生序列化？
    - **回答**：① **安全漏洞**：历史上有大量反序列化 RCE 远程代码执行漏洞（利用 `readObject` 构造 Gadget 链）；② **不能跨语言**；③ **码流体积大、序列化耗时高**（Protobuf / Kryo / Hessian2 体积只有原生的 1/3 ~ 1/5，速度快数倍）。

---

### Q22：什么是 SPI 机制？`ServiceLoader` 底层原理与 JDBC/Dubbo 对比？

- **直接结论**：**SPI（Service Provider Interface）** 是 Java 内置的面向接口的服务发现与可插拔扩展机制，体现**控制反转（IoC）**思想：由**调用方/标准制定方**定义接口，**第三方服务提供商**实现接口并在 jar 包内声明，框架在运行时自动扫描加载实现类。
- **`ServiceLoader` 底层源码流程**：
  1. 厂商在自己 jar 包的 **`META-INF/services/接口全限定名`** 文件中，按行写入实现类的全限定名。
  2. 调用方执行 `ServiceLoader<Driver> loader = ServiceLoader.load(Driver.class);`。
  3. 源码中构造了内部懒加载迭代器 `LazyIterator`，此时**尚未读取文件也未加载类**。
  4. 调用 `iterator.hasNext()` 时，通过当前线程上下文类加载器（`Thread.currentThread().getContextClassLoader()`）读取所有 jar 包中对应的 `META-INF/services/...` 文件并解析类名。
  5. 调用 `iterator.next()` 时，通过 `Class.forName(cn, false, loader)` 加载类并调用无参构造器 `newInstance()` 实例化，存入缓存 `providers` 中返回。
- **面试官连环追问**：
  - **追问 1**：JDBC `DriverManager` 为什么必须用线程上下文类加载器（TCCL）打破双亲委派？
    - **回答**：`java.sql.DriverManager` 位于核心类库中，由**启动类加载器（BootstrapClassLoader）**加载；而 MySQL 驱动包中的 `com.mysql.cj.jdbc.Driver` 位于应用 `classpath` 下，Bootstrap 根本加载不到它！因此 `DriverManager` 只能借用 `Thread.currentThread().getContextClassLoader()`（默认为 `AppClassLoader`）逆向委派加载实现类，打破了双亲委派模型。
  - **追问 2**：JDK 原生 SPI 有什么局限？Dubbo SPI 怎么改进的？
    - **回答**：JDK `ServiceLoader` 只能通过遍历一次性实例化所有配置实现类（无法按需取某一个，造成资源浪费），且不支持 IOC 和 AOP 包装。Dubbo SPI 改用 `key=value` 配置格式，支持**按 Name 精准懒加载**、自适应扩展（`@Adaptive` 动态编译代理）、IOC 自动注入依赖以及 Wrapper 装饰器 AOP 增强。

---

### Q23：Java 枚举（`enum`）底层原理是什么？为什么说枚举是实现单例的最佳方式？

- **底层实现原理**：
  - `enum` 是编译器语法糖。编译后用 `javap` 反编译可以看到，它本质上是一个**继承自 `java.lang.Enum<E>` 的 `final class`**。
  - 每个枚举常量都会被编译为 `public static final` 静态实例字段，在静态代码块 `static {}` 中调用私有构造方法完成初始化，并保存于内置 `VALUES` 数组中供 `values()` 返回。
- **为什么枚举单例被《Effective Java》推崇为最佳单例模式？**
  1. **天然线程安全**：静态实例在类加载初始化 `<clinit>` 阶段由 JVM 加锁保证仅创建一次。
  2. **绝对防反射攻击**：普通懒汉/饿汉/静态内部类单例，都可以通过 `Constructor.setAccessible(true)` 反射强行 `newInstance()` 创建第二个实例；而 JDK **`Constructor.newInstance()` 源码**中写死了校验：
     ```java
     if ((clazz.getModifiers() & Modifier.ENUM) != 0)
         throw new IllegalArgumentException("Cannot reflectively create enum objects");
     ```
     **直接从 JVM 语言底层彻底禁止反射实例化枚举！**
  3. **绝对防反序列化破坏**：`ObjectInputStream` 反序列化枚举时仅读取枚举名，通过 `Enum.valueOf()` 查表返回 JVM 中现有的唯一枚举对象，不会新建对象。

---

## 5. 反射、动态代理、泛型与注解（Q24 ~ Q30）

### Q24：什么是反射？获取 `Class` 的 3 种方式？反射为什么慢及 JDK 内部优化？

- **直接结论**：反射允许程序在运行时动态获取任意类的构造器、字段、方法、注解，并动态创建对象或调用方法。获取 `Class` 对象的 3 种方式：
  1. `User.class`（编译期已知，**不触发类初始化 `<clinit>`**）。
  2. `user.getClass()`（已有实例）。
  3. `Class.forName("com.example.User")`（动态类名，**默认触发类初始化**；可用三参数重载关闭）。
- **反射为什么比直接调用慢？**
  1. **无法享受 JIT 深度优化**：动态解析阻碍了方法内联（Method Inlining）与逃逸分析。
  2. **额外包装与拆装箱**：入参需打包为 `Object[]` 数组，基本类型需频繁装箱拆箱，返回值需类型校验强转。
  3. **访问权限校验**：每次调用默认检查修饰符权限（`setAccessible(true)` 可跳过校验大幅提速）。
- **JDK 内部的黑科技优化**：
  - **JDK 8 ~ 17（Inflation 膨胀机制）**：`Method.invoke()` 前 **15 次**（由 `-Dsun.reflect.inflationThreshold=15` 控制）走较慢的本地方法 `NativeMethodAccessorImpl`（省去生成字节码的启动开销）；一旦某个方法反射调用达到 **15 次**，JVM 会动态在内存中生成字节码类 `GeneratedMethodAccessorN`，内部直接转为正常的 `invokevirtual` 字节码调用，之后性能极其接近直接调用！
  - **JDK 18+（JEP 416）**：废除了老旧的字节码动态生成反射访问器，全面改用 **`MethodHandle`（方法句柄）** 实现反射底层，消除额外类加载导致的元空间占用，JIT 友好度再上台阶。
- **实战避坑**：切忌在高频接口循环体内部反复调用 `clazz.getDeclaredMethod(...)`！类元数据查找很耗时，应当配合 `ConcurrentHashMap` 将 `Method` / `Field`（且提前 `setAccessible(true)`）缓存起来复用（或使用 Spring `ReflectionUtils` / `ReflectASM`）。

---

### Q25：`MethodHandle`（方法句柄）与传统反射（`Method`）有什么区别？

- **核心对比**：

| 对比维度 | 传统反射 (`java.lang.reflect.Method`) | 方法句柄 (`java.lang.invoke.MethodHandle`) |
| :--- | :--- | :--- |
| **设计层次** | **Java 语言层**（重量级，含注解、泛型签名等丰富元数据） | **JVM 字节码层**（轻量级，直接模拟 `invokevirtual`/`invokestatic` 等指令） |
| **权限检查时机** | **每次 `invoke()` 时检查**（除非 `setAccessible(true)`） | 在通过 `MethodHandles.Lookup` **查找创建句柄时一次性完成检查** |
| **JIT 与内联** | 早期较难被 JIT 内联优化 | 极易被 JIT 内联，且可直接配合 **`invokedynamic`** 指令工作 |
| **参数适配** | 固定传 `Object...` 变长数组 | 支持 `MethodType` 强类型匹配、参数绑定与适配器组合（`bindTo`/`asType`） |

- **底层地位**：Java 8 的 **Lambda 表达式**、JDK 9 的字符串动态拼接、JDK 17+ 模式匹配、JDK 18 的新版反射底层全都在使用 `MethodHandle` + `invokedynamic`。

---

### Q26：JDK 动态代理和 CGLIB 动态代理的区别？（附完整代码与陷阱）

- **深度对比**：

| 对比维度 | JDK 动态代理 | CGLIB 动态代理 |
| :--- | :--- | :--- |
| **实现原理** | 运行期生成实现目标接口的代理类字节码 `$Proxy0 extends Proxy implements TargetInterface` | 基于 **ASM** 字节码框架，生成目标类的**子类** `Target$$EnhancerByCGLIB` 并重写非 `final` 方法 |
| **目标类限制** | **必须实现至少一个接口**（因为代理类已单继承了 `java.lang.reflect.Proxy`） | 无接口要求，但目标类**不能为 `final`**，被代理方法**不能为 `final` 或 `static`** |
| **调用机制** | 通过 `InvocationHandler.invoke()` 反射调用目标方法 | 基于 **FastClass** 索引机制：为代理类与目标类各生成索引，通过方法签名哈希定位整数下标直接 `switch` 调用，避免反射开销 |
| **Spring 默认策略** | Spring 5 / Boot 1.x：有接口走 JDK，无接口走 CGLIB；**Spring Boot 2.x / 3.x 默认统一走 CGLIB**（`spring.aop.proxy-target-class=true`） |

- **核心实战代码**：

```java
// 1. JDK 动态代理（基于接口）
public class JdkProxyHandler implements InvocationHandler {
    private final Object target;
    public JdkProxyHandler(Object target) { this.target = target; }

    @SuppressWarnings("unchecked")
    public static <T> T createProxy(T target) {
        return (T) Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new JdkProxyHandler(target)
        );
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("[JDK] 前置增强: " + method.getName());
        Object result = method.invoke(target, args);
        System.out.println("[JDK] 后置增强");
        return result;
    }
}

// 2. CGLIB 动态代理（基于子类继承）
public class CglibInterceptor implements MethodInterceptor {
    @SuppressWarnings("unchecked")
    public static <T> T createProxy(Class<T> targetClass) {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(targetClass);
        enhancer.setCallback(new CglibInterceptor());
        return (T) enhancer.create();
    }

    @Override
    public Object intercept(Object proxyObj, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("[CGLIB] 前置增强: " + method.getName());
        // 关键：必须调用 invokeSuper 调用父类原方法，若误调 methodProxy.invoke(proxyObj, args) 会无限递归爆栈！
        Object result = methodProxy.invokeSuper(proxyObj, args);
        System.out.println("[CGLIB] 后置增强");
        return result;
    }
}
```

- **实战避坑**：
  1. **Spring AOP 同类内部自调用失效**：在 `OrderService.createOrder()` 中直接 `this.updateStock()`（带有 `@Transactional` 或 `@Async`），由于 `this` 指向原始目标对象而非代理对象，**JDK 和 CGLIB 代理都会失效**！解法：注入自身代理对象（或 `AopContext.currentProxy()`）或拆分到独立 Service 类中。
  2. **JDK 17+ 模块强封装问题**：老版本独立引入的 CGLIB 在 JDK 17+ 上调用 `ClassLoader.defineClass` 会报 `InaccessibleObjectException`（Spring 6 / Boot 3 已内置新版 Spring CGLIB 完美兼容，若老项目升级需加 JVM 参数 `--add-opens java.base/java.lang=ALL-UNNAMED`）。

---

### Q27：什么是泛型与类型擦除（Type Erasure）？运行时如何获取真实泛型类型？

- **直接结论**：Java 泛型是**编译期伪泛型**（为了二进制向后兼容 JDK 1.4 之前无泛型的字节码）。编译成 `.class` 后，类型参数 `<T>` 会被擦除为它的**上界**（未显式指定则擦除为 `Object`，`<T extends Number>` 则擦除为 `Number`），并在取值处自动插入强转字节码 `checkcast`。
- **类型擦除带来的 5 大限制**：
  1. 不能用基本类型作为泛型实参（如 `List<int>` 非法，必须 `List<Integer>`）。
  2. 不能实例化类型参数 `new T()` 或创建泛型数组 `new T[10]`。
  3. 静态字段与静态方法不能直接引用**类级别**的泛型参数 `T`（静态成员随类加载，早于对象实例化确定 `T`）。
  4. 不能直接 `if (list instanceof List<String>)`。
  5. 运行时 `new ArrayList<String>().getClass() == new ArrayList<Integer>().getClass()` 结果为 **`true`**。
- **既然被擦除了，为什么 Fastjson / Jackson / MyBatis 还能在运行时拿到泛型真实类型？**
  - **关键机制**：类型擦除只擦除方法体局部变量和实例化时的信息；但**类的定义、父类继承、成员变量、方法参数与返回值上的泛型签名**被完整保留在字节码的 **`Signature` 属性表**中！
  - **经典实战（`TypeReference` 匿名子类技巧）**：

```java
// 父类定义一个抽象泛型基类
abstract class TypeReference<T> {
    private final Type type;
    protected TypeReference() {
        // 获取带泛型参数的父类类型（即 ParameterizedType）
        Type superClass = getClass().getGenericSuperclass();
        this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
    }
    public Type getType() { return type; }
}

// 使用时创建一个匿名子类，字节码 Signature 会固化 List<String> 信息！
Type realType = new TypeReference<List<String>>() {}.getType(); // 运行时成功拿到 java.util.List<java.lang.String>
```

---

### Q28：泛型通配符 `?`、`extends`、`super` 与 PECS 原则是什么？

- **前提认知（泛型的不变性 Invariant）**：虽然 `Integer` 是 `Number` 的子类，但 `List<Integer>` **不是** `List<Number>` 的子类！为了解决容器的协变与逆变需求，Java 引入了边界通配符：
  - **`<?>`（无界通配符）**：只读不写，读出的元素类型退化为 `Object`（除了 `null` 不能添加任何元素）。
  - **`<? extends T>`（上界通配符，协变）**：集合中元素是 `T` 或 `T` 的某个未知子类。**可以安全读取（当成 `T`），但禁止写入**（因为容器实际可能是 `List<Double>` 或 `List<Integer>`，编译器无法保证写入类型安全）。
  - **`<? super T>`（下界通配符，逆变）**：集合中元素是 `T` 或 `T` 的某个未知父类。**可以安全写入 `T` 及其子类**，但读取出来只能当成 `Object`。
- **PECS 黄金法则（《Effective Java》）**：
  - **Producer Extends（生产者用 `extends`）**：如果你只需要从集合中**读取/遍历元素**（集合生产数据给你），用 `<? extends T>`。
  - **Consumer Super（消费者用 `super`）**：如果你需要往集合中**写入/添加元素**（集合消费你的数据），用 `<? super T>`。
  - **JDK 源码典范**：`Collections.copy(List<? super T> dest, List<? extends T> src)`——`src` 只读用 `extends`，`dest` 只写用 `super`。

---

### Q29：什么是泛型桥接方法（Bridge Method）？

- **直接结论**：桥接方法是编译器为解决“**泛型擦除导致子类重写父类方法时字节码签名不一致，从而破坏多态**”而自动生成的合成方法（带 `ACC_BRIDGE` 与 `ACC_SYNTHETIC` 标志）。
- **原理演示**：

```java
interface Box<T> {
    void set(T t); // 擦除后字节码签名：set(Object t)
}
class IntBox implements Box<Integer> {
    @Override
    public void set(Integer t) { System.out.println(t); } // 实际签名：set(Integer t)
}
```

- 擦除后，`IntBox.set(Integer)` 与 `Box.set(Object)` 参数类型不同，本无法构成重写！编译器于是在 `IntBox.class` 中自动注入了一个桥接方法：

```java
// 编译器自动生成，源码不可见
public /* synthetic bridge */ void set(Object t) {
    this.set((Integer) t); // 强转后委派给真正的 set(Integer)
}
```

- **实战避坑**：当自己手写框架通过反射扫描类的所有方法（`clazz.getDeclaredMethods()`）来注册处理器或校验注解时，会多扫出编译器生成的桥接方法！务必加上 **`if (method.isBridge() || method.isSynthetic()) continue;`** 进行过滤。

---

### Q30：Java 注解的底层原理是什么？编译期注解处理器（APT）如何工作？

- **注解底层原理（运行期 `@Retention(RetentionPolicy.RUNTIME)`）**：
  - 注解本质上是一个继承了 `java.lang.annotation.Annotation` 的**特殊接口**。
  - 在运行期调用 `method.getAnnotation(MyAnno.class)` 时，JDK 底层借助 **`Proxy.newProxyInstance` 动态代理**，在内存中动态生成一个实现了 `MyAnno` 接口的代理类 **`$ProxyN`**，内部持有一个 **`AnnotationInvocationHandler`**（内部维护存放注解配置键值对的 `Map<String, Object> memberValues`）。调用注解方法如 `anno.value()` 时，实际上就是调用代理对象从该 `Map` 中取值返回！
- **4 个元注解**：`@Target`（作用范围）、`@Retention`（生命周期：`SOURCE` 源码 / `CLASS` 字节码 / `RUNTIME` 运行时）、`@Documented`（Javadoc）、`@Inherited`（仅类级别注解可被子类继承，接口和字段/方法无效）。
- **编译期注解处理器（APT, Annotation Processing Tool）**：
  - `javac` 在编译期解析源码后、生成字节码前，会自动扫描源码上的注解，回调继承了 `AbstractProcessor` 的处理器：
    - **生成新源码派（如 MapStruct、Dagger2）**：读取注解元数据后，利用 `Filer` 工具生成全新的 `.java` 转换实现类并交给编译器继续编译，运行期零反射开销。
    - **修改 AST 派（如 Lombok）**：在编译期通过内部编译器 API 直接向现有类的 **AST（抽象语法树）** 中插入 `getter`/`setter`/构造器语法树节点，再编译为字节码。

---

## 6. 异常体系与 IO / 零拷贝（Q31 ~ Q35）

### Q31：Java 异常体系架构是怎样的？`Error` vs `Exception`？受检 vs 非受检？

- **体系结构图**：

```text
java.lang.Throwable
 ├── java.lang.Error (JVM 级严重错误，程序无法恢复，不应捕获)
 │    ├── OutOfMemoryError (堆/元空间/直接内存/GC Overhead/无法创建本地线程)
 │    ├── StackOverflowError (方法调用栈过深，如无限递归)
 │    └── NoClassDefFoundError (编译期存在，运行期类加载/静态初始化失败)
 └── java.lang.Exception (程序可捕获处理)
      ├── 受检异常 Checked Exception (编译期强制 try-catch 或 throws)
      │    ├── IOException / FileNotFoundException
      │    ├── SQLException
      │    └── ClassNotFoundException
      └── RuntimeException (非受检异常 Unchecked，运行期逻辑缺陷)
           ├── NullPointerException
           ├── IndexOutOfBoundsException
           ├── IllegalArgumentException
           └── ClassCastException
```

- **面试官连环追问**：
  - **追问**：`ClassNotFoundException` 与 `NoClassDefFoundError` 有什么区别？
    - **回答**：`ClassNotFoundException` 是**受检异常**，在代码显式调用 `Class.forName()` 或 `ClassLoader.loadClass()` 动态加载类找不到时抛出；`NoClassDefFoundError` 是 **`Error`**，表示编译时明明有这个类，但运行时类路径缺 jar 包，或更常见的——**该类第一次加载时静态代码块 `<clinit>` 抛了异常导致类初始化失败，后续再次引用该类时就会直接抛 `NoClassDefFoundError`**！

---

### Q32：`try-catch-finally` 与 `return` 的执行陷阱？`try-with-resources` 原理？

- **核心执行陷阱**：
  1. 无论 `try`/`catch` 中是否有 `return`，**`finally` 都会在方法真正返回前执行**。
  2. **返回值暂存机制**：如果 `try` 块执行 `return x;`（`x` 为基本类型），JVM 会先将 `x` 的值**备份存入栈帧局部变量表的临时槽位**，再执行 `finally`，最后返回备份的值。因此 `finally` 中修改基本类型局部变量 `x = 99` **不会**改变返回值！
  3. **`finally` 中写 `return` 的灾难**：如果 `finally` 中也写 `return`，不仅会**直接覆盖掉 `try`/`catch` 的返回值**，还会**静默吞掉 `try`/`catch` 抛出的异常**，导致线上 bug 悄无声息消失！（《阿里开发手册》严禁在 `finally` 中使用 `return`）。
- **`try-with-resources`（JDK 7+）底层原理**：
  - 凡是实现 **`AutoCloseable`**（或 `Closeable`）的资源对象放在 `try (Resource r = ...)` 中，编译器会自动展开为严密的 `try-finally` 块，按声明的**逆序**自动调用 `close()`。
  - **异常屏蔽（Suppressed Exception）保护**：手写 `try-finally` 时，若 `try` 抛出业务异常 A 且 `finally` 调用 `close()` 又抛出异常 B，异常 B 会直接把异常 A 顶掉掩盖真正故障根因！而 `try-with-resources` 编译后的字节码会保留主异常 A，并将关闭异常 B 通过 **`primaryEx.addSuppressed(closeEx)`** 挂载到主异常上，排查问题时 `e.getSuppressed()` 一目了然。

---

### Q33：Java 异常处理的最佳实践与性能开销是什么？

- **为什么异常不能当做正常业务分支控制？**
  - 当执行 `new MyException()` 时，父类 `Throwable` 构造器默认会调用本地方法 **`fillInStackTrace()`**——遍历并冻结当前线程的**整个方法调用栈帧**生成堆栈快照！高并发场景下，这不仅耗时极大，还会消耗大量 CPU 与内存。
- **生产级最佳实践**：
  1. **自定义业务异常重写 `fillInStackTrace()`**：对于明确由参数校验失败、库存不足等常规业务状态触发的 `BizException`，重写该方法直接 `return this;`（或调用 `super(msg, null, false, false)` 禁用堆栈填充），**性能可提升数十倍**！
  2. **切勿吞掉异常**：不允许空的 `catch (Exception e) {}`，至少打 `log.error("xxx fail", e)` 保留堆栈。
  3. **Spring `@Transactional` 事务回滚陷阱**：Spring 声明式事务默认只对 **`RuntimeException` 和 `Error`** 回滚，遇到 `IOException` 等**受检异常默认不回滚**！务必显式声明 `@Transactional(rollbackFor = Exception.class)`。

---

### Q34：BIO、NIO、AIO 核心区别是什么？NIO 三大核心组件与堆外内存原理？

- **三大 IO 模型核心对比**：

| IO 模型 | 同步/异步 | 阻塞/非阻塞 | 线程模型 | 核心原理与适用场景 |
| :--- | :--- | :--- | :--- | :--- |
| **BIO** | 同步 | 阻塞 | 1 连接 = 1 线程 | 连接没有数据时线程挂起阻塞，海量连接会导致线程爆炸、栈内存 OOM |
| **NIO** | **同步** | **非阻塞** | 1（或少量）Selector 线程管理海量连接 | **IO 多路复用**（Linux 底层用 **`epoll`**），仅当通道真正就绪才读写；**Netty / Tomcat / Kafka 基石** |
| **AIO** | 异步 | 非阻塞 | OS 完成后主动回调 `CompletionHandler` | Windows IOCP 原生支持好，但 Linux 原生异步 IO（POSIX AIO）不成熟且无性能优势，因此 **Netty 4.x 移除了 AIO**，坚持用 NIO epoll |

- **NIO 三大核心组件**：
  1. **`Channel`（通道）**：双向传输（区别于 BIO 单向 Input/OutputStream），如 `SocketChannel`、`ServerSocketChannel`、`FileChannel`。
  2. **`Buffer`（缓冲区）**：数据容器（`capacity`、`limit`、`position`、`mark`；写入切读取需调用 `flip()`）。
     - **`DirectByteBuffer`（堆外直接内存）**：通过 `Unsafe.allocateMemory()` 直接在 **Native 本地内存**分配缓冲区（受 `-XX:MaxDirectMemorySize` 控制），省去了传统 `HeapByteBuffer` 读写网卡/磁盘前必须先从 JVM 堆拷贝到 Native 临时内存的开销（避免 GC 移动对象地址导致的拷贝）。回收依赖 `Cleaner` 虚引用机制，如果堆外内存泄漏且堆内迟迟不触发 GC，容易直接报 `OutOfMemoryError: Direct buffer memory`。
  3. **`Selector`（多路复用器）**：监听多个 `Channel` 注册的 `OP_ACCEPT` / `OP_CONNECT` / `OP_READ` / `OP_WRITE` 事件。Linux 下对应 `epoll`（红黑树存储注册句柄 + 就绪链表事件回调，时间复杂度 $O(1)$，克服了 `select`/`poll` 每次 $O(n)$ 全量轮询和 1024 句柄限制）。

---

### Q35：什么是零拷贝（Zero-Copy）？`mmap` 和 `sendfile` 有何区别？

- **直接结论**：零拷贝不是“零次拷贝”，而是指**避免 CPU 参与数据在“内核缓冲区”和“用户缓冲区”之间来回拷贝，并大幅减少用户态/内核态上下文切换次数**。
- **传统 `read() + write()` 传输文件到网卡**：**4 次上下文切换 + 4 次数据拷贝**（2 次 DMA 拷贝 + 2 次 CPU 拷贝：磁盘 -> 内核 PageCache -> 用户空间 Buffer -> 内核 SocketBuffer -> 网卡）。
- **两大零拷贝技术路线对比**：

```text
1. mmap + write (MappedByteBuffer): 4 次上下文切换 + 3 次拷贝 (2 DMA + 1 CPU)
   磁盘 ──(DMA)──▶ 内核 PageCache ◀──(虚拟地址映射共享)──▶ 用户态进程
                        │
                     (CPU 拷贝)
                        ▼
                   SocketBuffer ──(DMA)──▶ 网卡

2. sendfile + SG-DMA (FileChannel.transferTo): 2 次上下文切换 + 2 次拷贝 (2 DMA + 0 CPU!)
   磁盘 ──(DMA)──▶ 内核 PageCache ────────────(SG-DMA 直接从 PageCache 搬运!)────────────▶ 网卡
                        │                                                                   ▲
                   (仅写描述符地址/长度) ──▶ SocketBuffer ───────────────────────────────────┘
```

- **选型对比**：
  - **`mmap`（`FileChannel.map()`）**：用户空间可直接通过指针读写映射的内核页，适合**中小文件**或**需要应用程序修改数据内容**的场景（如 **RocketMQ** CommitLog 顺序写盘，单文件通常 1GB）。
  - **`sendfile`（`FileChannel.transferTo()`）**：数据全程不经过用户空间，**0 次 CPU 拷贝**，性能极致，适合**纯文件转发/静态资源/大文件下载**（如 **Kafka** Broker 向消费者推送消息、Nginx）。
- **实战避坑**：`MappedByteBuffer` 存在无法及时释放句柄的坑（`close()` 不会立刻解除映射，需等 GC 触发 `Cleaner` 或反射调 `DirectBuffer.cleaner().clean()`），否则在 Windows 下文件会被锁定无法删除。

---

## 7. JDK 8 ~ JDK 21 核心新特性（Q36 ~ Q40）

### Q36：Lambda 表达式底层实现原理是什么？与匿名内部类有何区别？

- **直接结论**：Lambda **不是**匿名内部类的语法糖！它在底层采用 **`invokedynamic` 字节码指令 + `LambdaMetafactory`** 动态生成实现类。
- **底层机制对比**：
  1. **匿名内部类**：编译期直接生成一个独立的 `.class` 文件（如 `Outer$1.class`），类加载期必加载解析，占用磁盘与元空间内存。
  2. **Lambda 表达式**：编译期在当前类中生成一个私有的静态（或实例）方法（如 `lambda$main$0`）存放 Lambda 体代码，并在调用处生成一条 **`invokedynamic`** 指令。只有在**运行期第一次执行到该行代码**时，JVM 才调用引导方法 `LambdaMetafactory.metafactory()` 动态生成函数式接口实现类并绑定调用点（无状态捕获的 Lambda 还会直接缓存成单例，零额外对象分配！）。
- **面试官连环追问**：
  - **追问**：JDK 8 常用四大内置函数式接口是哪四个？
    - **回答**：`Function<T, R>`（接受 T 返回 R）、`Consumer<T>`（接受 T 无返回）、`Supplier<T>`（无参返回 T）、`Predicate<T>`（接受 T 返回 `boolean` 判断）。

---

### Q37：Stream API 底层原理是什么？使用 `parallelStream()` 有哪些陷阱？

- **核心原理（惰性求值 + 操作融合）**：
  - Stream 分为**中间操作**（`filter`/`map`/`sorted`，无状态/有状态）和**终端操作**（`collect`/`forEach`/`count`/`findFirst`）。
  - 中间操作不会触发遍历计算，仅在底层双向链表管道（`AbstractPipeline`）上注册一个阶段节点；只有执行**终端操作**时，才会把整条链上的操作包装成 **`Sink` 责任链**，对集合进行**单遍遍历（Loop Fusion 循环融合）**完成计算（如遇到 `findFirst`/`anyMatch` 等短路操作还会提前中断遍历）。
- **`parallelStream()` 并行流四大避坑**：
  1. **共享全局 `ForkJoinPool.commonPool()` 阻塞灾难**：所有 `parallelStream()` 与未传自定义线程池的 `CompletableFuture` 共用同一个默认公共池（线程数为 `CPU核数 - 1`）。**严禁在并行流中执行数据库查询、RPC/HTTP 等阻塞 IO**，否则会饿死全应用的所有并行任务！
  2. **线程安全问题**：禁止在 `parallelStream().forEach()` 中 `add` 到普通 `ArrayList`/`HashMap`，应使用 `collect(Collectors.toList())` 进行线程安全的并发规约。
  3. **小数据量反而更慢**：任务拆分（`Spliterator`）与线程上下文切换开销大于串行遍历。
  4. **`ThreadLocal` 上下文丢失**：公共池的工作线程不是原 HTTP 请求线程，MDC 日志追踪 ID、登录用户信息会直接丢失。

---

### Q38：`Optional` 正确用法与避坑？JDK 8 全新日期时间 API 有何优势？

- **`Optional` 四大核心避坑**：
  1. **禁止未判空直接 `get()`**：和直接用 `null` 调方法一样会抛 `NoSuchElementException`。应使用 `orElseGet()`、`orElseThrow()`、`map()`、`ifPresent()`。
  2. **分清 `orElse(val)` 与 `orElseGet(Supplier)`**：**即使 `Optional` 不为空，`orElse(createDefault())` 括号里的 `createDefault()` 依然会被立即求值执行！**（极其隐蔽的性能与副作用坑，如在 `orElse` 里调用了数据库插入默认记录！）。务必改用懒加载的 **`orElseGet(() -> createDefault())`**。
  3. **禁止用 `Optional` 做类字段、方法入参或集合元素**：`Optional` 未实现 `Serializable` 接口且增加多余对象包装。
  4. **禁止用 `Optional.ofNullable(x).isPresent()` 代替 `if (x != null)`**：徒增对象创建还毫无可读性，应当使用 `map`/`flatMap` 链式处理。
- **JDK 8 日期时间 API（`java.time`）**：
  - **解决旧 API 致命痛点**：旧版 `java.util.Date` 是可变对象且年份从 1900 起、月份从 0 起极易出错；**`SimpleDateFormat` 非线程安全**（内部共享一个可变的 `Calendar` 字段，多线程并发 `parse`/`format` 会直接把日期串改或抛出 `NumberFormatException`！）。
  - **新版优势**：`LocalDate`、`LocalTime`、`LocalDateTime`、`Instant`、`ZonedDateTime` 与 **`DateTimeFormatter`** 全都是 **`final` 不可变类，天然线程安全**，且支持链式直观操作（`plusDays`、`with`、`TemporalAdjusters`、`Duration`）。

---

### Q39：JDK 11 ~ JDK 17 有哪些核心必考新特性？

- **JDK 11（LTS）重点特性**：
  1. **`var` 局部变量类型推断**（JDK 10 引入，JDK 11 支持 Lambda 参数）：编译期推断，字节码仍是强类型；不能用于成员变量、方法参数/返回值、也不能 `var x = null;`。
  2. **内置标准化 `HttpClient`**：原生支持 HTTP/2 多路复用、WebSocket 与异步 `CompletableFuture`。
  3. **String 增强 API**：`isBlank()`（判断纯空白符）、`strip()`（支持全角空格等 Unicode 空白符，比 `trim()` 更强）、`lines()`、`repeat(n)`。
  4. **ZGC 实验性引入**：可伸缩低延迟垃圾收集器。
- **JDK 17（LTS，Spring Boot 3 最低基线）重点特性**：
  1. **`Record`（记录类）**：`public record UserDTO(Long id, String name) {}`。隐式为 `final` 类，字段全为 `private final`，自动生成全参构造器、无 `get` 前缀访问器（`id()`/`name()`）、`equals`/`hashCode`/`toString`，天生不可变极适合做 DTO/VO/值对象。
  2. **`Sealed Classes`（密封类）**：`public sealed class PayResult permits Success, Fail {}`。精确限制允许继承的子类型，子类必须显式声明为 `final`、`sealed` 或 `non-sealed`。
  3. **文本块（Text Blocks）**：`"""..."""` 多行字符串自动管理缩进对齐，写 SQL、JSON 告别反斜杠拼接地狱。
  4. **`instanceof` 模式匹配 & `switch` 表达式**：`if (obj instanceof String s && s.length() > 3)` 省略强转；`switch` 支持 `case A, B -> value;` 表达式返回并防穿透。
  5. **强封装 JDK 内部 API & ZGC 转正**：默认封闭 `sun.misc.*` 等内部包的反射访问（反射非法修改 JDK 内部字段需配 `--add-opens`）；ZGC 正式生产可用。

---

### Q40：JDK 21（LTS）虚拟线程（Virtual Threads）底层原理是什么？有哪些避坑点？

- **直接结论**：虚拟线程（Project Loom）是由 **JVM 调度管理的用户态轻量级线程**（M:N 调度：百万级虚拟线程映射到少量 OS 平台载体线程 Carrier Thread 上），初始内存仅几百字节，专为解决 **高并发 IO 密集型场景**下传统 `Thread`（1:1 绑定内核线程、1MB 栈、切换成本高）的并发瓶颈。
- **底层调度原理（Continuation + FIFO ForkJoinPool）**：

```text
  海量虚拟线程 (VirtualThread = Continuation + Runnable)
        │  (挂载 Mount)                    ▲  (挂起 Unmount：栈帧拷贝到堆 Heap 释放载体!)
        ▼                                  │
  Carrier Thread 池 (FIFO 模式的 ForkJoinPool，默认线程数 = CPU 核数)
        │
        ▼ (1:1 绑定)
  OS 内核线程 (Platform Thread)
```

  - 当虚拟线程执行阻塞操作（如 Socket 网络读写、数据库查询、`Thread.sleep`、获取 `ReentrantLock` 失败）时，JVM 会自动触发 **`Continuation.yield()`**，将虚拟线程当前调用栈帧**保存拷贝到堆内存**中并挂起，**底层的平台载体线程瞬间被释放去调度执行其他就绪虚拟线程**！待 IO 就绪后再重新挂载恢复栈帧继续运行。
  - **使用方式**：`Thread.ofVirtual().start(task)` 或 `Executors.newVirtualThreadPerTaskExecutor()`（注意：**虚拟线程绝对不需要池化**，用完即抛）。
- **面试官连环追问**：
  - **追问 1**：JDK 21 使用虚拟线程最大的性能坑——**Pinning（钉住）**是什么？如何解决？
    - **回答**：在 JDK 21 ~ 23 中，如果虚拟线程在 **`synchronized` 同步代码块内部** 或 **调用 `native` 本地方法** 时发生阻塞操作，它会被**钉死（Pin）**在当前载体线程上**无法卸载（Unmount）**，导致大量载体线程被霸占，系统并发度瞬间退化甚至死锁！**解决方案**：将高频阻塞路径上的 `synchronized` 全部替换为 **`ReentrantLock`**（注：JDK 24 的 JEP 491 已彻底解决 `synchronized` 导致的 Pinning 问题）。
  - **追问 2**：虚拟线程适合 CPU 密集型计算吗？还能随便用 `ThreadLocal` 吗？
    - **回答**：① **不适合** CPU 密集型任务（物理 CPU 核数固定，不会变快反而增加调度切换开销）；② 百万级虚拟线程如果每个都用 `ThreadLocal` 缓存大对象会导致内存爆炸，JDK 21 推荐使用不可变、自动销毁的 **`ScopedValue`** 替代。
- **JDK 21 其他核心新特性**：
  - **分代 ZGC（Generational ZGC）**：通过 `-XX:+UseZGC -XX:+ZGenerational` 启用，按对象年龄分代回收，降低内存和 CPU 开销的同时保持亚毫秒级 STW 停顿。
  - **Sequenced Collections（序列化集合）**：为 `List`、`Deque`、`LinkedHashSet`、`LinkedHashMap` 等有序集合提供统一接口 `SequencedCollection`/`SequencedMap`，新增 **`getFirst()`、`getLast()`、`addFirst()`、`reversed()`**。
  - **`switch` 模式匹配正式转正**：支持类型模式、空值 `case null` 处理与 `when` 守卫条件。
