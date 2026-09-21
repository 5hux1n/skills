# Skills

我的 agent 技能合集，按 [skills.sh](https://skills.sh) 的目录约定组织：每个技能一个目录，内含 `SKILL.md`（YAML frontmatter + 正文）。

## 安装

```bash
npx skills add 5hux1n/skills@changelog-golden-rules
```

或手动放到 agent 的技能目录（DeepSeek Harness 是 `~/.dsh/skills/`）：

```bash
cp -r changelog-golden-rules ~/.dsh/skills/
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

## License

MIT
