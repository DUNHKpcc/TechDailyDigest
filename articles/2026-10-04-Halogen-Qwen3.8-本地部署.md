# Halogen Flash：AI Max+ 395 上本地跑 Qwen3.8，50 tok/s 先看清条件再谈速度

> [!info] 来源与核验
> 原文标题：〖教程〗用aimax395部署qwen 3.8 flash next并达到50toks/s (Halogen flash server)
> 作者：LeonLee
> 平台：LINUX DO · 开发调优
> 原文链接：https://linux.do/t/topic/2980693
> 原始发布时间：2026-10-03 14:55 UTC；搜索结果中的原帖时间为 October 3, 2026 2:55pm，本稿按该页面显示时间记录
> 选题窗口：2026-10-03 06:00 至 2026-10-04 06:00 UTC
> 核验时间：2026-10-04 07:14:12 UTC
> 稿件 ID：TD-20261004-002 · v1
> 仓库交叉核验：peonist-ai/halogen-flash-server commit 7f31bbd4021f217a1be9776bdb7304bcf8eca62d，0.16.2，提交时间 2026-10-03 04:29:22 UTC
> 最终文件：UTF-8，8605 bytes（8.4033203125 KiB）

## 先说结论

这篇 LINUX DO 帖子不是“又一个模型发布消息”，而是一份可以照着检查的本地部署记录：作者在 AMD AI Max+ 395 机器上，用 Ubuntu、Podman、ROCm 和 Halogen Flash Server 跑 Qwen3.8-Flash-Next，声称可达到约 50 tok/s，并用 oh-my-pi 跑了一个赛车游戏生成任务。

最值得保留的不是“50 tok/s”这个孤立数字，而是它附带的前置条件：BIOS 里把 UMA/显存 carve-out 调到最小、使用 gfx1151 对应的 AMD 栈、给容器暴露 `/dev/kfd` 和 `/dev/dri`、设置采样与思考参数。仓库的同一项目文档也强调，参考数字来自特定机器、功耗、ROCm 版本和上下文长度；换机器或换功耗档位，不能直接照搬。

原帖是单作者实测，评论区只有一条硬件购买讨论，没有第二台机器的独立复现实验。仓库 README 和固定提交能够核对命令、参数、性能条件与兼容边界，但不能替作者证明帖子中的赛车游戏演示或“50 tok/s”在其他机器上也成立。

## 原作者的配置与步骤

原帖给出的前提是：Ubuntu（推荐）、AMD AI Max+ 395 机器、Podman。作者特别提醒先在 BIOS 中把显存分配设为最小值，帖子中记录的机器是 512 MB。随后按三步完成：

1. 安装 ROCm，使用 `gfx1151` 对应的 AMD 软件包
2. 从 ModelScope 获取模型文件
3. 用 Podman 启动 Halogen Flash Server

原帖给出的容器参数包含 `/dev/kfd`、`/dev/dri`、`--ipc=host`、`--ulimit memlock=-1:-1`，并把 API 暴露在 8731 端口。模型参数包括 `HALOGEN_CHECKPOINT`、`HALOGEN_MODEL_ID=Qwen/Qwen3.8-Flash-Next`、`HALOGEN_VISION_TOWER=1`、`HALOGEN_TEMPERATURE=0.6`、`HALOGEN_TOP_P=0.95`、`HALOGEN_TOP_K=20`、`HALOGEN_ROPE_YARN=4` 和 `HALOGEN_REASONING_EFFORT=medium`。帖子使用的镜像标签是 `ghcr.io/peonist-ai/halogen-flash-server:0.15.1`。

本文保留这些公开配置以便复核，但没有在本机执行安装，也没有复制或保存作者的任何机器凭据、IP 或口令。

## “50 tok/s”到底是什么数字

原帖标题写的是达到 50 tok/s，正文还说演示任务使用 oh-my-pi、中等思考强度、耗时约 25 分钟。它没有给出完整的 prompt token 数、输出 token 数、并发数、功耗曲线或逐次测量表，所以不能把标题数字当成可重复基准。

在固定的 0.16.2 仓库 README 里，维护者给出了更完整的参考机条件：AMD Ryzen AI Max+ 395、128 GB 内存、ROCm 7.14.0、约 85 W 持续封装功耗。对应表格中的数字是：

- 8192 token prompt 的 prefill：1584 tok/s
- 32768 token prompt 的 prefill：1567 tok/s
- 131072 token prompt 的 prefill：1517 tok/s
- 32768 token 上下文、使用 draft head 的 decode：46.0 tok/s
- coding-agent turn、draft head 加 prompt lookup：55.7–56.3 tok/s
- 关闭推测、串行 greedy decode：1500 / 32768 token 上下文时分别 37.6 / 34.1 tok/s

这些仓库数字能解释“约 50 tok/s”可能来自哪类测法，但它们不是原帖作者那台机器的独立重测。比较时至少要同时记录镜像版本、模型权重/精度、上下文长度、temperature、reasoning effort、并发数、功耗和是否启用 speculation。

[原帖](https://linux.do/t/topic/2980693) · [固定版本仓库 README](https://github.com/peonist-ai/halogen-flash-server/blob/7f31bbd4021f217a1be9776bdb7304bcf8eca62d/README.md) · [固定提交](https://github.com/peonist-ai/halogen-flash-server/commit/7f31bbd4021f217a1be9776bdb7304bcf8eca62d)

## 对本地 Agent 和 API 客户端的实际意义

Halogen 在固定版本 README 中提供 OpenAI-compatible `/v1` 接口，并列出 `/v1/chat/completions`、`/v1/responses` 等使用方式；这意味着它可以作为本地模型服务，被支持这些协议的客户端或 Agent harness 调用。仓库还列出 Anthropic Messages API 和 Codex/Responses API 的使用章节，但实际接入前应以运行实例的 `/health` 输出为准。

这里有三个容易忽略的边界：

- OpenAI-compatible 不等于与每个 Agent 客户端完全兼容。工具调用、reasoning 字段、取消、流式响应和 response store 是否可用，要看运行版本和客户端协议
- 原帖的 `temperature=0.6`、`reasoning_effort=medium` 是作者这次配置，不是所有任务的最佳值。仓库文档说明请求字段会覆盖服务端默认值，temperature 为 0 时才走 greedy 和可复现路径
- 视觉能力不是默认开启。需要挂载相应 vision tower；仓库还说明通过 URL 让服务端自行抓图会被拒绝，应传 data URL 或 base64

如果只是想验证“能不能服务 Agent”，先用一个固定 prompt、固定 seed（如果客户端支持）、短上下文和单并发请求，记录首 token 时间、生成速度、错误码和 `/health` 版本，然后再逐步增加上下文或开启视觉输入。

## 实用的排错顺序

固定版本 README 的部署逻辑给出了比原帖更可靠的排查顺序：

1. 先确认硬件确实是 AMD Strix Halo / gfx1151，系统是原生 Linux；文档明确不把 WSL2 当作支持的宿主路径
2. 确认 BIOS 的 UMA frame buffer 是最小明确值，而不是 Auto；Auto 在某些主板上可能吞掉几十 GB 系统内存
3. 确认 `/dev/kfd`、`/dev/dri` 权限，以及 ROCm 与内核的 gfx1151/SVM 支持
4. 使用固定镜像标签，避免 `latest` 在复现过程中漂移；仓库明确建议把可持久化部署固定在版本标签
5. 启动后先查日志和 `/health`，确认模型版本、API、采样默认值、视觉支持和 token budget
6. 如果启动时报 out of memory，先调整 KV pool 或最大 prefill token，不要只盲目增加 slot 数
7. 如果长上下文变慢，检查 lookup table 的磁盘和页缓存压力；速度下降不一定是模型本身变慢

这些排错项比“换一个参数试试”更有价值，因为它们对应明确的资源或兼容性假设。

## 证据限制与风险

本次证据分成两层：

- **单作者实测**：LeonLee 的帖子、标题数字、oh-my-pi 赛车游戏演示和“约 50 tok/s”说法，均来自同一作者；没有第二台 AI Max+ 395 的完整复现实验
- **仓库交叉核验**：固定提交的 README、Quickstart、测量条件、容器命令和 API 边界可以确认项目确实支持这些配置方向，但仓库自己的参考机数据不能证明原帖演示结果

还要注意时间和版本差异：原帖命令写的是 Halogen 0.15.1；本文为避免混入之后版本，仓库链接固定到 2026-10-03 04:29:22 UTC 的 0.16.2 提交。0.15.1 与 0.16.2 的启动日志、默认值、问题修复和模型文件可能不同，不能把两者当成同一可执行物。

本稿没有执行容器、没有下载约百 GB 权重、没有验证测速，也没有确认作者的机器功耗或最终生成结果。不要在生产主机上直接粘贴带有 `--device`、`--ipc=host` 和大容量模型下载的命令；先审阅镜像、许可证、权重来源和主机权限，并准备回滚用的固定标签。

## 值得带走的判断

这篇帖子适合作为“本地推理性能如何写成可复现记录”的小样本：标题给出一个直观速度，步骤给出硬件与参数，仓库再补上上下文、功耗、版本和测量口径。真正有用的复现报告，应同时公布这些条件，并把生成质量、长上下文退化、显存/内存占用和 API 兼容性分别测试。

不要只记住“395 能跑到 50 tok/s”。更稳妥的结论是：在特定的 Strix Halo、ROCm、功耗和 Halogen 版本组合下，Qwen3.8-Flash-Next 可以被做成本地 OpenAI-compatible 服务；速度、稳定性和 Agent 兼容性仍需按自己的硬件与请求形态重新验证。