# Rendering — 渲染与资源

---

## 不要用 scale=0 隐藏物体

**为什么**：
- 物体仍在内存里，**渲染依然会算一次开销**
- 带物理的直接会报错
- 在大系统里某个 scale=0 导致的计算错误（除零、归一化异常）**极难定位**

**示例**：
```csharp
// bad
transform.localScale = Vector3.zero;
renderer.enabled = false;  // 也不够，物理/AI/各种组件仍在跑

// good
gameObject.SetActive(false);          // Unity: 完整停用
// 或者用专门的"可见/启用"标志，控制相应子系统
```

Moli 原话（直接开骂）："这几乎是最烂的办法。他最大的问题是容易出错。在大系统里，你可能还很难找到错误的原因就是因某个 scale=0……各种因为 scale=0 而导致的计算错误。带物理的你缩 0 直接就报错了。"

---

## 六角格/方格地图：美术表现层和逻辑网格层必须解耦

**为什么**：把资源按格子形状做，工作量巨大、资源体积大，而且格子接缝很难自然过渡；文明/策略游戏的做法是整张大地形图贴图，网格只是逻辑层 overlay。

**示例**：
```
bad: 每种地形为六角形切一份 png，凑起来接缝突兀
good: 渲染一张完整地形大图 / 程序化生成
     网格坐标只用于逻辑判断（谁在哪个格子、寻路距离、AOE 范围）
```

Moli 原话："人家就是渲染正常的大地形，格子只是后画上去的，资源不是按六角形格子做的。……你不要把表现跟逻辑格绑到一块去，这两个是可以分开的。"

---

## 像素/2D 游戏做阵营区分在材质里换色，不要用描边

**为什么**：等距离描边需要多绘制一次整张贴图（或多次采样），性能代价明显；材质里换色是 0 成本。

**示例**：
```
bad: 给每个单位生成带描边的贴图变体 / 额外描边 pass
good: 材质里 _FactionColor 参数，统一贴图，运行时切换
     shader 里做一个简单的 color multiply / tint
```

Moli 原话："像素描边要用等距离描边，要多绘制一次，性能不好。还不如材质上分阵营颜色呢，性能更好。"

---

## Unity 渲染大量重复物体用 DrawMeshInstancedIndirect / BatchRendererGroup

**为什么**：普通 Renderer 每个物体一个 drawcall；instance indirect / BRG 一次 draw call 批量绘制，几千几万物体没问题。

**示例**：
```csharp
// 小批量 (<1023 instances)
Graphics.DrawMeshInstanced(mesh, 0, mat, matrices);

// 大批量 / 高性能
Graphics.DrawMeshInstancedIndirect(mesh, 0, mat, bounds, indirectArgsBuffer);

// 新路径（Unity 2022+ HDRP/URP 推荐）
var brg = new BatchRendererGroup(OnPerformCulling);
brg.AddBatch(...);
```

**什么时候用哪个**：
- 固定数量、提前知道实例数：DrawMeshInstanced
- 数量动态 / GPU 决定剔除 / 百万级：DrawMeshInstancedIndirect
- 新项目、要对接 SRP：BatchRendererGroup

Moli 提醒：
> "渲染大量对象可以看 DrawMeshInstancedIndirect，还可以看看 BatchRendererGroup，这个也是用来处理大量显示对象的。"

配合 GPU 驱动做 UI 飘字/血条/粒子替代：见 [ui.md 的"大量 UI 元素用 GPU Instance"规则](ui.md)。

---

## Moli 对 Unity 合批机制的一些看法

（来自 FGUI 讨论中的观察）

- Unity 的合批依赖引擎内部机制识别可合批的材质 —— FGUI 的 Unity 实现里有大量排序只是为了让 Unity 能识别出"这些可以合批"
- URP 出来后，老的合批策略其实不是最优解了
- Unity 的 UBO 容量太小（1023），用 TBO 可以一次绘制几百万个
- 自写 2D 引擎的合批能力可以比 Unity 强很多倍 —— 但那是有工程师能维护引擎代码的公司的事

对独游开发者：按主流路径走（DrawMeshInstanced 家族），不用尝试魔改 Unity 源码。
