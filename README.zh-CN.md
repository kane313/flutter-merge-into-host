# flutter-merge-into-host

把另一个独立 Flutter 项目的 UI 代码作为自包含的 feature 子树合并到当前宿主 Flutter 项目里的 Claude Code skill。

[English](./README.md) | 中文

## 它做什么

给一个另外 Flutter 项目的路径（含自己的 `lib/`、`assets/`、`pubspec.yaml`），并把当前宿主 Flutter 项目作为 cwd，本 skill 会：

1. **扫描（Discover）**源结构、依赖、命名冲突、宿主约定
2. **决策（Decide）**落点、适配深度、依赖策略
3. **嫁接（Graft）**代码落到 `lib/features/<src-name>/<sub>/{pages,widgets,dialogs}/`，资源加命名空间，路由加 prefix，同名类用 `import as <alias>`，路由集成进宿主 `main.dart`
4. **适配（Adapt）**按宿主 CLAUDE.md 规则改造：`flutter_screenutil` 后缀、基础按钮包 `DebounceUtil`、`ToastUtil` 替换、宿主 theme、go_router 版本兼容
5. **验证（Verify）**`flutter analyze` 0 error + 资源 disk 存在性 grep + 路由注册检查 + 结构冲突预警

## 什么时候用

- 你给一个另外 Flutter 项目路径，想把它的 UI 合并到当前 cwd
- 或在 Flutter 项目根目录下说出迁移意图，让 skill 询问要迁移哪个项目

### 触发条件（任一满足即可）

**模式 A —— 路径 + 动词同句**（最明确）：
- 消息里有一个路径含 `lib/` + `pubspec.yaml`
- 且含动词关键词：`复制`、`迁移`、`合并`、`接入`、`嫁接`、`merge`、`migrate`、`graft`
- 例：`把 ../CuteAnimal-main 复制到本项目` / `merge ../foo`

**模式 B —— 独立意图短语，没给路径也触发**：
- `迁移`（单独出现，处于 Flutter 上下文）—— 如 `帮我迁移一下` / `这个项目要迁移`
- `flutter迁移` / `Flutter 迁移` / `flutter 迁移项目`（含空格也算）
- `flutter migration` / `flutter project migration`
- `合并 flutter 项目` / `把 flutter 项目接入`

模式 B 触发后第一步是 `AskUserQuestion`：「你想迁移哪个 Flutter 项目？请提供项目根目录路径（含 `pubspec.yaml`）」，**不要终止也不要乱猜路径**。

**前置检查**：宿主 cwd 必须是 Flutter 项目（含 `pubspec.yaml` + `flutter:` dep）。否则提示用户先 `cd` 到 Flutter 项目，不开始扫描。

如果用户在非 Flutter 上下文说"迁移"（如数据库迁移、服务器迁移），不应触发——上面的 cwd 检查会拦住。

## 什么时候不要用

- 单页 Figma 还原 → 用 [`flutter-figma-reproduce`](https://github.com/kane313/flutter-figma-skills)
- 构建可复用 package → 本 skill 产出 in-tree feature 代码，之后再手动抽到 `packages/`
- 源本身就是 library/package（没有 `lib/main.dart`、没有 `MaterialApp`）→ 简单 `cp` 就够了

## 5 阶段流水线

| Phase | 名字 | 强制 checkpoint | safe | auto |
|---|---|---|---|---|
| 0 | Discover（扫描） | 是 | 等待 | log + 推进 |
| 1 | Decide（决策） | 是（3 个 AskUserQuestion） | 等待 | 用默认 |
| 2 | Graft（嫁接） | 否 | 推进 | 推进 |
| 3 | Adapt（适配） | 否 | 推进 | 推进 |
| 4 | Verify（验证） | 是 | 等待 accept | 报告 + 推进 |

各阶段细节在 `skills/flutter-merge-into-host/references/0N-<phase>.md`。

## 模式

| 模式 | 行为 |
|---|---|
| `safe`（默认） | Phase 0、Phase 1（3 个 AskUserQuestion）、Phase 4 后强制 checkpoint 暂停 |
| `auto` | 端到端跑，用合理默认值；强制 pre-flight 检查仍执行 |

`auto` 触发词：`连续执行`、`自动执行`、`不用问`、`一口气`、`跑到底`、`autopilot`、`yolo`、`just do it`。

中途打断 auto（`等等` / `wait` / `暂停` / `让我看看`）→ 自动降级为 safe。

## Phase 3 改造规则（Rules）

| # | 规则 | 触发条件 |
|---|---|---|
| 1 | `flutter_screenutil` 全量后缀（`.w/.h/.sp/.r`） | 宿主 CLAUDE.md 提到 screenutil |
| 2 | 基础按钮包 `DebounceUtil` | 宿主 CLAUDE.md 提到 DebounceUtil |
| 3 | 源 `AppToast.show` → 宿主 `ToastUtil.show` | 源含 toast 实现 |
| 4 | 宿主 theme 集成 | 保守跳过（颜色 1:1 映射时才做） |
| 5 | go_router API 版本兼容 | 版本分歧时 |
| 6 | 分类+列表页 → PageView 联动 | Phase 0 检测到 candidate |
| 7 | 源 splash → 宿主中台 splash 模块（widget 注入或 logo 替换） | 宿主有中台 splash 模块且源有 splash |
| 8 | 文件拆分（> 300 LOC / > 5 private widget / > 10 const 任一） | 文件超阈值 |
| 9 | 基础按钮 label 防溢出（FittedBox.scaleDown 守卫） | 源有 base button 且页面用窄 flex 组合 |

## 关键防御点（auto 模式不可跳过）

即便 `auto` 移除了用户暂停，下面这些机械检查仍必须执行（漏一个 = 运行时 silent failure）：

1. **同名类 grep**（Phase 0/2）
2. **路由 literal grep 加 prefix**（Phase 2）—— 漏一个就会某个按钮点了 404
3. **资源 disk 存在性 grep**（Phase 2/4）—— 漏一个就运行时空白图
4. **Pubspec 子目录显式注册**（Phase 2）—— Flutter 不递归
5. **`flutter pub get` 成功**（Phase 2）
6. **`flutter analyze` 0 error**（Phase 2/3/4）

## 安装

本仓库是一个 Claude Code plugin marketplace，推荐用 `/plugin` 命令安装：

```
# 1. 把本仓库添加为 marketplace
/plugin marketplace add kane313/flutter-merge-into-host

# 2. 从中安装插件
/plugin install flutter-merge-into-host@flutter-merge-skills

# 3. 重载使 skill 注册生效
/reload-plugins
```

装好后，`flutter-merge-into-host` skill 会在 merge/迁移 等触发词出现时自动激活（见[什么时候用](#什么时候用)）。可用 `/plugin list` 确认已安装。

- Marketplace 名：`flutter-merge-skills`
- Plugin 名：`flutter-merge-into-host`

后续更新：`/plugin marketplace update flutter-merge-skills`，再重新跑一遍 `/plugin install flutter-merge-into-host@flutter-merge-skills`。

<details>
<summary>手动安装（不走 marketplace）</summary>

克隆后把 skill 放到 Claude Code 的 plugins 目录：

```
~/.claude/plugins/<plugin-name>/skills/flutter-merge-into-host/
```

</details>

## 目录结构

```
skills/flutter-merge-into-host/
├── SKILL.md                          # 入口 — 最先读
├── references/
│   ├── 00-discover.md                # Phase 0 扫描清单
│   ├── 01-decide.md                  # Phase 1 三个决策 + 默认值
│   ├── 02-graft.md                   # Phase 2 文件搬迁 + import 深度算法 + 路由 prefix
│   ├── 03-adapt.md                   # Phase 3 九条 Rule 完整说明
│   ├── 04-verify.md                  # Phase 4 验证 + runtime overflow 检测
│   ├── failure-modes.md              # 实战踩坑清单
│   └── fastlane.md                   # safe vs auto 模式差异
└── templates/
    ├── discovery.md.tmpl             # discovery.md 模板
    ├── plan.md.tmpl                  # plan.md 模板
    └── verify-report.md.tmpl         # verify-report.md 模板
```

会话产物落在宿主项目的 `.flutter-graft/<src-name>/` 下：
- `discovery.md` — Phase 0 扫描结果
- `plan.md` — Phase 1 决策 + 各 Rule 候选清单
- `graft-log.md` — Phase 2 动作日志
- `verify-report.md` — Phase 4 最终分卡

## 源起

从一次真实的 `CuteAnimal` 项目（7k 行 / 31 文件 / 9 依赖 / 8 子 feature）合并到现有 Flutter 宿主项目 `flutter_center`（baseline 402×874，CLAUDE.md 强制 screenutil + DebounceUtil + ToastUtil + module_base_page 中台 splash）的实战中沉淀。`failure-modes.md` 列举了具体踩过的坑。

## 典型用法示例

### 1. 用户给路径，safe 模式逐步确认

```
用户："把 ../CuteAnimal-main 复制到本项目"
↓
Phase 0 自动扫描 → 写 .flutter-graft/cute_animal/discovery.md
[CHECKPOINT Phase 0] 用户审阅 → 回 continue
↓
Phase 1 弹 3 个 AskUserQuestion（落点 / 适配深度 / 依赖策略） + 必要时第 4 问（哪层 tab 驱动 PageView）
[CHECKPOINT Phase 1] 用户选完 → 写 plan.md → 回 continue
↓
Phase 2 / 3 自动推进（文件搬 / 路由 prefix / asset 重写 / 9 条 Rule 全跑）
↓
[CHECKPOINT Phase 4] 报告 0 error + 资源全 OK + 结构警告（如 _StudioPageState 800 行 follow-up）
用户回 accept → 完成
```

### 2. Auto 模式一口气

```
用户："@../X-flutter 合并到本项目，不用问我了"
↓
auto 模式触发（"不用问我了" 匹配触发词）
↓
全部 5 阶段一气呵成 + 9 条 Rule + 所有 pre-flight 检查
↓
[REPORT] 最终汇报
```

## 许可

Apache 2.0
