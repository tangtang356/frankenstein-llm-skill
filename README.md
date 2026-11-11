# Frankenstein · Local LLM Deployment & Optimization Skill

![Frankenstein — Deploy / Optimize / Resume](docs/images/hero.png)

**Stitch together proven inference components. Adapt what is necessary. Verify the complete deployment.**

English | [简体中文](README.zh-CN.md)

Frankenstein is an Agent Skill centered on composing existing inference implementations across repositories. Combinations such as **Strata + 1Cat + Basalt + Radiance** express its core approach: select useful parts, connect their interfaces and dependencies, and adapt them to the target hardware and model. Find out which path is fast and why, inspect the source, then reuse the code or port its performance mechanism to the target device. Missing target-device support calls for a concrete implementation plan, not automatic rejection. Write the adaptations needed to preserve that mechanism while avoiding unrelated reinvention.

The workflow connects hardware-aware deployment and optimization to actual runtime behavior: compute, transfers, waiting, prefix reuse, memory budgets, and real-task results. The repositories in a combination depend on the workload; the named combination is not a universal recipe or a performance claim.

This repository contains instructions, implementation methods, and small test fixtures. It does not include an inference engine or model weights. The execution guides are currently in Chinese; this English page introduces the workflow.

## What you can use it for

| Starting point | What the agent does |
|---|---|
| A new machine | Inspect hardware, research compatible options, propose a deployment, and implement the agreed configuration |
| An existing deployment | Preserve the working setup, inspect what is actually running, identify bottlenecks, and build on verified improvements |
| One specific problem | Diagnose slow responses, decode performance, cache loss, transfer stalls, or a backend issue without restarting the entire workflow |
| A long optimization project | Resume from deployment notes and live process state, reusing completed work |
| A backend port | Reuse compatible runtime and operator implementations, bridge the missing interfaces, and develop only the remaining required pieces |

The workflow covers local and SSH hosts, GPU and CPU inference, multi-GPU execution, and expert-offload configurations. Compatibility is checked on the target system.

## Start here

Read [how to use AI for local inference optimization](docs/ai-optimization-guide.zh-CN.md) (Chinese) for a practical walkthrough covering identifying costs, tracing code, making changes, and checking real tasks. Detailed [request-path methods](skills/local-llm-deployment-speedup/references/request-path-overhead.md) cover high-cache-hit latency, CPU tensor transport, duplicate rendering, and persistent image features.

1. Download or clone this repository.
2. Install the complete [`skills/local-llm-deployment-speedup`](skills/local-llm-deployment-speedup) folder into your agent's supported skill directory. Keep its references, assets, and metadata together. Alternatively, ask the agent to read its [`SKILL.md`](skills/local-llm-deployment-speedup/SKILL.md) directly.
3. Give the agent access to the target terminal or SSH host, plus your model and usage requirements. It must be able to inspect files and code and perform the work you authorize.

**For a new deployment:**

> Use local-llm-deployment-speedup to inspect this machine and recommend a local model deployment for [my workload]. Explain the model, precision, memory budget, backend, and tradeoffs. Once we agree on the plan, deploy it, optimize it, and verify the actual application entry point.

**For a model you already run:**

> Use local-llm-deployment-speedup to continue optimizing my existing deployment at [path or host]. Read its deployment notes and inspect the active service first. Preserve accepted improvements and my configuration. Diagnose [slow decode / response latency / cache loss], implement relevant fixes, and verify the affected behavior.

## How optimization progresses

![Two entry points: a new machine or an existing deployment, followed by relevant measurement, optimization, and validation.](docs/images/workflow-en.svg)

**Inspect → research → agree on a plan → deploy the full configuration → optimize short-request execution → validate longer context and cache reuse → run your actual tasks → tune applicable speculative depth → compare final results → verify quality.**

Cold-prefill optimization is a separate, optional performance phase. Required prefill implementation and correctness fixes are part of deployment. Existing deployments reuse valid stage results; focused repairs do not require the whole sequence.

The agent checks complete execution costs, including transfers, synchronization, expert supply, and state updates. Prefix KV/hybrid-state reuse and GPU expert caching are measured separately. Actual generated-history continuation is tested, not just repeated identical prompts.

## Implementation guides

- [Find useful repositories and inspect why they are fast](skills/local-llm-deployment-speedup/references/research-and-weekly.md).
- [Compose components and port performance mechanisms](skills/local-llm-deployment-speedup/references/cross-repository-composition.md).
- [Reduce whole-round compute, transfer and waiting costs](skills/local-llm-deployment-speedup/references/millisecond-case-lessons.md).
- [Improve expert supply, device scheduling and state management](skills/local-llm-deployment-speedup/references/optimization-engineering.md).
- [Diagnose generated-prefix and hybrid-state reuse](skills/local-llm-deployment-speedup/references/cache-invalidation.md), including [long reasoning](skills/local-llm-deployment-speedup/references/reasoning-prefix-cache.md).
- [Optimize long attention](skills/local-llm-deployment-speedup/references/long-attention-optimization.md) and [request preparation](skills/local-llm-deployment-speedup/references/request-path-overhead.md).

The guides explain how to locate code, make changes and validate the running deployment. They are not a benchmark suite or a promise of a particular speed. Detailed instructions are in Chinese.

## Two deployment case studies

- [Four V100 GPUs: Pi 27B and DSpine](skills/local-llm-deployment-speedup/references/deployment-cases.md#案例一四卡-v100): projection fusion, draft integration, shape coverage and generated-prefix recovery. Historical temperature-zero, no-thinking short tests reached about 522 tok/s for a single request's full decode phase; a separate round-measurement run reached about 523 tok/s.
- [Dual RTX 2080 Ti: IQ3_XXS → IQ3_S → IQ3_XXS](skills/local-llm-deployment-speedup/references/deployment-cases.md#案例二双-2080-ti): expert supply, SM75 kernels, draft output heads, pipeline overlap and state rotation. Later temperature-zero, no-thinking Python tests recorded typical rounds of 21–22 ms, a brief minimum near 18 ms, and peak decode throughput above 220 tok/s as separate observations.

The case studies retain workload and timing boundaries. These are historical local records, not new benchmarks or upstream out-of-the-box results; private raw logs are not included. Detailed case studies are in Chinese.

## What a completed deployment includes

- The real application/API entry point and start, stop, and reuse instructions.
- The configuration and build actually loaded by the service.
- Performance observations with timing boundaries and cache conditions.
- Results from your chosen tasks, plus failures, untested items, and limitations.
- A deployment note that lets another agent continue without duplicating work.

The guides distinguish observed behavior from unavailable internal metrics. An inaccessible counter stays unverified; it does not become an invented cache hit or prevent unrelated useful work.

## Your real acceptance task

Fill in the [task section at the end of the skill](skills/local-llm-deployment-speedup/SKILL.md#操作者填写真实测试任务): the actual request, input materials, required deliverables, and quality requirements that must not regress. Keep the filled version in your deployment notes. The agent preserves a comparable baseline and checks the final configuration against the same task and criteria; speed cannot compensate for failed requirements.

## Feedback

For a useful issue, include hardware, model and quantization, backend version, workload, timing definition, cache conditions, and the observed result. Use public or redacted examples. Reports from additional hardware and execution agents are especially valuable.

Choose research-only updates or [authorized autonomous research and optimization](skills/local-llm-deployment-speedup/references/autonomous-optimization.md). In autonomous mode, the agent resumes deployment notes, selects candidates, implements and validates changes, and records what to do next without asking again for actions already authorized. Scheduling requires a separate explicit request; installing or updating this skill starts neither a scheduled task nor an optimization run.

## License

[MIT](LICENSE). Models, runtimes, and third-party code retain their own licenses.
