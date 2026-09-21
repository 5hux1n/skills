# Skills

我的 agent 技能合集，按 [skills.sh](https://skills.sh) 的目录约定组织：每个技能一个目录，内含 `SKILL.md`（YAML frontmatter + 正文）。

## 安装

```bash
npx skills add 5hux1n/skills@changelog-golden-rules
npx skills add 5hux1n/skills@ios-tweak-packaging
```

或手动放到 agent 的技能目录（DeepSeek Harness 是 `~/.dsh/skills/`）：

```bash
cp -r ios-tweak-packaging ~/.dsh/skills/
```

## 技能列表

### `changelog-golden-rules`

写 changelog / release notes / 更新日志时使用，也用于审阅已有的 changelog 草稿。

核心命题：**commit ≠ changelog entry。** Commit 是开发历史，Changelog 是产品版本历史，两者不应该一一映射。Changelog 描述的是「上一个**已发布**版本 → 本次发布版本」的最终可观察差异，而不是「开发这个版本期间发生过什么」。

九条规则：

1. 只对比「已发布版本 → 新版本」，不描述开发历史
2. 不写「开发期内引入、又在同一个未发布周期里修掉」的 bug
3. 不写重构、变量改名、代码清理、格式化、注释、内部架构调整、实现细节、临时回退、调试改动、纯测试改动
4. 修复类只在该 bug **在上一个已发布版本中确实存在**时才写
5. 内部改动只有在**实质影响用户**时才写（性能 / 兼容性 / 安全 / 行为 / 资源占用）
6. 写「什么变了」，不写「怎么实现的」
7. 提交 / 任务 / 代码改动是**证据**，不自动成为条目
8. 不确定是否与用户相关 → **省略**
9. **简短**。一条一行说完；不解释 bug、不解释成因、不复述失败现象

另外包含一节「输出前强制自查」：逐条要求给出该问题所在的已发布版本号，答不上来即删；并强制列出被删候选及理由、统计条数、通读排查中间未发布的版本号。

好坏对照：

> BAD: 修复地名与所选坐标不符：拖动地图后下方地名有时仍是上一个地点，确认后还会把这个错的地名连同新坐标一起存进历史记录
>
> GOOD: 修复地图选点后地名与坐标不符

### `ios-tweak-packaging`

构建、打包、验证、排查 iOS 越狱插件（Theos / Logos）时使用，尤其是 **roothide（RootHide）** 与 **巨魔 / TrollStore 注入**场景。

把踩过的坑固化成检查项，含一张「症状 → 病因」速查表：

| 症状 | 病因 |
|---|---|
| 所有配插件的 App 一启动就闪退，`pc=0` | roothide 缺 rpath，弱符号 CydiaSubstrate = NULL |
| 开机/注销后异常，或插件完全不生效 | 用 rootless 布局冒充 arm64e 交付 |
| 覆盖安装报 `trying to overwrite` | 本地 deb 包名与已装包不一致 |
| 守护进程不启动 | LaunchDaemon plist 是文本格式，或路径硬编码 `/var/jb` |
| SpringBoard 无限安全模式 | `_logosLocalInit` 无条件构造器 |
| 日志或 `%orig` 一调就崩 | 接口声明类型与实际签名不符 |
| hook 语法正确但完全不生效 | 目标是 Swift 类，走 vtable 派发 |
| 启动后一两百毫秒就崩 | `%ctor` 里碰了目标 App 的类（如 Swift 类触发 MMKV 初始化） |
| 巨魔注入后 App 打不开 | 强链接 CydiaSubstrate + `.jbroot` rpath 解析不到 |
| `make` 报路径乱码 | 中文路径 + GNU Make 3.81 |
| class-dump 说没有 ObjC 运行时信息 | class-dump 太老，不认识 chained fixups |
| 搜二进制说字符串没编进去 | 非 ASCII 字面量在 `__ustring`，是 UTF-16 |

章节：

- §0–1 环境对照表 + 构建命令 + Makefile 骨架
- §2 roothide 四个专属坑（rpath / jbroot / plist 格式 / 包名自洽）
- §2.5 巨魔 / TrollStore 注入：弱链接 + `dlsym` 判空兜底 + 切片架构匹配
- §3 Logos 代码坑（`%group` 门控、签名必须精确、别盲 hook Swift 类、`%ctor` 里不要碰目标 App 的类）
- §4 工具链坑（`LC_ALL=C`、class-dump 失效后的 `otool -oV` 替代方案、UTF-16 字符串误判）
- §5 交付前验证清单（从 deb 里实读，不看「编译成功」）
- §6 症状 → 病因速查表

## License

MIT
