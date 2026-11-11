# 平台与目标路由

阶段1第一次接触目标电脑时先读本文件。只读取现场，不把下列命令当成必须存在的工具。完整部署优化的总顺序以 [主入口](../SKILL.md) 为准；已有部署或限定范围诊断只核对相关项，不强制补跑无关阶段。

## 1. 确认执行边界

记录：本机或 SSH、操作系统、容器/虚拟机、工作区、权限、联网状态、代理、磁盘余量、现有服务与真实调用入口。已有部署时先识别唯一权威入口，不另建演示服务。

## 2. 识别计算平台

| 平台 | 常见只读线索 | 首要风险 |
|---|---|---|
| NVIDIA CUDA | `nvidia-smi`、驱动/CUDA、compute capability、拓扑/P2P | 新量化或 kernel 不支持旧架构；显存够不等于 kernel 快 |
| AMD GPU | 系统设备清单、实际 Vulkan/HIP/ROCm backend；可用时查 `rocminfo`、`amd-smi` | 不因品牌指定 ROCm；CUDA 专用优化不可直接套用，OS/架构/runtime 支持需逐项核对 |
| AMD 统一内存 APU（如 Ryzen AI Max / Strix Halo） | CPU/APU 型号、安装内存、系统可见内存、UMA/VGM 分配、GPU 预算；按下节核对 | GPU 可使用统一内存；系统可见 RAM 或传统显存字段不能单独代表整机容量或模型上限 |
| Apple Silicon | `system_profiler`、统一内存、Metal/MLX runtime | 统一内存容量与带宽、模型格式和 Metal kernel 是主要边界 |
| CPU | CPU 型号、核心/NUMA、RAM/带宽、指令集、线程和 BLAS | 量化格式、内存带宽、NUMA/线程设置通常比 GPU 参数重要 |
| Intel GPU / 其他加速器 | 系统设备清单、厂商 runtime、engine backend 列表 | 按实际 backend 核对，不推断为 CUDA/ROCm |
| 多机/混合设备 | 网络、RDMA/以太网、设备映射、offload | 通信、分片与跨设备搬运可能主导延迟 |

Windows、Linux 和 macOS 使用各自等价工具。ROCm 工具可能是 `amd-smi`、`rocm-smi` 或安装版本的等价接口；CLI 缺失时回到设备清单和加载日志。对于跨厂商后端按实际 backend 识别，不仅按 GPU 品牌推断。不要为了盘点先安装大型依赖。

### 旧设备与后端可用性

- 分开核对硬件指令集/架构、驱动、工具链、发布包实际包含的后端及模型算子支持。CPU 包存在某个微架构 DLL 不等于已选中它；模型架构名称出现在源码也不证明当前组合可运行。盘点阶段查帮助与已有日志，获准试跑后再确认实际加载后端及真实请求。
- `nvidia-smi` 的 CUDA 版本表示驱动支持上限，不证明已安装该 Toolkit 或下载包支持此 GPU。升级驱动不能补回二进制缺少的架构/kernel；旧 GPU 的工具链边界须查原始说明，例如 [NVIDIA 架构支持说明](https://developer.nvidia.com/blog/navigating-gpu-architecture-support-a-guide-for-nvidia-cuda-developers/)。不凭显存能装多少层承诺卸载加速倍数。
- Vulkan loader DLL 存在不等于驱动/设备可用；初始化失败先报告当前 backend 不可用。Windows 的驱动发现还包括 PnP 等位置，单个 `Khronos\Vulkan\Drivers` 键或某个安装目录缺失不能证明驱动被精简，更不能凭第三方软件目录归咎该软件。按 [Khronos 驱动发现说明](https://github.com/KhronosGroup/Vulkan-Loader/blob/main/docs/LoaderDriverInterface.md)及实际 loader 日志定位；设备列表为空先核对命令语义与加载失败，不编造“新版不注册后端”的原因。
- 内存条数量不直接证明实际通道布局，规格计算的带宽要标理论值。已验证的 CPU 路径可以先交付；未核实的 GPU/其他引擎保持未决，不称“唯一可行”或以故障猜测触发驱动升级。

### 统一内存与容量口径

先识别独显、统一内存或混合设备，再做容量筛选。Ryzen AI Max 的 UMA/VGM 可把部分系统内存分配给 GPU；不是只有 Apple 使用统一内存，也不能把普通独显借用系统内存视为同等带宽的显存扩容。

1. **分开采集并标注来源/单位**：整机安装物理内存、OS 可见/当前可用 RAM、GPU 专用分配、共享内存上限、当前 backend 的 heap/budget/usage。Windows 用 `Win32_PhysicalMemory.Capacity` 求和，并可用 `GetPhysicallyInstalledSystemMemory` 交叉核验安装量；`Win32_OperatingSystem.TotalVisibleMemorySize` 与 `FreePhysicalMemory` 是另一口径，不能直接当安装量。SMBIOS 缺失或相互矛盾时保留未知，不从处理器支持的最大容量补值。
2. **解释分配，不能先判错**：例如安装 128 GiB，OS 可见约 64 GiB、GPU 专用分配约 64 GiB，可能是 UMA 预留造成，不能报告为“整机只有 64 GiB”。Windows 的 `Win32_VideoController.AdapterRAM` 是 32 位字段，大容量设备可能截断或报告不完整，不用它断言只有 4 GiB 显存。用 DXGI 的 64 位专用/共享容量、可用的当前预算及驱动/固件配置交叉核对；未读到具体 UMA/VGM 设置时只报告已测分配，不编造设置值。
3. **共享数字不能相加当额外内存**：DXGI 的共享上限与 Vulkan 的 heap 大小/预算不是独立内存条，也不是当前空闲量。引擎总数大于 OS 可见 RAM 时先核对安装量、分区、字段定义、重复 heap/适配器与计量单位；不立即断言驱动异常，也不直接认定模型能装下。多条适配器记录需结合设备标识/位置核对物理设备数，不能按同名记录数累加容量。
4. **按实际内存域设预算**：GPU 权重/KV/工作区与 CPU 加载、映射/副本、运行缓冲及其他应用分别核算，统一物理页不重复计数，独立副本不能漏算。GPU 已预留区域不包含在 OS 剩余 RAM 时，不能用后者作为全部 GPU 权重的硬上限；同时预留 CPU 加载和系统空间。分页/提交上限不等于高速驻留容量。容量未核实前保留待定，不仅凭一个字段自动缩小推荐模型或关闭自动适配；配置候选仍须满足改前依据与回归规则。
5. **读取与调整分开**：支持可调分配不代表任意比例、即时转换或所有 backend 都可用。BIOS/UMA/VGM、驱动、分页文件和重启涉及共享环境，不在盘点中自动修改。实际模型上限与速度由当前分配、backend 和有界实测确认，不能套用厂商最高配示例。

口径依据：[AMD UMA/VGM 说明](https://www.amd.com/en/blogs/2025/faqs-amd-variable-graphics-memory-vram-ai-model-sizes-quantization-mcp-more.html)、[Windows 安装内存 API](https://learn.microsoft.com/en-us/windows/win32/api/sysinfoapi/nf-sysinfoapi-getphysicallyinstalledsystemmemory)、[AdapterRAM 字段](https://learn.microsoft.com/en-us/windows/win32/cimwin32prov/win32-videocontroller)、[DXGI 容量](https://learn.microsoft.com/en-us/windows/win32/api/dxgi/ns-dxgi-dxgi_adapter_desc)、[DXGI 当前预算](https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_4/ns-dxgi1_4-dxgi_query_video_memory_info)。厂商示例不代表本机配置，运行时字段以安装版本为准。

## 3. 识别模型路径

至少确认：

- Dense 或 MoE；总参数与每 token 激活参数；
- 文本、视觉、音频或多模态，是否存在独立 encoder；
- 权重容器与量化（例如 safetensors、GGUF、GPTQ/AWQ/EXL2 等）及其 kernel 实现；
- Attention/状态结构（MHA/GQA/MLA、滑窗、混合循环/SSM等）、上下文长度、KV/状态 dtype、RoPE/位置编码扩展；不要给所有模型套相同 KV 公式；
- 普通自回归、MTP、Draft/Verify 或其他 speculative 方法；
- 是否 CPU/GPU offload、分层加载、tensor/pipeline/expert parallel。

模型卡声明只作线索。以实际配置、权重元数据、加载日志和运行时报告为准。区分API模型别名、投机method入口、实际模型架构与加载权重；兼容入口名可能沿用旧实现，不能据此认定仍使用旧草稿。MoE 的激活参数量不能作为全部权重显存需求；AMD/Apple 等统一内存不能把“RAM”和“显存”当两份独立容量相加，按上节区分分配与共享口径。

## 4. 识别推理栈

按进程、包版本、启动参数和 API 行为识别实际 engine。常见路线仅用于分类：

- 服务型 GPU engine：vLLM、SGLang、TensorRT-LLM 或定制 runtime；
- 本地量化 runtime：llama.cpp、Ollama/LM Studio 背后的 engine、MLX、ExLlama 系列；
- 通用框架：Transformers/eager/compile、自定义 PyTorch/CUDA/ROCm；
- 上层：OpenAI-compatible API、反向代理、上下文压缩、Agent/Harness、桌面 UI。

参数、kernel 和 profiler 方法必须按现场版本核对。不能因为 API 兼容就假定底层 engine 相同。

首次选引擎时，把“硬件 + 模型/量化 + 用途 + 用户入口”作为一个组合核验，而不是按显卡品牌直接指定框架。沿以下顺序收敛：

1. 现有引擎满足必需能力时优先复用；缺能力时指出具体差距，并查其他仓库是否有可接入部件，不为采用教程的统一环境而迁移。主选可以是宿主加来源组件的完整组合，无须强求单一仓库包办全部能力。
2. 不支持当前 OS/设备/模型格式、所需模态或应用协议的路径不能作为已可直接部署的主选；按完整流程“未知主机的自主适配”分清具体缺口，再决定修复、实验或排除，不能从支持列表缺项推断永久不可实现。随后比较单用户首字/任务时间、并发需求、安装与维护成本。所需工具/结构化功能须核对客户端与后端的实际实现。
3. 对小白优先选择有可核验发布包、原生使用入口和匹配版本文档的候选；需要 nightly、特定 fork 或定制编译的路径标为实验路线。确有当前模型支持或性能依据时才考虑它，不为“最新”默认源码编译。
4. 给一个主选和必要时一个有具体触发条件的备选；不能同时安装多个引擎试运气。现有量化变体不能因不同引擎名字相同就假定可直接复用；格式转换须单独计入预算。

## 5. 选择主目标和护栏

| 用户感受/目标 | 主指标 | 必看护栏 |
|---|---|---|
| “启动后第一问慢” | 冷启动与模型加载后的首次请求分别测 | Decode、内存、正确性 |
| “长文开始回答慢” | 真实前缀命中/新处理量、图片/模板/复制、排队、首内容 | 冷输入单列；必要prefill计算按需最后优化 |
| “一直看不到答案” | 用户可见首字、客户端首内容 | engine TTFT、隐藏推理、输出质量 |
| “吐字慢” | ITL、单流 Decode TPS | TTFT、输出质量、功耗/温度 |
| “多人一用就慢” | 达标请求吞吐、排队 P95 | 总吞吐、单请求延迟、OOM、拒绝率 |
| “长文本跑不了” | 可用上下文、KV 容量 | TTFT、Decode、质量 |
| “应用整体慢” | 端到端 P50/P95 | engine、代理、工具循环各层时间 |
| “部署不稳定” | readiness、错误率、恢复时间 | 性能指标不能掩盖故障 |

完整部署优化时，阶段4先固定短任务整轮ms与传输等待，阶段5以长上下文速度为主线，阶段6再验操作者任务，正式A/B在阶段8。只要求解决表中某个问题时，直接进入相关诊断，复用可信基线并检查受影响功能，不扩成整套部署。

## 6. 条件分支

- 只有已确认使用 CUDA backend 才应用 CUDA/NCCL/CUDA Graph 特定参数；其他 backend 使用各自机制。兼容层可能沿用 `torch.cuda` 一类 API 名称，不能仅凭命名判定设备。
- 只有多 GPU/多机才分析 TP/PP/EP、P2P、collective 和通信重叠。
- MoE核对专家路由、激活参数、EP与负载；存在CPU/RAM/GPU专家卸载时，阶段3共同预算专家缓存和KV，阶段4必查显存专家命中/供给/传输等待，阶段5/6复查竞争。Dense offload也查搬运，但不伪造专家命中率。
- 只有多模态请求才测 encoder、视觉/音频 token 和跨模态路由；文本 Decode 加速不能代表 encoder 加速。
- speculative启用时核对完整Round和Draft；普通解码仍计完整step。前期固定草稿/K，接受数仅作解释；任务内K调优在阶段7，不能因硬件盘点就开始扫K。
- 涉及多设备协作或低位格式兼容时，读[对应检查](case-dual-2080ti-27b.md)；内核、shape、精度与资源参数按目标实现核对。
