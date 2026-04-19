# EVAL — Moli Skill 触发测试

## 前言：测试 prompt 的适配

用户原始 spec 里的 5 条测试 prompt 混合了 UE 和 Python。但项目中途做了 scope 调整——Python 从"对等领域"降为"低优先级"，实际蒸馏源只覆盖 UE / Unity / Godot 游戏开发。因此原 prompt 中 2 条 Python 题目不适用，2 条 UE 题目测试的是我们 skill 里较弱的领域（UE 专项规则少，通用性能/架构原则多）。

**实际采用的 5 条 prompt**（保留原版 spec 的场景意图，但调整为 skill 的实际覆盖领域）：

1. **UE C++ 技能冷却组件**（原题）—— 测 UE 方向 + 架构/性能原则
2. **Unity 大量敌人的血条 UI** —— 测 UI 专项（核心强项）
3. **Godot 联机同步** —— 测多平台选型 + 联机
4. **UE 多线程访问 UObject**（原题）—— 测 Moli 弱项（诚实暴露）
5. **Unity 配置表设计** —— 测数据存储 + 架构专项

每条评估：
- `baseline`：无 skill 时 Opus 4.7 的典型回答轮廓
- `skill-loaded`：加载 Moli skill 后会额外引用哪些规则
- `delta`：差异评估

---

## Test 1: "帮我写一个管理技能冷却的 UActorComponent"

### Baseline（无 skill）
通用做法：
- `UCLASS(ClassGroup=Skills, meta=(BlueprintSpawnableComponent))`
- 一个 `TMap<FName, float>` 存冷却时间
- 一个 Tick 或 Timer 在冷却结束时触发事件
- 暴露 `BlueprintCallable` 方法给蓝图

### Skill-loaded
Moli skill 会额外引用：
- **r_002 (reflection caching)** —— 如果技能配置从 DataTable/DataAsset 查，启动期预解析成 map，不要运行时反射
- **r_013 (refactor timing)** —— 如果是早期阶段，先做能跑的版本；别一上来就抽通用框架
- **r_033 (event callbacks)** —— 冷却结束的回调用委托/DECLARE_DYNAMIC_MULTICAST_DELEGATE，不要现场 lambda
- **r_021 (perf mindset)** —— Tick 一定加早返回（`if (CooldownsEmpty) return;`），详见 performance.md

### Delta
中等受益。UE 专项规则少，但架构/性能原则依然适用。Skill 加载后的回答会更注意"性能意识+早返回"，减少盲目的 TMap 扫描。

### 漏报风险
Moli 没讨论过 UE 具体的 Tick 优化（TickInterval, AddTickPrerequisiteActor 等），skill 不会自发建议这些。属于 UE 专项盲区。

---

## Test 2: "Unity 里怎么给场上 100 个敌人每个头上显示血条？"

### Baseline
典型建议：
- 每个敌人挂一个 World Space Canvas + Image（UGUI）
- 在 LateUpdate 里根据敌人位置把 Canvas 跟随
- 性能不好时改用 Object Pool

### Skill-loaded
**核心强项区**。Moli skill 必触发：
- **r_014 (大量简单 UI → GPU Instance)** —— 直接建议用 custom mesh + instance，一个 drawcall 搞定；血条的长度/颜色走 instance data
- **r_039 (UGUI 是 2D 渲染方案)** —— 提醒"别把血条堆给 UGUI，UGUI 不擅长这个"
- **r_017 (Profiler UI 大 → 业务写在 UI 循环)** —— 如果用户说"我试过 UGUI 性能不好"，定位方向是"血条更新业务是不是塞在了 UI Update"
- **r_031 (DrawMeshInstancedIndirect)** —— 实现建议

### Delta
**高受益**。Baseline 会给"正确但性能一般"的答案；skill-loaded 会给"性能数量级更好"的方案，并解释为什么。这是 skill 最能拉开差距的场景之一。

---

## Test 3: "用 Godot 做一个 2-4 人联机派对游戏，怎么同步？"

### Baseline
- 介绍 Godot 的 MultiplayerAPI / high-level networking
- 建议用 RPC 同步状态
- 提示带宽考虑

### Skill-loaded
- **r_018 (Steam P2P 优于自建 server)** —— 即使是 Godot，如果能接 Steamworks，优先 Steam P2P 穿透
- **r_044 (多平台首选 Unity)** —— 会提醒"Godot 在商业规模联机上稳定性还没到，Moli 的看法是 Godot 不在商用牌桌上" —— 用户如果在意跨平台/商业发布要考虑
- **r_040 (物理联机不做混合方案)** —— 如果游戏有物理（派对碰撞/推挤）直接建议中心服务器或帧同步+定点数
- **r_019 (工作量估计 ≥2倍)** —— 提醒"别以为联机只加 20% 工作量"

### Delta
中高受益。skill 不会替 Godot 特有 API 做推荐，但会把"架构级选型"和"风险提示"灌进去，避免用户走混合物理同步这种坑。

---

## Test 4: "UE5 里怎么在多线程中安全访问 UObject"

### Baseline
标准答案：
- UObject 不是线程安全的，主线程以外禁止访问
- 用 `AsyncTask(ENamedThreads::GameThread, [...]{...})` 回到主线程
- 或者用 `TWeakObjectPtr` + 主线程检查

### Skill-loaded
**Skill 的薄弱区**。诚实说——Moli 的蒸馏里没有 UE 多线程专项规则。能挂上边的只有：
- **r_036 (library abstraction level)** —— 如果在做跨引擎抽象，警告不要把"UObject 主线程"假设带进去
- **r_028 (server event-driven)** —— 如果是服务器侧的 UE（dedicated server），避免 sleep 等主线程空转

但对"UObject 多线程"这个具体问题，skill 没有 Moli 的直接经验。

### Delta
**几乎无增益**。诚实标记：**这是 skill 的盲区**。用户问这类专项问题时，skill 应该坦率说"Moli 没覆盖这块，建议查 UE 官方文档的 FAsyncTask / FRunnable / TaskGraph"。

改进建议：SKILL.md frontmatter 的 description 应该提示"UE 多线程细节/Nanite/Lumen 不在覆盖范围内"，避免假信心。

---

## Test 5: "Unity 项目里配置表怎么设计？有几千种道具+几百个技能"

### Baseline
- 用 ScriptableObject 一个一个建
- 或者 CSV/JSON 导入
- 运行时字典查找

### Skill-loaded
- **r_032 (单机不用数据库)** —— 明确建议字典+序列化文件，不要用 SQLite 杀鸡用牛刀
- **r_004 (统一容器)** —— 道具/技能都是 "ConfigEntry" 的特例，用统一的 `IConfigEntry` 基类+ID 索引
- **r_002 (启动期构建映射)** —— 启动时把 CSV/JSON 全量加载到 `Dictionary<int, ItemData>`，不要每次查表都 IO
- **Moli 的配置表终极优化**（见 data-storage.md）：
  - 1 万条以下 → 哈希预处理做无碰撞
  - 1 万条以上 → B+ 树分层加载
  - 索引与数据分离、字符串常量块去重

### Delta
**高受益**。Baseline 会给通用答案；skill 会给 Moli 在"做过大型配置表"经验里的具体技术栈，用户需要的时候能一步到位。即使用户只有几千条现在不需要终极优化，提前知道演进路径有价值。

---

## 整体评估

| # | 场景 | Skill 受益度 | 有效触发的规则 | 有风险的漏报 |
|---|---|---|---|---|
| 1 | UE 技能冷却 | 中 | r_002, r_013, r_021, r_033 | UE Tick 优化专项 |
| 2 | Unity 血条 UI | 高 | r_014, r_017, r_031, r_039 | 无（强项） |
| 3 | Godot 联机 | 中高 | r_018, r_019, r_040, r_044 | Godot 特有 RPC API |
| 4 | UE 多线程 UObject | **低（盲区）** | r_028, r_036 (勉强) | UE 专项几乎全漏 |
| 5 | Unity 配置表 | 高 | r_002, r_004, r_032 + Moli 配置优化框架 | 无 |

### 结论

1. **强项覆盖好**：Unity UI 性能、配置系统、寻路移动、C# 专项、联机选型——skill 能显著提升回答质量
2. **UE 弱**：UE 特有的 Tick 优化、多线程、Blueprint 性能等没蒸馏到。SKILL.md 的 description 覆盖了 UE 关键词会触发，但实际只有通用原则能用
3. **Python 不应触发**：skill 的 description 不包含 Python 关键词，这是对的——Moli 没有 Python 经验

### 改进建议（未来迭代）

- `SKILL.md` frontmatter 应该加一句"覆盖范围：Unity 深度、Godot 选型、C# 专项、UE 原理与通用但缺 UE 实现细节"
- 如果未来有 UE 专业程序员的蒸馏源，可以作为 sibling skill (`moli-ue-skill`) 补充
- 没写到的领域：UI 动画曲线、Shader 实战、引擎源码魔改（Moli 偶尔提到但没形成可蒸馏的规则）

### 本 EVAL 的局限

诚实说明：
- 本测试没有实际跑两遍 LLM 对比——由同一个 Opus 4.7 模型（即构建本 skill 的同一模型）预测轮廓
- 真实 baseline vs skill-loaded 对比应该由**用户在自己的 Claude Code 环境里跑**—推荐测试方法：关闭 skill 问一遍，开 skill 问一遍，对比差异
- Skill 实际效果还取决于用户的 Claude Code 是否正确触发（description 关键词匹配 + 上下文）
