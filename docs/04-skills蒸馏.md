# 04 · Skills：把经验蒸馏成手册

`.agents/skills/` 下有 11 个技能，每个是一份 900–2,000 词的 markdown 手册，agent 干特定活时按需加载——不进默认上下文，不是常驻的长 prompt。

## 完整清单（按诞生日期）

| 日期 | Skill | 干什么 |
|---|---|---|
| 06-13 | dsh-code-review | 怎么 review 本仓库的 PR |
| 06-20 | dsh-find-simplifications | 怎么找简化/删代码的机会 |
| 07-02 | dsh-translate-docs | 中英双语文档翻译 |
| 07-04 | dsh-doc-standards | 文档分层与预算审计 |
| 07-06 | dsh-pre-push-checks | push 前跑哪些检查 |
| 07-06 | dsh-merging-stacked-prs | 怎么合并堆叠 PR |
| 07-13 | dsh-doc-site-sync | 文档站同步 |
| 07-13 | dsh-prose-standard | 技术散文写作标准 |
| 07-23 | record-browser-gif | 录浏览器演示 GIF |
| 07-26 | dsh-archive-agent-notes | 决策文档归档判断 |
| 08-09 | dsh-trim-cot-leakage | 清理"思维链泄漏" |

## 三个关键事实

**1. 没有一个 skill 是预先设计的。** 诞生顺序讲了一个故事：第 3 天出 code-review（先解决"AI 产出谁来把关"），第 10 天出 find-simplifications（AI 高产必然过度生产，要配套删减能力），最后 8 月 9 日的 trim-cot-leakage 的 commit 消息原话是 "distill the CoT-leakage purge into dsh-trim-cot-leakage"——把一次大扫除的经验蒸馏成技能。**全部是问题出现之后的沉淀。**

**2. skill 是持续重构的对象，不是写完即止。** skills 目录共 137 次提交；单看 code-review 一篇，两个月修订 25+ 次，每次对应真实的 review 经验（"distill adopted review feedback"、"codify semantic review rules"）。仓库改名、目录重组时，skill 里的引用同一批提交跟着改。

**3. 每篇都写着同一句免责声明**：这是指导不是清单/脚本，跟着代码走，保持判断力。他们明确防的就是 agent 把手册当脚本机械执行。

## 两个值得细看的样本

### dsh-code-review：给 AI reviewer 的立场设定

开头定调：一条有实据的 blocker 胜过一堆 nit。检查项全是机器门禁查不了的语义层面：追踪接口两侧实现是否匹配、每个抽象有没有真实的生产消费者、从模型视角检查它实际收到的 prompt 和工具 schema、测试断言是否真的会在预期回归时失败。

结尾一句最扎眼（转述）：收到 review 时，逐条验证对方的说法，在技术层面修复或反驳，**不要表演性附和**——这条显然是针对 LLM 爱说"你说得对"的毛病写的。

### dsh-trim-cot-leakage：对 AI 写作病最精准的解剖

他们发现 agent 写的注释和文档里满是"作者会话视角残留"：引用只有那次会话才看得到的东西（"见决策 7"、"设计稿 §4.7"）、跟已经离场的 reviewer 辩论（"这个 cast 是安全的，因为……"）、叙述改动而非状态（"不再使用旧的 X"）。

判定标准就一条：**一个只有当前代码、没有任何会话记录的读者，能否解析文中每个引用、验证每个断言？** 不能就是泄漏。手册细分了八类病症和对应改法，连"哪些看着像泄漏但必须保留"都列了（issue 编号、实测数据、外部标准引用），防止矫枉过正。

## 元系统：skill 的半自动修订流水线

最激进的一步是一篇 proposed note：《dsh-code-review 的周期性人工反馈维护》——**用 AI 流水线维护给 AI 用的 review 手册**：

1. 定期扫描已合并 PR，抓取**人类**在合并前留下的 review 意见（作者类型必须是 User，机器人反馈排除）
2. 用 git 三方比对严格验证"意见真的被采纳了"——merge 了不算、thread resolved 不算、作者回句 "fixed" 也不算，必须在代码 diff 里找到证据，还要排除"目标分支恰好也改了同一处"的假阳性
3. **两个独立配置的 AI reviewer**（不同 provider/模型，工具拒绝运行两个字节相同的可执行文件）分别判定，一致才通过
4. 通过的意见蒸馏成 skill 修改候选，再经双 AI 审同一份 diff，最后仍走人工 PR review——工具永远不自动 merge
5. 安全细节：review 内容用 128 位 nonce 包在 untrusted 标签里防 prompt 注入，子进程环境变量全部洗掉

验收标准里有真实运行记录：扫 62 个 PR、426 条人类反馈、产出 0 个候选——**0 也如实写进文档**，包括一次 adapter 幻觉 ID 被 fail-closed 兜住的细节。

还有一个反直觉的决定：这个工具本身**不入仓库**——单人维护的小工具过全套门禁不划算，于是协议入仓、实现私有，连豁免理由和"交接时如何请回来"都留了案。**每条重规则都要过成本收益关，过不了就明着豁免**——这比规则本身更能说明他们的方法论是活的。

## 蒸馏闭环

把第 2、3、4 章连起来，就是完整的生产线：

```
事故/经验发生
  → 一次性处理（postmortem / 大扫除 / review 轮次）
  → 蒸馏：能机械检查的 → verify-* 脚本进 CI
         不能机械检查的 → 写进 skill
  → 每次使用后修订（发现不准，同 PR 改掉）
  → skill 修订本身也在被半自动化
```

对比市面上"先立规范再要求遵守"的方案，这条线的方向是反的：**规范是实践的沉淀物，不是实践的前提**。

---

[← 上一章](03-机器门禁.md) · [返回目录](index.md) · [下一章：事实供给 →](05-事实供给.md)
