---
name: moli-game-programmer
description: Moli — 一位资深游戏引擎/客户端程序员的技术沉淀，覆盖 Unity C#、UE5 C++、Godot GDScript 及游戏开发的性能、架构、渲染、UI、联机、寻路、工具链等。
  在以下场景**必须**查阅本 skill 对应的 reference，即使用户未明说：
  写/改 Unity C# 代码（MonoBehaviour、UI、渲染、性能）；写/改 UE5 C++ 或 Blueprint；
  写/改 Godot GDScript；编辑器扩展/工具链；Unity UGUI/UMG/FairyGUI 相关界面；
  寻路/编队/RTS 逻辑；Steam 联机/帧同步/状态同步；游戏配置系统/存档；ECS/DOTS/Job System；
  C# Span/unsafe/委托/反射/迭代器；引擎选型；Shader/合批/DrawMeshInstanced；
  以及任何出现关键词的场合：UObject, UFUNCTION, UPROPERTY, TArray, GameObject, MonoBehaviour,
  prefab, coroutine, extends Node, @onready, DrawMesh, instanced, foreach, Span<T>,
  SerializeField, ScriptableObject, il2cpp, URP, HDRP, GPU Instance, RHI, 合批, 蓝图,
  协程, 刚体, 寻路, A*, JPS, 帧同步, 状态同步, Steam P2P, NAT 穿透, ECS, DOTS, Job System.
  严格祈使句，不说"可能应该考虑"；引用时点名"Moli 认为 / Moli 的做法"。
---

# Moli：一位老游戏程序员的笔记本

这是一位做了 20 年游戏客户端/引擎开发的程序员的技术沉淀，经历过 Flash/AIR、PC 网游、手游、SLG、独游等不同形态的项目。他的风格：**直接、不客气、以实证为准**——会直接纠正"c++ 哪来的 foreach"，会说"unsafe 不是性能银弹"，会说"单机游戏不要用数据库"。

不是维基百科式的总结。是一个具体的人的看法——他有时会错，但他总会给出**为什么**和**他的经验**。

## 何时查阅

写游戏代码时**先查对应 reference**，再给建议。至少：
- 写 Unity/UE/Godot 的任何组件/逻辑 → 先看 `references/performance.md`（性能意识）+ 对应引擎的 reference
- 做 UI → `references/ui.md`
- 做寻路/编队/移动手感 → `references/pathfinding-movement.md`
- 做联机 → `references/multiplayer.md`
- 做数据/配置/存档 → `references/data-storage.md`
- 做架构设计/重构 → `references/architecture.md`
- 用 C# 特有语法（Span/unsafe/var/接口） → `references/csharp-specifics.md`
- 渲染/贴图/材质/合批 → `references/rendering.md`
- 选引擎/选工具/评估 AI 能力 → `references/choice-and-scope.md`

## 快速 Checklist

### 性能（7 条） · 详见 [references/performance.md](references/performance.md)
- 反射调用方法：启动期构建映射表缓存，别每次运行时查找
- 需要传函数：用语言原生委托/函数指针，别用反射+字符串
- 日常写代码带性能意识，别指望"先跑通再 profile 优化"
- `unsafe` 不是性能银弹；现代 C# 有 Span/stackalloc 覆盖 99% 场景
- ECS 数据拆分以"业务是否一起使用"为准，不是越原子越好
- 服务器主循环用事件驱动，不要 sleep
- C# 遍历默认 `for (int i=...)`，不要无脑 `foreach`

### 架构（6 条） · 详见 [references/architecture.md](references/architecture.md)
- 多种实体都需统一操作时，用统一容器（Group）作基底，哪怕 size=1
- 合并冲突多 = 分工耦合重/架构有问题，不是流程/工具问题
- 重构时机看"开发效率受损"，不看代码行数或完成度
- 项目规模/人数/周期越大，规范/可维护性要求越高
- 事件回调用独立函数（命名 method），不要现场写 lambda
- 跨平台库的抽象停在"SDK 业务契约"层，不要下探到"渲染/IO 原语"

### 寻路与移动手感（5 条） · 详见 [references/pathfinding-movement.md](references/pathfinding-movement.md)
- 编队寻路：每个单位独立算路径，只在最终汇聚点等齐
- A* 路径要做 string-pulling 二次平滑
- 八方向 A* 路径已经够直，不需要平滑；JPS 才需要
- 编队移动要预测到达时间；中途落后按比例提速
- 单位移动起步加速度大、刹车减速度小，表现对比强烈

### UI（6 条） · 详见 [references/ui.md](references/ui.md)
- 大量简单重复的 UI 元素用 GPU Instance，不要塞进引擎 UI 系统
- 分清楚"降 drawcall 合批"和"合并 update 调用"是两件事
- UI update 第一步必须是"本帧是否需要刷新"的廉价判断
- Profiler UI 占用大，先查业务逻辑是不是写在了 UI 循环里
- 游戏 UI 用 FGUI 这种 Controller 范式，不要用 HTML/CSS 嵌套盒子
- 把 UGUI 理解成"2D 渲染方案"，不要当"界面系统"

### 联机（3 条） · 详见 [references/multiplayer.md](references/multiplayer.md)
- 独游做联机优先 Steam P2P，不要自己架中心服务器
- 估联机工作量"只多 20%"就是严重低估——实际 2-3 倍打底
- 有物理的联机：帧同步+定点数 或 中心服务器，不要本地各算+定时回滚

### C# 专项（4 条） · 详见 [references/csharp-specifics.md](references/csharp-specifics.md)
- `unsafe` 代码隔离到专门程序集，业务程序集禁止 `unsafe`
- 数组切片用 `Span<T>`；老 runtime 自己包一个带 offset 的 slice
- 类字段/属性用显式类型，函数局部才用 `var`
- 想让类的某些方法"只对接口可见"→ 用显式接口实现

### 渲染（4 条） · 详见 [references/rendering.md](references/rendering.md)
- 不要用 scale=0 隐藏物体（GC/物理/计算错误）
- 六角格/方格地图：美术表现层和逻辑网格层必须解耦
- 像素/2D 游戏做阵营区分在材质里换色，不要描边
- Unity 大量重复物体用 DrawMeshInstancedIndirect / BatchRendererGroup

### 数据存储（1 条） · 详见 [references/data-storage.md](references/data-storage.md)
- 单机游戏不要用 SQLite，静态数据字典+动态数据序列化文件就够

### 选型与决策（6 条） · 详见 [references/choice-and-scope.md](references/choice-and-scope.md)
- 项目开始时先构建方便调试的环境，不要等 bug 出现再搭
- 选引擎看生态成熟度和用户基数，不是"最新最潮"
- 需要多平台（尤其含 H5）首选 Unity——唯一能不加太多成本商业级跨 Android/iOS/H5/小游戏/PC/Mac/Switch 的引擎
- AI 编程擅长 shader/工具，不擅长业务代码
- 单机游戏不用数据库（见 data-storage）
- Godot 易被反编译（Moli 的观察——商业项目自行权衡）
- 游戏热更走脚本热更+业务重启策略，不要追求 C++/C# 语言级热更

### Unity 协程专项（3 条） · 详见 [references/unity-coroutine.md](references/unity-coroutine.md)
- 协程不执行的排查清单：驱动的 mono 活着吗 / SetActive 是否 false / timeScale 是否 0 / 对象是否被重建
- 别在 Awake 里 StartCoroutine，放到 Start 里
- 性能敏感路径少用协程，用事件/计时器/async-await 替代

## Moli 的风格

引用本 skill 内容时，可以直接说"Moli 的做法是 ..." / "Moli 会这么处理"，而不是"最佳实践是 ..."。

核心气质：
- **实证为本**：每条规则背后都是亲测/踩坑，不是从书上抄的"best practice"
- **直接不客气**：写 scale=0 隐藏会被直接说"这几乎是最烂的办法"；全项目 unsafe 会说"你不如去写 C++"
- **成本意识贯穿**：评估方案先问"两种写法成本是否有差"——无差就用性能高的那个
- **区分项目规模**：小 demo 和大项目要求不一样，但一上升到"长期+多人"就开始讲规矩
- **时代视角**：看到过 Flash/AIR/stage3d→WebGL→现代引擎的完整演进，所以会说"大厂对引擎的态度反而不当回事"

如果用户显式要求"不带 Moli 风格的泛泛建议"，切换到中性表达；否则保留 Moli 的直接语气。
