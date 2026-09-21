---
name: changelog-golden-rules
version: 1.1.0
description: >-
  写 changelog / release notes / 更新日志 / "这次更新了什么" 时使用，也用于审阅已有的
  changelog 草稿。强制 CHANGELOG GOLDEN RULES：只写「上一个已发布版本 → 本次发布」的
  净可观察差异；禁止写入开发历史、开发期内自引入又修掉的 bug、重构与实现细节；
  输出前必须逐条完成溯源自查。Use when writing or reviewing a changelog, release
  notes, or a "what shipped" summary.
---

# Changelog Golden Rules

## 何时使用 / 不适用

用：写 changelog、release notes、更新日志、应用商店更新说明；审阅已有的 changelog 草稿。

不用：迁移文档、内部技术复盘、commit message。

## 核心命题

**commit ≠ changelog entry。**

Commit 是开发历史；Changelog 是产品版本历史。两者本来就不应该一一映射。

Changelog 描述的是「上一个已发布版本 → 当前发布版本」的**最终可观察差异**，
而不是「开发当前版本期间发生过什么」。

任务、提交、代码改动都只是**证据**，不是条目的来源。

## 铁律

1. Compare RELEASED VERSION → NEW RELEASE. Do not describe development history.
2. Never include bugs introduced and fixed within the same unreleased development cycle.
3. Never include: refactoring, variable renaming, code cleanup, formatting, comments, internal architecture changes, implementation details, temporary regressions, debugging changes, test-only changes.
4. Include a fix only when the bug existed in a previously released version.
5. Include internal changes only when they materially affect users: performance, compatibility, security, behavior, resource usage.
6. Describe WHAT changed, not HOW it was implemented.
7. A commit/task/code change is evidence, NOT automatically a changelog entry.
8. When uncertain whether something is user-relevant, OMIT it.
9. Be terse. One short line per entry. State what changed; do not explain the bug, the cause, or the failure mode. Use the fewest words that still identify the change.

## 好坏对照

规则 6（写 WHAT，不写 HOW）：

BAD:
> Replaced NSVisualEffectView wrapper with native SwiftUI glassEffect API.

GOOD:
> Improved visual consistency with macOS.

用户无法有意义地观察到差异时，直接省略。

规则 9（短）：

BAD:
> 修复地名与所选坐标不符：拖动地图后下方地名有时仍是上一个地点，确认后还会把这个错的地名连同新坐标一起存进历史记录

GOOD:
> 修复地图选点后地名与坐标不符

## 操作流程

1. **建立基线** —— 找出上一个**已发布**版本，写出它的版本号。从未发布过的版本号不算基线；它们之间的迭代不产生条目。
2. **枚举候选** —— 列出两次发布之间的全部变更。提交、任务、diff 都只是证据。
3. **逐条过筛** —— 见下一节的强制自查，每条都要过。
4. **收敛成净差异** —— 中途新增又移除 = 净变化为零 → 不写。开发中引入又修掉 → 不写。
5. **不确定就省略** —— 宁可少一条，不要多一条（规则 8）。

## 输出前强制自查（必须执行后才允许输出）

先写出**对照基线**：`对照基线 = <上一个已发布版本号>`。写不出来就先停下，不要开始写条目。

然后对**每一条**准备写入的条目逐条回答，答不上来即删：

1. **溯源** —— 这条对应的失败现象，在对照基线版本里能复现吗？复现版本号是？
   - 答不出「能」 → **删**（规则 4）
   - 答出的版本号 ≥ 本次待发布版本号 → **删**（规则 2，说明是开发期内自己引入又修掉的）
2. **可见性** —— 用一个**用户能做的动作**描述这条变化（「用户点 X，现在会看到 Y」）。
   - 描述不出来 → **删**（规则 5、6、8）
3. **WHAT / HOW** —— 这句话里有没有出现标识符、函数名、类名、文件路径、标志位、内部框架名、算法名？
   - 有 → 改写成用户语言；改不出来 → **删**（规则 3、6）
4. **净差异** —— 这条描述的状态，在对照基线版本里是不是已经如此（本次改动被后续回滚或等价）？
   - 是 → **删**（规则 1）
5. **三态** —— 标出这条属于哪一态：已实现（在仓库里）／已发布（用户能拿到）／已实机验证。
   - 对外 changelog 里不得把「已实现」或「构建通过」写成「已修复上线」。
6. **长度** —— 这条有没有复述 bug 现象、解释成因、或复述修复过程？有 → 砍到只剩「什么变了」。
   - 一句话能说清就不要两句。默认每条一行，能一行写完就不写两行。

汇总自检（三件，缺一不可）：

- **列出被删掉的候选及理由**，每条理由必须指向具体铁律编号。理由写不出来的条目 → 说明删错了，拿回来重新过筛。
- **数条数**：条目数多于「用户可观察面」的数量，说明还在写开发历史，整份重做。
- **通读查版本号**：文中是否出现了中间未发布的版本号（例如基线 8.7.1、本次 8.9.3，却出现 8.8.x / 8.9.0–8.9.2）。有 → **删**（规则 1）。

输出末尾附一行溯源结论，格式：

> 对照基线 = 8.7.1（已发布）。已删除候选 N 条：其中 X 条违反规则 2，Y 条违反规则 3，Z 条违反规则 8。

## 常见翻车

| 翻车 | 违反 |
|---|---|
| 把本次开发引入又修掉的 bug 写进 changelog | 2 |
| 把只该待在代码注释里的实现细节写进去 | 3、6 |
| 把 commit message 直接翻译成条目 | 7 |
| 逐个罗列中间未发布的版本号 | 1 |
| 重构 / 改名 / 清理 / 格式 / 注释被写成条目 | 3 |
| 按「我们做了哪些工作」组织，而不是「用户看到什么变化」 | 1、6 |

## 写作要求

- 一条一个**可观察结果**，尽量一行说清。修复类条目用几个字点出问题即可，不铺陈现象、不解释成因、不复述修复过程。
- 跟随目标文件的既有约定：排序、分节、编号、语言、栏目名。已有条目**只追加、不重写**（除非该条目内容本身有误）。
- 分节按**用户视角的模块**或**变更类型**（新增 / 修复 / 改进 / 移除），不按代码目录。
- 已知限制就写成限制，不要包装成「已解决」。

## 示例

候选（来自提交）：修复 A 崩溃｜重构 B 模块｜把 C 从 X 换成 Y｜新增 D 功能｜修复开发中引入的 E 崩溃｜补 F 的注释

判定（对照基线 = 上一个已发布版本）：

- 修复 A 崩溃 —— A 的崩溃在基线版本里存在 → **写**
- 重构 B 模块 —— 用户不可观察 → **删**（规则 3）
- C 从 X 换成 Y —— 只有性能 / 行为 / 兼容性变了才写，否则 → **删**（规则 5）
- 新增 D 功能 → **写**
- 修复开发中引入的 E 崩溃 —— E 从未发布过 → **删**（规则 2）
- 补 F 的注释 → **删**（规则 3）

净结果：两条。落笔形态（规则 9 要求短）：

> 1、修复 A 导致的闪退
> 2、新增 D 功能

溯源结论：对照基线 = 上一个已发布版本；已删除候选 4 条（规则 2 ×1，规则 3 ×2，规则 5 ×1）。
