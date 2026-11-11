# 本地模型部署与提速 Skill

面向让AI协助部署本地语言模型/视觉语言模型的使用者。核心是把不同仓库的成熟部件缝合起来：先找来源快的原因并追到源码，复用代码或移植其机制，补必要接口与硬件/模型实现，再验证完整部署。Strata＋1Cat＋Basalt＋Radiance体现这种组合方法，具体来源按任务选择；不默认重新大量研发。

覆盖GPU、CPU与专家卸载混合部署。验收先把短任务的完整轮次毫秒、传输和等待做好，再验证长上下文、前缀复用及真正的常用任务。

作者从2026年9月开始打磨本Skill，持续把本地部署中的方法整理为定位、实现与验证步骤。

作者推荐由 **GPT6 ASTRA / KIMI K3.1 PRO / FABLE5** 等后端工程能力强的AI执行。执行者需要目标机终端/SSH、文件和代码/网页读取能力；本包不提供推理引擎、模型权重或账号凭据。推荐名称按作者写法保留，不是API ID或公开性能排名。

## 使用

把整个`local-llm-deployment-speedup`文件夹安装到所用AI工具的技能目录。按[Codex官方说明](https://learn.chatgpt.com/docs/build-skills)，用户级可放到`~/.agents/skills/local-llm-deployment-speedup`，项目级可放到项目的`.agents/skills/local-llm-deployment-speedup`；没有出现时重开会话。旧环境若已从`~/.codex/skills`加载，可沿用其有效配置，避免在多个扫描目录安装同名副本。其他工具按其技能加载方式读取 [SKILL.md](SKILL.md)。不要只复制入口而漏掉references和assets。

示例请求：

> 使用这个技能检查我的目标电脑。我希望部署本地模型，用于……。先盘点硬件、调研案例，给出主模型、草稿、精度、KV和底座方案，与我讨论确认后再实施。后续按技能顺序优化，我会提供常用任务。

尚未授权实施时，AI先和你讨论完整方案；已确认或已委托自主实施的范围不重复询问，先把主模型、草稿、KV/状态池、底座内核、所需视觉与专家缓存完整部署并核实生效；之后才开始短ms与传输等待，再长上下文/缓存、常用温度/高思考和你指定的任务，最后在该任务下调K并正式对标。Prefill只在需要时最后处理，始终不能破坏缓存。采用的GitHub仓库应点Star并保留来源。

调研按 [六步](references/research-and-weekly.md) 覆盖任务模型/量化/特别后训练、KV管理/快照/卸载、新草稿、内核和显卡专属算法，并尽量整理有依据的ms。可选仅增量调研或已授权的自动调研与优化；沿用已有决定，定时任务另按明确指令创建。全部优化后必须完成 [最终质量检查](references/final-quality.md)。

自动模式示例：

> 对【目标部署】自动调研并优化，沿用已有授权和限制，从原部署记录的未完成步骤继续。自行选择范围内候选并实施、验证和交付，不逐阶段询问是否继续。记录失败方向与下一步。正式切换按已约定范围执行。本次直接执行一轮，不创建定时任务。

详见[自动优化与跨轮接续](references/autonomous-optimization.md)。该流程不自带调度器，技能安装或更新不会启动优化；实际运行、Goal和日程按各自明确指令执行。

## 填写真实测试任务

在[SKILL.md末尾](SKILL.md#操作者填写真实测试任务)填写任务原文、材料/入口、必需成果与不能退化项。已有任务直接引用。执行AI保留可比基线，在最终配置完成相同任务并按相同标准检查；没有任务或尚未检查时不得宣布最终质量通过。填好的私人内容留在实际部署记录，不上传公共技能仓库。

## 包内容与边界

- [主流程](SKILL.md) 与按阶段读取的参考文件。
- [短代码/缓存/视觉样例](assets/short-tests.md)，包含本包自制的两张小PNG；不代表真实任务或全面质量验收。
- [两个部署案例](references/deployment-cases.md)：四卡V100的Pi／DSpine接入与续接，以及双2080Ti的IQ3优化路径；保留历史配置、步骤和测量口径。
- [症状路由](references/experience-cases.md)，按当前成本选择调研、代码修改与验收步骤。
- [供给、计算与调度推进](references/case-swift-iq3xxs-python.md)，按真实瓶颈组合优化并跨模型复用有效机制。
- [跨仓库缝合](references/cross-repository-composition.md)，先找快的部位与原因、追到源码，再选择宿主与来源、编写目标硬件适配并接通依赖，以完整请求收益裁决。
- [GitHub源码对照](references/github-runtime-comparison.md)，按当前瓶颈核对固定版本、调用链、分派及本地补丁差异，在授权内接续实施。
- [请求处理与图片优化](references/request-path-overhead.md)，从高缓存命中后的软件等待追到占位扫描、张量传递、重复渲染和图片特征持久复用。
- [毫秒实施细节](references/millisecond-case-lessons.md)，展开请求分支、准备/等待、形状覆盖、融合、事件与测量的实施步骤。
- [优化工程拆解](references/optimization-engineering.md)，按瓶颈讲解专家供给组合、设备适配、多卡流水线、状态轮换、续接和跨模型继承。
- [缓存破坏专项](references/cache-invalidation.md)、[十二类故障处理](references/cache-failure-cases.md)及[长思考与KV快照专项](references/reasoning-prefix-cache.md)，从思考/工具/图片往返追到共同恢复边界、依赖页和回收生命周期。
- [长注意力专项](references/long-attention-optimization.md)与[五个仓库源码入口](references/long-attention-cases.md)，覆盖分块、并行归约、KV布局、长度分派、Graph及生效验证。
- [Python基准任务](assets/python-round-baseline.md)，包含原始12函数题、温度0/关闭思考设置及请求计时对齐方法。
- [草稿接入与形状覆盖](references/case-v100-pi-dspine.md)，检查身份、桥接、分派、资源与生成续接。

运行日志、账号、机器地址、权重与个人任务数据留在实际部署目录，不写入本技能。本包包含沿用原项目的 [MIT许可证](LICENSE)；引用或移植其他仓库代码时仍按其各自许可证处理。本次包内为指南和自制测试素材，不包含外部引擎代码。
