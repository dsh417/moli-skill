# Performance — Moli 的性能观

核心原则：**写代码时就带性能意识，而不是 profile 完再回头改**。Moli 的逻辑——同样的代码，有/没性能意识花的编码时间是一样的，但后期到大型项目里定位热点、改设计的代价高得多。

---

## 反射调用方法：启动期一次性构建映射表缓存

**为什么**：运行时反射开销高（查找 + 权限检查）；服务器/客户端启动期扫描并缓存 MethodInfo 代价可忽略，因为"启动时间"对服务器来说无所谓。

**示例**：
```csharp
// bad: 每次调用 type.GetMethod(name)
// good: 启动期扫描所有 [MyAttribute] 方法，填充 Dictionary<string, MethodInfo>
//       运行时直接 dict[name].Invoke(...)
```

---

## 需要传函数：用语言原生委托/函数指针，别用反射+字符串查找

**为什么**：C# 的 `delegate`/`event`/`Action`、Java 的 Functional Interface、C++ 的函数指针都是**零运行时查找成本**；反射每次调用都带查找+权限检查开销。

**示例**：
```csharp
// bad
var m = type.GetMethod("OnHit");
m.Invoke(obj, args);

// good
public Action<DamageInfo> OnHit;  // 持有委托，直接调
```

---

## 日常写代码就带性能意识

**为什么**：大型项目里任何代码都可能成为热点；profile 出来后反向追溯+改设计的代价远高于一开始就写好。而**同等写法下有/没性能意识花的编码时间是一样的**。

**示例**：
```
写循环 —— 顺手想一下：有没有不必要的分配？
传容器 —— 顺手想一下：要不要传引用？
选数据结构 —— 顺手想一下：命中率够不够？
```
这些都不花额外时间，但省了后期翻架子。

Moli 说："代码写得奔放的，设计也奔放，到时就不是'有性能压力'的地方优化的问题了，是放架子，开发效率就是慢。"

---

## unsafe 不是性能银弹

**为什么**："关闭下标检查"在紧循环里确实能快，但大多数场合 unsafe 带来的收益远小于它带来的调试和崩溃风险；现代 C# 有 Span/stackalloc/ArrayPool 能覆盖 99% 的实际场景。

**示例**：
```csharp
// bad: 所有性能敏感函数都开 unsafe，写指针算术
// good: 优先用 Span/Memory/ArrayPool，只在基础层封装里才用 unsafe
```

Moli 的观察："现在出了安全的高效内存操作不用，非得在 unsafe show 显的自己很专业……"

---

## ECS 数据拆分以"业务是否一起使用"为准，不是越原子越好

**为什么**：xy 坐标几乎永远一起用，拆成两个 component 反而破坏 cache 局部性、是负优化；ECS 原子化的目的是"业务组合灵活 + cache 命中率高"，两者都在"高频一起用"时合并。

**示例**：
```csharp
// bad: class Position_X : IComponent { float x; } + class Position_Y : IComponent { float y; }
// good: struct Position { float x, y, z; }
```

Moli 原话："ecs 的核心是对数据有意义的原子化，xy 基本是一起工作的，这个拆就没有必要了。"

---

## 服务器主循环不要用线程 sleep，用事件驱动

**为什么**：sleep 直接降低并发能力；死循环不 sleep 又会吃满 CPU 单核；事件驱动（socket 中断 / epoll / poll）才是服务器性能的基础——等 IO 就让出 CPU。

**示例**：
```csharp
// bad
while (true) { ProcessAll(); Thread.Sleep(1); }

// good
while (true) { WaitForSocketReady(); ProcessReady(); }
```

Moli 原话："我网络这边是直接用 socket 这边的中断的，能不用线程的 sleep 就不用。"

---

## C# 中遍历集合默认用 for (int i=...)，不要无脑 foreach

**为什么**：
- array 的 foreach 被编译器特化，接近 for 性能
- 但 List<T> / 自定义 IEnumerable 的 foreach 每次迭代**多一层函数调用** + 可能的**堆分配**
- 写起来成本一样——既然无差，就写性能好的那个
- Unity IL2CPP 下 foreach 历史上 GC 严重

**示例**：
```csharp
// bad (尤其对 List / 自定义集合)
foreach (var x in list) { Use(x); }

// good
for (int i = 0; i < list.Count; i++) { Use(list[i]); }
```

Moli 实测数据：
- IL2CPP（三星 S8）下 **list 的 for 2ms / foreach 6ms / 退化成 IEnumerable 的 foreach 18ms**（百万次循环）
- array 在 PC 上 for 和 foreach 无差（编译器特化）

反驳"优化点太小"的常见说法：
> 业务大了，每个热点业务来个 5000 次循环，一帧也就 200 次（如果这个热点业务是一个通用 API，不注意的话，一帧可能不止一次调用）——就能顶到这个值。30fps 下一帧 33ms，有多少个 4ms/16ms 可以浪费？

在成本没有区别的情况下，写性能最高的那个。

---

## 总结一句

"我说的'为什么不用 XX'都是有对比成本的。在成本没区别的情况下，我就是用性能最高的那个。"
