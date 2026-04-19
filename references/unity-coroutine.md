# Unity Coroutine — 协程的正确用法和坑

Moli 在项目里**极少用协程**（原话："其实我项目里很少很少很少用协程，原因是性能不好"）。但作为 Unity 生态的常用工具，遇到协程 bug 时他有一套快速定位的流程。

---

## 协程不执行时的排查清单

**为什么**：协程是由 `StartCoroutine` 调用者——那个 MonoBehaviour 实例——驱动的。不是由方法体所在的类驱动。驱动者一旦挂了，协程就停。方法体放在哪无所谓，它只是一个迭代器。

**按顺序排查**：

```csharp
// 1. 确认 StartCoroutine 的那个 mono 的 Update 还在跑
void Update() {
  Debug.Log($"[{GetInstanceID()}] Update frame={Time.frameCount}");
}

// 2. 给这个 mono 的 OnDisable / OnDestroy 加日志
void OnDisable() { Debug.Log($"[{GetInstanceID()}] OnDisable"); }
void OnDestroy() { Debug.Log($"[{GetInstanceID()}] OnDestroy"); }

// 3. 打印 Time.timeScale（timeScale == 0 协程不前进）
Debug.Log($"timeScale={Time.timeScale}");

// 4. 打印 GetInstanceID —— 确认不是被重新实例化了
// （常见踩坑：StartCoroutine 后对象立刻重建，协程挂在了旧对象上）
```

### 四个原因占了 99%

按出现频率排序：
1. **mono 被 SetActive(false)** —— 最常见，特别是 pooling 场景
2. **mono 被 Destroy** —— 切场景、对象池销毁
3. **外部 StopCoroutine / StopAllCoroutines** —— 被无意调了
4. **Time.timeScale == 0** —— 通常发生在"暂停"逻辑没正确恢复

Moli 给群友 debug 的原话：
> "协程是由 StartCoroutine 这个 mono 驱动的，所以正不正常，主要看驱动的那个 mono，至于你的方法体在哪里，只要执行里没有什么空引用，都无关紧要。"

---

## 不要在 Awake 里 StartCoroutine

**为什么**：Awake 是实例化后的第一个回调，这时 GameObject 可能还没完全激活、父子关系可能还没处理完。在某些创建时序下（新生成对象 Awake 后当帧又被 SetActive(false)），协程立刻被停——是非常难定位的诡异 bug。

**示例**：
```csharp
// bad
void Awake() {
  StartCoroutine(MyRoutine());  // 某些时序下会立刻被停
}

// good
void Start() {
  StartCoroutine(MyRoutine());  // Start 晚于 Awake，此时对象已完全初始化
}
```

Moli 分析："Awake 时整个 mono 对象实际还没有启动，当帧的 update 有没有执行，取决于你是在哪个阶段 new 的这个 mono。"

这个坑通常在"对象池创建时立刻 SetActive(false) 隐藏起来，用的时候再 SetActive(true)"这种模式下暴露——Awake 和 SetActive(false) 都在当帧发生，协程刚启动就被停。

---

## 少用协程，性能敏感路径用事件/计时器/async-await 替代

**为什么**：协程每次 yield 都走迭代器 + 可能的闭包分配；数量一上来 GC 压力可观。对大量实体（每个敌人自己的延迟攻击、冷却、轮询）用协程会分配大量迭代器对象。

**替代方案**：

```csharp
// bad: 每个 enemy 自己启一个协程
foreach (var e in enemies) {
  e.StartCoroutine(e.DelayedAttack());
}

// good A: 集中驱动
class CombatTickSystem : MonoBehaviour {
  List<(Enemy e, float dueTime)> queue;
  void Update() {
    var t = Time.time;
    for (int i = queue.Count - 1; i >= 0; i--) {
      if (queue[i].dueTime <= t) {
        queue[i].e.Attack();
        queue.RemoveAt(i);
      }
    }
  }
}

// good B: UniTask（Unity 生态的高性能 async）
await UniTask.Delay(TimeSpan.FromSeconds(1f), cancellationToken: ct);
enemy.Attack();
```

Moli 原话："我项目里很少很少很少用协程。原因是性能不好。"

---

## 协程什么时候还是合适

不是说完全不用——一次性的、数量少的、不跑在紧循环里的场景协程仍然好用：
- 场景加载的等待序列（比如加载资源 → 等一帧 → 初始化）
- 单次的序列动画（UI 入场 tween）
- 不频繁触发的延时回调（每秒至多几次）

**不合适**：
- 每个 enemy/bullet/particle 一个协程
- 热路径上的延迟/轮询
- 大量同时存在的等待状态
