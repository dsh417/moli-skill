# UI — 游戏 UI 系统

Moli 从 Web UI 到 VS/宝兰/Ava 到 FGUI 全都用过，自己还设计过几种界面方案。核心结论：**游戏 UI 本质是状态机，不是文档流；别拿 HTML/CSS 那套嵌套盒子来套。**

---

## 大量简单重复的 UI 元素用 GPU Instance，不要塞进引擎 UI 系统

**为什么**：引擎 UI 系统每个元素有固定的 tick/layout/批次开销；简单元素（血条/tips/伤害飘字）做成一个 instance mesh + 一个 drawcall，性能数量级差别。

**示例**：
```
bad: 每个敌人头上 UGUI/UMG Widget 独立 tick
good: 自定义 mesh，每个血条一个 instance
     材质里用 instance data 驱动填充比例
```

Moli 原话："tips 表现很简单，可以做成 gpu inst 驱动，一个批次一个 mesh 就行，性能高，内存占用少。"

---

## 分清楚"降 drawcall 合批"和"合并 update 调用"是两件事

**为什么**：
- **合批**是为了降低 CPU 每次提交 drawcall 的固定开销
- **update 合并**改变不了总 CPU 成本（一千个循环 vs 一千次函数调用开销同量级）

这俩看起来都是"合起来快"，但**解决的瓶颈不一样**，搞错了优化方向。

**示例**：
```
UI 性能不好？先区分：
- drawcall 太多 → 考虑合批（同一材质/Atlas）
- update 里做了重逻辑 → 和合批无关，是业务写的问题
```

Moli 原话："你把 for 里面每个 item 当一个元素，就是几百个；你把 for 本身当一个元素，就是 1 个。本质上这个跟你有几百个界面没关系，只跟你的优化方面有关系。"

---

## UI Update 第一步必须是"本帧是否需要刷新"的廉价状态判断

**为什么**：元素不变时，update 开销应该接近"执行一个 if"级别（CPU 对 if 几乎零消耗）；盲目跑 layout/render 是浪费。

**示例**：
```csharp
void Update() {
  if (!IsDirty(lastState, currentState)) return;  // 第一行
  // 只有状态变了才走到下面的 layout/render
  DoLayout();
  Rebuild();
  lastState = currentState;
}
```

Moli 原话："界面如果没有发生变动，那应该只会有非常低的第一层级的状态判断消耗——类似于你执行了与界面元素个数相同的 IF。CPU 对 if 消耗微乎其微。"

---

## Profiler UI 占用大：先怀疑业务逻辑写在了 UI 循环里

**为什么**：UI 系统本身开销在写对的情况下占比很低；"UI 很慢"经常是业务/数据处理被塞到了 UI 的 update，被算到了 UI 的账上。

**示例**：
```csharp
// bad: BattleUI.Update() 里直接算伤害、算命中、广播事件
// good: 业务走独立系统，UI 只订阅事件后更新显示
```

Moli 原话："UI 很多时候在 profiler 上消耗占比比较大是因为，业务写在 UI 的循环里了，业务没写好，或量巨大，算到 UI 的消耗里去了。"

---

## 游戏 UI 用 FairyGUI 这种 Controller 范式，不要 HTML/CSS 嵌套盒子

**为什么**：游戏 UI 的关联本质是 N×N 的状态切换（选中/禁用/战斗态/菜单态...）；网格盒子嵌套去表达会一层套一层，性能差+自己维护不过来。FGUI 的 Controller 把多维状态展开为一维，美术能不改代码拼出完全不同的界面。

**示例**：
```
bad: Unity UGUI + Layout Group 嵌套做战斗 HUD
     切换模式要改 active/anchor/parent
good: FGUI Controller
     一个变量切换所有元素的显隐/位置/动画
     UI 动画和业务状态天然耦合
```

Moli 原话："FGUI 强的地方在于控制器、布局关联（去掉了布局那套，把 n×n 的关系，展开为一维的关系），然后由控制器这个概念带来的 UI 动画表现跟实际业务沟通——美术在不改变代码的情况下，可以拼出完全不同的界面。"

关于 Unity 新 UI：
> "Unity 最新的 UI 方案，就想把 HTML 的那套搬过来，我敢打赌，想出这个方案的人，实际没做过什么游戏业务。"

---

## 把 UGUI 理解成"2D 渲染方案"而不是"界面系统"

**为什么**：UGUI 名字叫 UI 但本质只解决了渲染/批处理；作为界面系统它缺少状态组合、动画驱动、控制器这些核心概念——别指望它承载复杂 UI 业务，上面另起一套（FGUI/自研）才合理。

**示例**（心智模型）：
```
UGUI = 2D Mesh 渲染器 + 输入路由
上层应用 = FGUI / NoesisGUI / 自研，把 UGUI 当底层用
```

Moli 原话："你把 UGUI 理解成一个 2D 的渲染方案，你就能跟他和解。请不要把他理解为一套界面系统。"

---

## 为什么"UI 网格布局"是陷阱

大部分 UI 系统（UGUI Layout Group、Godot container nodes、Web CSS flex/grid）走的都是**网格布局嵌套**。Moli 的批评：

> 大部分的 UI 系统，走的都是网格布局那套——这套有一个问题，一个复杂的关联，要一层套一层，性能差不说，自己也很容易搞晕掉。

解决方向不是"换更好的 Layout 算法"，而是**换抽象**：Controller 用一维状态描述多维组合。
