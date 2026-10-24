# 重构 Agent 后怎么查回归：Stately Agent 新增 compareRuns
> [!info] 来源与核验信息
> - 草稿 ID：TD-20261023-001；版本：v1
> - 原帖标题：@statelyai/agent@2.0.0-alpha.30
> - 作者：davidkpiano（关联提交）；发布账号：github-actions[bot]
> - 平台：GitHub，statelyai/agent
> - 原帖：https://github.com/statelyai/agent/releases/tag/%40statelyai/agent%402.0.0-alpha.30
> - 发布／本次事件时间：2026-10-22 15:54:56 UTC（北京时间 23:54:56）
> - 核验时间：2026-10-23 12:57 UTC（延迟补执行；不是首发时间）
> - 固定实现提交：7e97c1f6c6e079db3233b4a86ffd0b82107a29f2
> - 最终 UTF-8 Markdown 实测大小：3671 字节 / 3.58 KiB

## 今天选了什么

Stately Agent 的 alpha.30 预发布新增了 `compareRuns`：让旧实现与候选实现运行相同场景，比较最终结果和副作用，而不是要求工具调用顺序完全一致。发布时间落在本次定时任务的 24 小时窗口内。[发布记录](https://github.com/statelyai/agent/releases/tag/%40statelyai/agent%402.0.0-alpha.30)

这是一篇开源维护者实现解读，不是独立用户性能实测。今天检查了 V2EX 和 IDC Flare 的公开内容，未从所检查帖子中找到更适合、证据更完整的实测选题，转而选择这个可定位到固定提交的更新。

## 判定关注的不是“步骤一模一样”

接口接收场景集合、baseline 和 candidate。发布说明把比较结果分为 same、intended 和 regression。其意义是：迁移 Agent 时，可以保留业务行为而调整内部步骤，不必机械复刻旧流程；旧实现本身违反规则时，也不该把修复误判为退步。[发布说明](https://github.com/statelyai/agent/releases/tag/%40statelyai/agent%402.0.0-alpha.30)

固定提交中的评估文档要求比较 outcome、effects 和 violations；候选违反规则、抛出异常，或出现其他未被解释的差异，都会落入回归分类。副作用应完整记录，否则只比较“最终成功”可能漏掉不应发生的写入。[固定提交](https://github.com/statelyai/agent/commit/7e97c1f6c6e079db3233b4a86ffd0b82107a29f2)

## 使用时先把评价依据写清楚

以下是本文的实践建议，不是实测结果：

- 先冻结场景及预期结果，再替换实现，避免边改代码边修改验收答案。
- 把文件写入、外部请求和状态修改纳入观察范围，不只记录最后一条回复。
- 优先用可控的模型替身定位流程差异，再单独评估真实模型的随机性。
- 把“这批样例没有回归”与“所有情况下等价”分开表述。

例如，一个 Agent 最终都回复“处理完成”，但候选实现额外发送了一次请求，单看文本无法发现区别。这个例子说明的是应如何设计 effects，并不声称工具能自动捕获所有外部行为。

## 核验范围与限制

已交叉检查 Release 页面、GitHub Release API 的发布时间、关联固定提交及其中的评估文档。提交中可见新增测试文件，但本次未独立运行测试，也未取得独立用户复现；不能把“存在测试文件”写成“测试已通过”。

该版本仍是 alpha 预发布。比较依赖使用者提供的场景、预期与观测数据，遗漏的副作用不会因为引入比较器而自动可见。文档也提示真实模型需要重复运行或使用确定性替身。[实现与限制](https://github.com/statelyai/agent/commit/7e97c1f6c6e079db3233b4a86ffd0b82107a29f2)

值得借鉴的是“同场景、双实现、比较结果与副作用”的验证方式，而不是据此宣称 Agent 已经安全可靠。
