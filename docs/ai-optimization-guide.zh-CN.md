# 怎样让AI优化本地部署

操作者确定用途、部署入口和质量要求；具备终端/SSH与代码能力的AI负责调研实现、追踪实际成本、修改和验证。优化可以涉及请求处理、调度、内核和缓存，不限于调整启动参数。

## 给AI一个能执行的任务

安装完整[技能目录](../skills/local-llm-deployment-speedup)，在[技能末尾](../skills/local-llm-deployment-speedup/SKILL.md#操作者填写真实测试任务)填写真正要完成的任务、输入材料、成果要求与不能退化项。已有部署可这样发起：

> 使用 local-llm-deployment-speedup 优化我在【部署目录或主机】上的模型。实际入口是【应用/API】，主要问题是【生成慢／每轮等待久／工具或图片之后重算】。真实任务和质量要求见【已填写任务或记录】。先核对运行中的模型、配置、代码和已有优化，再从完整轮次、真实生成续接和请求处理找瓶颈。调研相关仓库为什么快，复用或移植有效机制，完成范围内的修改和验证。保留已约定能力和质量基线，交付实际收益、未通过项与下一步。

新部署先按[主流程](../skills/local-llm-deployment-speedup/SKILL.md)盘点、选型和组装。已有授权与结果继续复用，不需要操作者替AI猜kernel、K或缓存池参数。

## 第一步：把“慢”拆成可以定位的问题

| 观察到的问题 | AI具体检查什么 | 实施入口 |
|---|---|---|
| 生成过程中持续慢 | 完整轮次里计算、供给、传输和等待的成本，实际分派是否进入快路径 | [毫秒主章](../skills/local-llm-deployment-speedup/references/millisecond-optimization.md)、[实施细节](../skills/local-llm-deployment-speedup/references/millisecond-case-lessons.md) |
| 工具返回或继续聊时重读旧内容 | 前后请求token首差、提交前沿、各状态组能否共同恢复 | [缓存诊断](../skills/local-llm-deployment-speedup/references/cache-invalidation.md)、[故障处理](../skills/local-llm-deployment-speedup/references/cache-failure-cases.md) |
| 长思考后续接特别慢 | 思考往返、原始生成token、检查点及滑窗依赖寿命 | [长思考排查](../skills/local-llm-deployment-speedup/references/reasoning-prefix-cache.md) |
| 高命中仍迟迟不出首内容 | 图片、渲染、分词、CPU张量传递、排队及计时边界 | [请求处理](../skills/local-llm-deployment-speedup/references/request-path-overhead.md) |
| 外部项目快但不支持本机 | 来源快因、局部接口、硬件专用实现与本机替代路径 | [跨仓库缝合](../skills/local-llm-deployment-speedup/references/cross-repository-composition.md) |

按当前症状选路径，不全部重跑。先查现成日志和真实请求，需要时补少量计时或token/状态追踪；内部指标取不到时说明限制，继续独立工作，不编造根因。

## 第二步：调研能解决当前成本的实现

将热点改写成机制、算子、格式和shape关键词，跨仓库、模型和设备寻找方法。沿函数名、依赖、fork、PR和测试追到原实现，锁定阅读版本。候选必须说明：减少什么工作，怎样映射到本机，哪里需要改写，新增转换/显存/同步成本是什么，以及第一步怎样验证。

不按Star或整仓库TPS直接选型。能复用的部件直接复用；没有本机版本时，保留算法/布局/调度机制，编写必要的设备实现与配套。详见[发现与筛选](../skills/local-llm-deployment-speedup/references/research-and-weekly.md#先发现有用的实现再收敛候选)。

## 第三步：让改动接入真实运行链

沿入口→准备→分派→kernel/Graph→状态提交→消费者实施。实际shape、量化尺度、工作区、stream/event和缓冲所有权一起处理；不能只抄函数或打开开关。先把必要依赖接成可运行组合，再做受影响的局部检查和短请求反馈。

用加载产物、日志或trace确认服务确实执行新代码，诊断关闭后再看正常轮时。局部kernel快了但新增准备或等待抵消收益，应继续处理已知依赖；组合完整仍无收益，就撤回该候选并记录原因。

## 第四步：把前缀续接当成真正的多轮问题

至少跟踪一次实际生成→工具或用户追加→下一请求。先比较上轮输入与生成token是否保留，再查Attention、Hybrid状态、滑窗和页映射是否在同一边界可恢复。token不一致先修模板/parser/身份；token完整但状态缺失就查提交、快照依赖与回收，不重复折腾文本。

图片特征复用和文本KV分别计量；新增工具、换图、历史编辑、会话切换按真实语义正确失效。冷请求、正常块边界尾算与故障重算分开。不能用重复同一prompt或删除历史代替这项验证。

## 第五步：以操作者的真实任务完成验收

保留可比配置、材料和初始项目状态，在最终实际入口运行同一任务。检查代码运行、工具执行、图片归属及必需成果，并记录从开始到可用结果的时间、返工和失败。速度、缓存计数和质量各有证据，任何一个不能替代其余项。

最终更快但质量不符合原标准，就修复或撤回相关改动。没有可比基线只能说明最终可用性；操作者要求亲自检查的部分保留真实产物待验，不由AI自评分代签。详见[任务填写区](../skills/local-llm-deployment-speedup/SKILL.md#操作者填写真实测试任务)和[最终质量](../skills/local-llm-deployment-speedup/references/final-quality.md)。

交付时写清真实入口、已加载版本、有效改动、速度/缓存/质量结论和下一步。把这些记录放在实际部署目录，下一次从它继续，不重装环境或从头重复调研。
