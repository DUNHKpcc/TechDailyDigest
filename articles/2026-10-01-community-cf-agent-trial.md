# 社区实操试刊 用 cf 把 Cloudflare 资源盘点交给 Agent

> [!info] 来源与核验信息
> - 原帖：[Cloudflare CLI cf工具 OAuth 授权 更方便使唤Agent操作 Cloudflare 了](https://linux.do/t/topic/2973273)，crazypeace／LINUX DO
> - 首发：2026-10-01 00:31:46 UTC（北京时间 08:31）；已核对原帖公开时间字段
> - 辅助核对：Cloudflare 官方 [Get started](https://developers.cloudflare.com/cf/get-started/) 与 [Use cf with coding agents](https://developers.cloudflare.com/cf/agents/)，页面均标注更新于 2026-09-29
> - 核验时间：2026-10-01 13:14 UTC；最终文件大小：3012 字节，UTF-8

今天 LINUX DO 有个适合自己管服务的人看的小实践。crazypeace 把 Hermes 接到 Cloudflare 的 cf 命令行工具，展示了从自然语言指令到浏览器授权的过程。对部署 PccAgent、new-api 的人，如果域名或入口服务放在 Cloudflare，这条路线值得在测试环境试一次。

## 原帖实际走到了哪一步

作者使用 Hermes，模型是 Nous Portal 的 stealth/space-bunny-alpha。他先让 Agent 阅读 cf 文档，再启动 `cf auth login`；Agent 返回网址和验证码，由本人在浏览器完成确认。作者附图并称授权成功，还说明：若 Agent 没有主动确认，可以告诉它授权已完成。

接下来，作者建议让 Agent 列出账户里的域名、Worker、Pages、KV、D1 和 R2。这里要分清进度：帖子展示了登录流程，但没有给出完整资源查询结果，也没有部署前后对照。回帖中，他把收益概括为降低操作门槛。本文未独立复现，因此不把它写成“全自动运维已经验证”。

## 试用时先拿到一份只读清单

可以先提出一个小任务：确认当前账户，再列出域名；记录实际命令、输出和失败信息，不修改配置。官方文档提供了 `cf auth whoami` 和 `cf zones list`。官方也提醒列表查询可能分页，不能只读第一页就声称盘点完整。

有两个细节容易漏掉：cf 不复用 Wrangler 登录；环境变量中的 `CLOUDFLARE_API_TOKEN` 优先于保存的登录凭据。因此，即使网页授权成功，也应先核对当前身份与账户，避免拿旧 Token 操作错地方。浏览器授权应由本人检查权限后确认。[辅助核对](https://developers.cloudflare.com/cf/get-started/)

## 接到现有部署前的边界

已有 Wrangler 项目应先保留原部署方式。官方明确提醒：只有 Wrangler 配置、缺少 `cloudflare.config.ts` 时，直接执行 cf 的 dev、build 或 deploy 可能忽略旧配置或失败；迁移应先预览、再检查生成内容。[迁移提醒](https://developers.cloudflare.com/cf/agents/)

我的建议是先验收一份准确的资源清单，再考虑让 Agent 提交 DNS 或 Worker 修改方案。对自建 AI 网关，这能少翻几层控制台；这项实践尚不能证明上游模型会更稳定。今天的价值就在这个能小范围验证的操作入口。
