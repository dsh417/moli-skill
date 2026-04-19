# C# Specifics — C# 专项规则

Moli 写了十几年 C#，对 C# 的细节有具体看法。这里是他最常纠正别人的几个点。

---

## unsafe 代码隔离到专门程序集，业务程序集禁止 unsafe

**为什么**：
- 业务到处开 unsafe 等于整个项目都不 Safe
- 真出崩溃根本无法定位是哪个 unsafe 写坏了
- 把危险操作框在极小范围、用安全 API 暴露给业务，是 unsafe 的**原本用法**

**示例**（工程结构）：
```
MyGame.Core.Unsafe.dll      (allow unsafe)
  └─ 只放类型转换/byte copy/指针算术这类基础操作
     对外暴露安全的 API，比如 SafeSpan<T> TryCast<T>(...)

MyGame.Business.dll         (no unsafe)
  └─ 业务层只调用 Core.Unsafe 里封装好的安全 API
```

Moli 原话："unsafe 本来就不是业务上用，你搞一个程序集用这个，定义出 API，你在业务里就是用正常的、可读性很强的 API 了。他的本意就是把危险操作框在一个非常非常小的范围，其它地方全是安全代码。"

配合规则："unsafe 不是性能银弹"（见 performance.md）—— 真要性能优化也优先用 `Span<T>`/`stackalloc`/`ArrayPool`。

---

## 数组切片用 Span<T>/ReadOnlySpan<T>

**为什么**：Span 是对"原数组引用 + 起始位置 + 长度"的零拷贝抽象；没有它的老 runtime，BitConverter 这些 API 本来就支持 offset 参数，只是可读性差，自己包一层就行。

**示例**：
```csharp
// modern
ReadOnlySpan<byte> slice = buf.AsSpan(offset, length);
int value = BinaryPrimitives.ReadInt32LittleEndian(slice);

// legacy workaround (老 Unity / Mono)
struct Slice<T> {
  public T[] buf;
  public int offset;
  public int length;
  public T this[int i] => buf[offset + i];
}
```

Moli 原话："没有切片这个 API 时，你自己写一个切片就行了，无非就是把对数组进行操作的 API 再封装一下，好用一点。"

关于 `Span` 和 `stackalloc` 的区别：
> "Span 跟 stackalloc 不冲突。stackalloc 只是不会在堆上申请内存而已，但是你用在这个场合还是要 copy，而 Span 可以不用 copy。"

---

## 类字段/属性用显式类型，函数局部变量才用 var

**为什么**：
- 字段/属性的**阅读频率**远高于写入；显式类型是给未来读代码的人的注释
- 函数局部变量范围小，`var` 简化表达没代价

**示例**：
```csharp
public class Foo {
  Dictionary<string, int> _cache;        // 字段显式 ✓
  public List<IItemData> Items { get; }  // 属性显式 ✓

  public void Bar() {
    var tmp = _cache.ToList();           // 局部 var ✓
    for (int i = 0; i < tmp.Count; i++) { // 循环变量也是局部 ✓
      ...
    }
  }
}
```

Moli 原话："微软设计很聪明：类字段、属性在全局范围内用，明确定义可以减少维护成本；而函数里面的变量只在局部使用，所以允许类型推断简化表达。"

关于类成员用 var：
> "类成员都能推断类型，那就成弱类型了。"

---

## 想让类的某些方法"只对接口可见" → 用显式接口实现

**为什么**：显式接口实现的方法必须通过接口引用调用，持有具体类引用的业务层看不到这些方法；适合迭代器、序列化钩子、内部协议这类不该被业务层误调用的方法。

**示例**：
```csharp
public class MyCollection : IEnumerable<int> {
  List<int> _items = new();

  // 显式接口实现：只能通过 IEnumerable<int> 引用调用
  IEnumerator<int> IEnumerable<int>.GetEnumerator() {
    return _items.GetEnumerator();
  }
  IEnumerator IEnumerable.GetEnumerator() {
    return _items.GetEnumerator();
  }

  // 业务 API
  public void Add(int x) => _items.Add(x);
}

// 用法
var c = new MyCollection();
c.Add(1);
foreach (var x in c) { ... }  // OK，foreach 通过 IEnumerable 调
// c.GetEnumerator() ← 编译不过，具体类上看不到
```

Moli 原话："我其实也在很多设计里，用这招隐藏我不想让业务层看到的方法。"

---

## C# 里遍历用 for 不用 foreach（大规模场景）

这条详见 [performance.md](performance.md)，关键：
- array 的 foreach 被特化，接近 for
- List<T> / 自定义 IEnumerable 的 foreach 多一层函数调用 + 可能堆分配
- IL2CPP 下数量级差别
- 写起来成本一样——既然无差，就写性能好的那个

---

## 其他 Moli 观察到的点

- **C# 的 `IDisposable` 用 `using` 语句** —— 避免忘记释放；Unity IL2CPP 下 GC 更重，more important
- **避免热路径上的 `new`** —— 任何能池化的对象都池化（ArrayPool<T>, ObjectPool<T>）
- **不要为没预期的业务提前做"预留字段"** —— 后期按需加，比预留后被用歪成本低
