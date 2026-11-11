# 条件案例：双 2080 Ti 22GB + 27B Dense Decode

现场硬件、模型和目标相近时优先读取；其他平台可借鉴完整轮时和状态管理方法，具体参数须重新匹配，不能把案例写成固定答案。主流程的短→长顺序优先：先查短任务完整ms与全部传输等待，前期不做正式A/B、不调K或追接受率，正式比较在阶段8。

## 来源与边界

本案例从双 2080 Ti Decode 调研及后续本地实机执行记录提炼为方法示例，不附原始私人文档、运行地址或硬件性能承诺。历史成绩说明当时组合的实测结果；新硬件、新版本仍按当前实际部署验证。

原始场景：两张 22GB 改装 RTX 2080 Ti、Turing/SM75、27B Dense、可能退化的 NVLink，重点优化 Decode。本案例不提供框架、量化、K、TPS 或 Round 的通用默认值；必须从当前现场填写，不得臆造。

## 专项调查顺序

1. 同硬件：双 2080 Ti、2080 Ti NVLink、22GB 改卡；
2. 同芯片/同代：Titan RTX、RTX 6000/8000、TU102；
3. 同架构旁证：其他 SM75/Turing，例如 T4；
4. 同模型：完全相同模型/版本/量化优先，同系列 27B 次之，24B–32B Dense 只能旁证；
5. 高性能案例：分开普通 autoregressive Decode 与 speculative Decode。

每个外部案例记录 GPU/数量/互联、模型、量化、框架、上下文、batch/并发、普通或投机、K、TPS、Round、平均每轮有效输出、特殊优化、日期/commit 和证据等级。

## SM75 能力审计

从官方规格与现场软件分别核对 FP32/FP16/BF16/INT8/INT4/FP8/FP4、Tensor Core、显存带宽、L2、共享内存、寄存器、PCIe 和 NVLink。

必须区分：

- 硬件原生指令或 Tensor Core 支持；
- 可以存储该格式；
- 通过 unpack/dequantize 转成其他精度计算；
- 已有成熟 SM75 kernel；
- 理论可移植但需要重写。

对 Marlin、W4A16/W4A8/W4A4、FP8/FP4/NVFP4、自定义 CUDA kernel 逐项寻找 SM75 源码和 benchmark。旧 GPU 上低位权重节省容量不保证 Decode 更快。

用现场拓扑、P2P/NCCL 测试和错误日志确认 NVLink；不能从“残血”一词推算可用链路或带宽。

改装显存的容量识别不等于长时间稳定性。结合实际负载核对温度、功耗、降频、驱动错误及输出异常；没有传感器的数据标为未知。CPU governor 的收益也要记录系统范围、功耗与重启后状态；不为复现旧成绩盲目刷 VBIOS 或变更硬件。

## Whole Round 专项

若启用 speculative Decode，时间线按实现覆盖 Prepare、Draft、Target/Verify、Collective、Sampling、State Update、Host Gap、stream sync/graph 边界和 rank arrival skew。Target 与 Verify 可能重合，collective 可能与计算重叠；不强造独立阶段，也不相加重叠时间。

只有同时知道同一串行 round 集合的平均每轮有效输出 `E` 和同边界 TPS 时才反推：

`Round(ms) = E / TPS × 1000`

不能使用 `K / TPS`；接受率定义不清时不得参与计算。均值和中位数不可混用，详见 [测量契约](measurement-contract.md)。

候选从量化/反量化、GEMM/GEMV、Attention/KV、CUDA Graph/scheduler/host launch、TP/NCCL/P2P/NVLink、operator fusion、Draft/Target/Verify 中产生，并映射到本机关键路径。

原场景可用的初筛优先级是：预计 Whole Round 节省至少 1.0 ms、0.5–1.0 ms、0.2–0.5 ms、低于 0.2 ms。但这些阈值只适用于毫秒级 Round 的当前案例；其他模型按绝对值、相对比例和测量噪声重新定标。

## 从执行路径提炼的检查顺序

1. **冻结当前组合**：核对原入口、源码与已编译扩展身份、实际 Target/Draft、KV 与 SSM dtype、Graph、Prefix/align、近似 Prefill、K、并发及 CPU governor。旧计划的容量和缓存开关可能已过时；现场优先，不将多份历史文档拼成当前配置。
2. **先处理正确性告警**：真实 Agent 出现虚构工具、路径转写、错误后重复调用或异常长生成时，先按测量契约做短复现与受影响缓存检查，长链验证安排在阶段5/6；现有长链故障仅做必要定位，不全套扩测。既往短状态测试通过不排除长链问题；未定位前不能声称是 Draft 接受率、量化或缓存某一项造成。
3. **再看当前关键路径**：按 Host 重复工作、CPU 调频/等待、跨轮准备、Draft Attention、Target 矩阵和通信依次检查本机证据。历史候选只有在上游依赖图、shape 或实现改变时才重开；不要原样重跑已经证伪的 tile/segment/旁流方案。
4. **对 shape 做窄分派与负向测试**：SM75/SM 数、M/K/N、dtype、权重 scale/group、TP、页大小、workspace 与地址稳定性必须匹配；同时测不匹配时的原路径。NVFP4 检查点可能经 FP16 激活兼容 kernel 执行，不能称原生 FP4 激活计算。不同矩阵不能统一机械提高 occupancy/blocks。
5. **单独评估首次回答**：缓存复用和近似 Prefill 不属于同一种加速；approximate 状态必须有质量验收。记录端点首答案与实际 UI 首字边界，未观测 UI 就不宣称测到显示时间。
6. **验最终组合与正式应用**：短输入不等于全程短上下文；记录长输出后段长度和性能。模型输出代码/JSON 的测试不能代替外部工具往返。配置上限和并发一成功不代表长窗口及多人服务验收；按用户需要与窗口验证，未测就保留限制。

## Hybrid 状态、Graph 与旁流注意事项

- 对 Attention KV 与 GDN/SSM recurrent/conv 状态分别计容量、dtype 和生命周期，不能只验证一个最终 state tensor。
- 逐消费者审计 anchor、已接受前缀、中间 snapshot、rejected suffix、下一轮 pending token、Prefix Align 保存/恢复及页映射。具体 commit 范围以实现为准，不能把 correction/bonus 错当本轮已提交状态。
- 覆盖首拒绝、全接受、取消/无提交、实际 page/chunk/align 边界±1及会话A→B→A切换后的后续生成（按阶段决定长度，这不是性能A/B要求）；缓存池使用回落与 HTTP 健康不证明内容状态正确。
- Graph replay 使用的 buffer 地址稳定不等于内容所有权稳定。旁流 event 必须保护真实消费者结束，不能让 Draft/下一轮复用仍被 Target 提交链持有的 metadata view。先证明安全的串行顺序，再证明重叠有净收益。
- 减少 snapshot 写入若换来第二遍 recurrent 计算，必须测完整层数和全部提交长度的净收益；不能只报 record-only 提速或写流量减少。局部数值容差通过但服务级取消恢复失败时仍撤回。
- 遇到近似 Prefill、缓存或数值路径分叉，先按固定前缀核对首次差异与业务影响。既往成功记录不覆盖后续新窗口/长工具链的异常，后者优先诊断。

## Draft/DFlash 路线

草稿选型与训练边界见 [方案与阶段](full-workflow.md)，指定任务内K调优见 [任务、K与最终对标](task-k-final.md)。阶段4固定草稿与K，不因接受率低自动换草稿或训练；本案例不提供额外训练授权。

本平台额外核对 SM75 是否支持 Target 和 Draft 各自的实际 kernel，以及从其他设备迁移来的模型、scale、tap 和验证路径是否一致。已有失败 pilot 只能作为带条件的负证据；缺少新数据/损失/依赖变化时不重复训练，也不因低接受率放宽 greedy/Verify 规则。当前案例没有提供可核验的训练提速成绩。

硬件与方法原始入口：[NVIDIA Turing 调优指南](https://docs.nvidia.com/cuda/archive/12.9.1/turing-tuning-guide/index.html)、[DFlash 原始项目](https://github.com/z-lab/dflash)。这些资料不证明本案例的自定义 runtime 与已发布实现完全相同，参数与移植结论仍须核对当前 commit。
