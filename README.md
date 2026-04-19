# moli-skill

一个 **Claude Code skill**，内容是一位资深游戏引擎/客户端程序员（化名 **Moli**）的 20 年技术沉淀——Unity / UE5 / Godot 游戏开发里的性能、架构、渲染、UI、联机、寻路、C# 专项细节等。

## 这不是什么

- 不是官方文档的压缩版
- 不是"最佳实践"合集
- 不是"AI 写的通用游戏建议"

## 这是什么

- 一个**有风格的人**的具体看法：会直接说"scale=0 隐藏是最烂的做法"，会说"unsafe 不是性能银弹"
- 每条规则都有**为什么**（踩过的坑、原理、权衡）和**最小代码示例**
- 分类组织，按你当前在做的事情找对应 `references/<category>.md`

## 安装（作为 Claude Code skill）

把本仓库克隆或下载，放到你的 Claude Code skills 目录：

```
~/.claude/skills/moli-skill/
```

或在项目级 `.claude/skills/moli-skill/`。

Claude Code 会自动读取 `SKILL.md` 作为入口。

## 结构

```
moli-skill/
├── SKILL.md                        # 入口（含触发关键词和分类 checklist）
├── README.md                       # 这个文件
└── references/
    ├── performance.md              # 性能意识、foreach、反射、ECS、unsafe
    ├── architecture.md             # 架构、重构时机、跨平台抽象
    ├── pathfinding-movement.md     # A*、编队寻路、移动手感
    ├── ui.md                       # FGUI vs UGUI、drawcall 合批、性能
    ├── multiplayer.md              # Steam P2P、物理同步、工作量估算
    ├── csharp-specifics.md         # Span、var、显式接口、unsafe 隔离
    ├── rendering.md                # 合批、Instance、贴图、隐藏技巧
    ├── data-storage.md             # 单机不用数据库、配置优化
    └── choice-and-scope.md         # 引擎选型、AI 能力边界、Godot 反编译
```

## 使用方式

Claude Code 在你写游戏相关代码时会**自动触发**本 skill（SKILL.md 的 `description` 字段里有一长串关键词）。

如果你想手动参考某一类，比如正在做寻路：

```
> 参考 Moli 的 pathfinding 经验，帮我实现编队移动
```

Claude Code 会去读 `references/pathfinding-movement.md`。

## 关于"Moli"

- 这是一位具体的人的技术沉淀，不是泛化的集合
- 他的风格：**直接、不客气、成本意识、实证为本**
- 他的时代视角：做过 Flash/AIR/stage3d、PC 网游、手游、独游
- 他不代表所有游戏程序员，但他的视角非常具体、可追溯

所有涉及个人身份的信息（昵称、QQ、具体时间戳、公司名）都已去除。保留的只是**观点和做法**。

## 反馈 / 贡献

发现某条规则和你的实战经验不符？开个 issue 讨论。

Moli 不是不会错——他会直接承认"dotnet 的 foreach 和 unity 的 foreach 表现不一样，Unity 那个是垃圾"。有反驳欢迎带数据。

## 许可

[未定，按你仓库的许可]

---

**注**: 本 skill 从群聊记录蒸馏而来，经过人工精炼。源话题 id 在每条 rule 的 `source_bursts` 字段（见 `distilled/raw_rules.jsonl`，不在 git 版本里）。
