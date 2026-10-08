# BlueField-3 上用 VPP/ACL 做 100G 线速包过滤：实测瓶颈在小包与缓存

> [!info] 来源与核验信息
> - 原帖标题：Packet Filtering with VPP on BlueField-3 Arm Cores and 100G NICs
> - 作者：Denys Haryachyy
> - 原帖平台：个人技术博客（FastNetMon 于 2026-10-05 获授权转载）
> - 原帖链接：https://haryachyy.wordpress.com/2026/10/01/packet-filtering-with-vpp-on-bluefield-3-arm-cores-and-100g-nics/
> - 转载链接：https://fastnetmon.com/2026/10/05/packet-filtering-with-vpp-on-bluefield-3-arm-cores-and-100g-nics/
> - 原帖发布日期：2026-10-01；转载/事件日期：2026-10-05
> - 核验时间：2026-10-06 06:01 UTC（北京时间 14:01）
> - 稿件版本：v1
> - 稿件 ID：TD-20261006-001
> - 最终 UTF-8 Markdown 大小：6263 bytes（6.116 KiB）
> - 证据等级：作者单次实机测试；FastNetMon 获授权转载；未找到独立复现同一 VPP/ACL 工作负载的报告

## 结论先看

作者把 VPP 的 ACL 过滤插件编译为 arm64，运行在 NVIDIA BlueField-3 DPU 的 16 个 Cortex-A78AE 核心上，并用两张 ConnectX-5 Ex 作为流量发生器，通过两条 100G 链路压测。这个具体配置在 IMIX 和 256B 及以上报文上可以达到单口 100G 线速；64B 小包约 60 Mpps、42 Gbps，128B 约 52 Mpps、64 Gbps，瓶颈明显转为 CPU/缓存，而不是规则数量。

这是一份高质量的工程实测，适合用来估算 DPU 做 DDoS/ACL 数据面的上限，但不能直接外推到所有 BlueField-3、VPP 版本或不同规则插件。

## 测试环境

- DPU：NVIDIA BlueField-3，ConnectX-7，2×100G uplink
- DPU CPU：16× Arm Cortex-A78AE，2.13 GHz，30 GB DDR5，16 MB 共享 L3
- 主机：AMD EPYC 9534
- 流量发生器：Ryzen 9 9950X，2× ConnectX-5 Ex，TRex 3.06
- 软件：BlueField bundle 2.9.1、DOCA 24.11、Ubuntu 22.04、5.15.0-1057-bluefield
- VPP：25.10，arm64，DPDK mlx5，Docker 容器运行
- 内存与队列：2 MB hugepages×4096，VPP heap 4G；每端口 12 个 RX/TX 队列，descriptor 4096，IPv4/IPv6 RSS
- CPU 绑定：core 0 运行 VPP 主线程，core 1–12 运行 12 个 worker，core 13–15 留给 Linux、OVS 和 SSH
- 测量方法：读取 VPP `show hardware-interfaces` 的 `rx_good_packets` 与 `rx_missed_errors`，每个点观察 15 秒；`rx_missed_errors` 增长表示 Arm 核心已经饱和

## 吞吐实测

单口输入、ACL drop 规则、12 个 worker 的结果：

| 报文 | 丢弃吞吐 | 转发吞吐 | 结论 |
|---|---:|---:|---|
| 64B | 60 Mpps，42 Gbps | 50 Mpps，35 Gbps | 小包受 CPU 限制 |
| 128B | 约 52 Mpps，64 Gbps | 原文未给出同等完整表项 | 仍明显低于线速 |
| IMIX | 33.8 Mpps，约 100 Gbps | 33.8 Mpps，约 100 Gbps | 达到单口线速 |
| 1500B | 8.1 Mpps，约 100 Gbps | 8.1 Mpps，约 100 Gbps | 达到单口线速 |

双口同时加载时，1500B 报文约 16 Mpps、200 Gbps 丢弃；64B 报文总吞吐仍约 59 Mpps。作者通过 `set interface rx-placement` 把两个端口的 RX 队列分别固定到两组 worker，恢复了双口小包场景的最佳结果，避免每个 worker 同时轮询两个端口造成缓存争用。

## 规则数量不是主要瓶颈，工作集才是

作者加载了 983,045 条 ACL 规则，并将其与 10,000 条规则比较。对于相同匹配形状，单包成本都约 29.1 ticks，说明未被访问的规则主要停留在哈希结构中，规则总数本身没有明显影响。

真正影响性能的是活跃流和匹配形状：

- 从 10,000 条增加到 983,000 条、匹配形状不变：29.1 → 29.1 ticks
- 增加 5 条端口范围规则、出现第二种匹配形状：29.1 → 47.7 ticks
- 活跃流从 1,000 增至 10,000：29.1 → 56.6 ticks

作者解释为工作集超出 512 KB L2 与 16 MB L3 后，缓存命中率下降。BlueField-3 的 `show runtime` tick 是 330 MHz 的 Arm 通用计时器，约 29 ticks 对应约 190 个核心周期，不能直接和 x86 的 TSC tick 数字比较。

## 部署要点与风险

1. 必须把卡切到 DPU mode。作者用 `mlxconfig` 设置 `INTERNAL_CPU_MODEL=1`，并强调只有整机冷启动才会生效；温重启不够，切换前两侧都要 `sync`。
2. DPDK 需要从 OVS bridge 释放 uplink，管理流量仍走独立的 `oob_net0`，否则 SSH 可能随数据口一起中断。
3. 容器需要 `--privileged`、host network、`/dev/hugepages`、`/dev/infiniband` 和 `/run/vpp`，这意味着隔离边界和权限要求都比较高。
4. 预编译 FD.io 包使用通用 `armv8-a+crc` 基线；若要针对 Cortex-A78AE 使用 `-mcpu=cortex-a78ae`，需要连同 VPP/DPDK 数据路径一起从源码构建，单独编译插件不足以完成优化。
5. 两端口场景应按端口固定 RX 队列；作者观察到 14 个 worker 反而因为共享 L3 竞争低于 12 个 worker。

## 交叉核验与证据限制

- FastNetMon 转载页与作者个人博客的硬件、软件、队列、测量方法和主要结果一致，转载页明确标注“经作者许可转载”。
- NVIDIA/DPDK 的 BlueField-3 性能报告显示，在另一套 DPDK 工作负载下，12 个 Arm 核心可在大于等于 512B 报文达到较高线速，且 64B 报文吞吐显著下降；这只验证硬件性能量级，不能视为对本文 VPP ACL 数值的独立复现。
- 本文没有提供完整的原始 pcap、TRex 配置文件、源码仓库或第三方实验日志；ACL 过滤插件和规则分布也会改变结果。因此不能把“100G 线速”理解为任意小包、任意 ACL、任意 VPP 构建都能达到。
- 结论更适合用于容量规划：大包/IMIX 场景可接近链路上限，小包防护应按约 40–64 Gbps 的单口 CPU 上限预留，并优先做队列亲和性和缓存工作集测试。

## 核验结论

该资料属于可复核配置清晰、测量方法明确的单作者实机报告，日期和转载关系已核验。工程结论可信度高于普通经验贴，但仍缺少独立复现；采用前应在自己的 BlueField 固件、DOCA、VPP/DPDK 构建和规则集上重新压测。

---

审核方式：回复“通过”批准 TD-20261006-001 v1 公开提交到 GitHub；回复“修改：意见”要求生成修订稿并重新审核；回复“跳过”则本篇不发布、不导入。
