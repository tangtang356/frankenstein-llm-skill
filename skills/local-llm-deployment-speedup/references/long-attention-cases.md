# 长注意力：可借鉴的源码入口

源码核对日期：2026-11-11。与[专项操作章](long-attention-optimization.md)配合使用；这里只列有实际代码入口的方法，不把论文标题或发布宣传当作已部署证据。下列固定版本是阅读锚点，实施前仍核对目标版本的调用者、支持范围、构建与许可证，不要求把部署升级到这些版本。

## A. 1Cat：能力分派、分页与分段归约

### 公开实现可以直接查什么

核对版本：`1CatAI/1Cat-vLLM@75e51a5bc405e85c15a7060f28c12de6ace56655`。

- [sm70_grouped_long.py](https://github.com/1CatAI/1Cat-vLLM/blob/75e51a5bc405e85c15a7060f28c12de6ace56655/vllm/v1/attention/ops/sm70_grouped_long.py)：`resolve_long_attention`、`long_attention_graph_contract`、`long_attention_page_supported`和`wrap_long_attention`。代码核对query rows、batch、head dim、六个Q头对应一个KV头、页与上下文能力，不满足时fallback；还按算子revision判断能力。
- [grouped-attention.cu](https://github.com/1CatAI/1Cat-vLLM/blob/75e51a5bc405e85c15a7060f28c12de6ace56655/csrc/attention/sm70_grouped_long/kernel/grouped-attention.cu)：grouped attention主体与`flash_attention_decode_reduce_kernel`，包含E4M3转换、分页寻址、分段结果和补偿累加。该快照已含多条路径，必须沿实际分派读选中的模板实例。

**可采取的行动：**目标设备的长上下文变慢时，先检查现有算子是否实际加载，以及当前page/query尾宽是否触发fallback，再研究tile、读取和归约。不能根据“支持SM70”就假定所有模型、头维度、KV格式都支持；旧环境变量也要在目标版本重新查定义。

## B. NInfer：按长度分派，以及Graph区间内的split容量

核对版本：`tuxKOH/ninfer-v100xN@e9509170bf691a79496c9feba06f874a4071ed30`。

在[解码launcher](https://github.com/tuxKOH/ninfer-v100xN/blob/e9509170bf691a79496c9feba06f874a4071ed30/src/ops/launcher/gqa_attention_decode.cu)查：

- `gqa_small_t_split_upper_bound`：4096、8198、16390附近使用不同每split目标长度。
- `gqa_small_t_split_count`：额外按KV dtype、query宽度与长注意力能力选择特化，说明长度不是唯一条件。
- `gqa_small_t_launch_capacity`：检查执行区间两端及策略分段端点，计算Graph工作区所需最大split数量。因为策略跨阈值可能下降，最大长度对应值不一定是区间最大值。

**可采取的行动：**出现某个长度突然变慢，先对照阈值与实际shape；出现Graph长短切换错误，检查partial容量、有效split及页id暂存界限，而非盲目加大总显存。这里的阈值服务该仓库geometry，不能照抄到另一种模型。

[长窗口算子测试](https://github.com/tuxKOH/ninfer-v100xN/blob/e9509170bf691a79496c9feba06f874a4071ed30/tests/ops/test_gqa_attention_long_context.cpp)针对split饱和后每段页数增长，使用非恒等物理页映射和少量query/head的全历史FP64参考。可借鉴“测试必须真的跨过旧容量边界”的方法；其大窗口是该实现测试范围，不是本技能默认冷测试要求。引用源码不代表目标机验证通过。

## C. FlashInfer：分段结果合并、布局接口与规划约束

核对版本：`flashinfer-ai/flashinfer@d2981ed4a64e926530ebd236168f9c39363c88e1`。

- [decode.cuh](https://github.com/flashinfer-ai/flashinfer/blob/d2981ed4a64e926530ebd236168f9c39363c88e1/include/flashinfer/attention/decode.cuh)：`SingleDecodeWithKVCacheDispatched`按序列与并行能力选择KV chunk，分段写部分输出/LSE并调用`MergeStates`；批处理路径有`VariableLengthMergeStates`。研究整个分段加合并路径，不只拿主体微测。
- [state.cuh](https://github.com/flashinfer-ai/flashinfer/blob/d2981ed4a64e926530ebd236168f9c39363c88e1/include/flashinfer/attention/state.cuh)：`state_t::merge`通过最大值重标定状态；`normalize`处理空状态。可用于复核自写归约的数值语义。
- [decode.py](https://github.com/flashinfer-ai/flashinfer/blob/d2981ed4a64e926530ebd236168f9c39363c88e1/flashinfer/decode.py)：`BatchDecodeWithPagedKVCacheWrapper`暴露NHD/HND、plan/run及工作区。`fixed_split_size`的单位是**页**，该接口说明只支持tensor-core decode；固定batch也不保证变长KV的CTA数量可供同一CUDA Graph重放。`disable_split_kv`是该版本的可选控制，不是默认提速建议。多query行还需核对plan阶段的`q_len_per_req`。

**可采取的行动：**已有FlashInfer后端优先检查planner、布局、query行数和实际选中backend；若借鉴源码移植到旧卡，逐项核对指令和构建架构，不能从通用Python接口推断SM70支持。改变split后一起计入plan、workspace、merge和Graph回退成本。目标机收益由实际完整路径验证。

## D. llama.cpp：结合驻留、SM与KV tile数量分配工作

核对版本：`ggml-org/llama.cpp@b8abe41a5081728f9f4c824ef810902a11ab7ffd`。

- [fattn.cu](https://github.com/ggml-org/llama.cpp/blob/b8abe41a5081728f9f4c824ef810902a11ab7ffd/ggml/src/ggml-cuda/fattn.cu)：`ggml_cuda_flash_attn_ext_mma_f16_switch_ncols1/ncols2`依据硬件、query数、head形状与GQA条件分派，包含Volta相关分支。
- [fattn-common.cuh](https://github.com/ggml-org/llama.cpp/blob/b8abe41a5081728f9f4c824ef810902a11ab7ffd/ggml/src/ggml-cuda/fattn-common.cuh)：`launch_fattn`查询实际kernel共享内存条件下的occupancy，结合SM与KV tile数量选`parallel_blocks`，兼顾末波利用率和额外工作量；另有Stream-K分工。多块输出经`flash_attn_combine_results`或相应fixup收束。

**可采取的行动：**当长decode线程块不足时，用当前shape的实际可驻留块数估算起点，不按GPU利用率猜split数量。改tile后重新计算寄存器/共享内存驻留与partial成本；记录实际分派。向量、MMA、Stream-K及特定稀疏分支是不同条件路径，不能宣称所有卡都执行同一个Flash Attention。这里借鉴调度方法，不把上游条件下的收益承诺给当前部署。

## E. vLLM历史实现：短路径与长路径不是同一最优解

为展示清楚稳定的分派案例，固定读取历史tag **v0.6.1** 的[paged_attn.py](https://github.com/vllm-project/vllm/blob/v0.6.1/vllm/attention/ops/paged_attn.py)，不把它当作当前vLLM所有后端的规则。

`PagedAttention.forward_decode`使用`_PARTITION_SIZE=512`计算分段数。源码在max_seq_len不大于8192，且只有一个partition或序列数×头数较大时选择V1，以避免额外归约；其他情况走V2，并分配`tmp_output`、`exp_sums`和`max_logits`。长序列切换也与共享内存约束有关。

**可采取的行动：**不要把“分段更先进”变成全局强制开关；短输入或batch已充分并行时，多一次归约可能倒退。当前vLLM部署需从实际attention selector和选中backend继续定位，不能直接修改历史文件路径或把8192作为新平台统一阈值。

## 选择和落地

优先复用目标底座已有实现；没有合适路径时，再移植与硬件/shape匹配的最小完整单元：planner或launcher、主体、归约、工作区与对应数值检查。来源仓库、版本、符号、适用条件和实际收益保留在部署记录中；移植代码遵循来源许可证。仅用于阅读比较不等于实际采用，不自动安装、替换底座或给所有参考仓库点Star。

执行AI按[专项操作章](long-attention-optimization.md)在目标机完成生效与任务验收后，才能报告新的提速结论。
