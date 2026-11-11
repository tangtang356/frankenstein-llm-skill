# Frankenstein · Local LLM Deployment & Optimization Skill

![Frankenstein — Deploy / Optimize / Resume](docs/images/hero.png)

**Give your AI agent an engineering workflow to deploy local models and keep improving the ones you already run.**

English | [简体中文](README.zh-CN.md)

Frankenstein is an Agent Skill for hardware-aware model selection, deployment, inference optimization, and real-task validation. It connects research to actual runtime behavior: compute, data transfers, waiting, prefix reuse, and memory budgets.

This repository contains instructions, engineering case studies, and small test fixtures. It does not include an inference engine or model weights. The execution guides are currently in Chinese; this English page introduces the workflow.

## What you can use it for

| Starting point | What the agent does |
|---|---|
| A new machine | Inspect hardware, research compatible options, propose a deployment, and implement the agreed configuration |
| An existing deployment | Preserve the working setup, inspect what is actually running, identify bottlenecks, and build on verified improvements |
| One specific problem | Diagnose slow responses, decode performance, cache loss, transfer stalls, or a backend issue without restarting the entire workflow |
| A long optimization project | Resume from deployment notes and live process state, reusing completed work |
| A backend port | Implement and validate runtime, operators, and model components before optimizing the complete configuration |

The workflow covers local and SSH hosts, GPU and CPU inference, multi-GPU execution, and expert-offload configurations. Compatibility is checked on the target system.

## Start here

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

## Engineering cases

Developed through the author's deployment work since September 2026. These are historical results from particular configurations, not speed guarantees for other systems.

| Case | Reported result | Scope |
|---|---|---|
| 4× V100, Pi 27B NVFP4 + DSpine K7 | About **522 tokens/s** peak single-request decode; a separate follow-up measured about **523 tokens/s** | Short Python tasks, temperature 0, thinking off; excludes prefill and first-token latency |
| Dual 2080 Ti, IQ3 optimization path | Later Python measurements around **21–22 ms/round**, short best around **18 ms**, throughput peak **220+ tokens/s** | Different stages and workloads are documented separately; this is not one controlled before/after comparison |

- [V100: research, DSpine integration, runtime optimization, and cache repair](skills/local-llm-deployment-speedup/references/case-v100-pi-dspine.md)
- [Dual 2080 Ti: expert supply, kernels, pipeline, and state management](skills/local-llm-deployment-speedup/references/case-swift-iq3xxs-python.md)
- [Long-context attention: implementation workflow and five source-level repository cases](skills/local-llm-deployment-speedup/references/long-attention-optimization.md) — split-KV, reduction, KV layout, dispatch, CUDA Graphs, and runtime verification (Chinese).
- [Implementation methods organized by bottleneck](skills/local-llm-deployment-speedup/references/optimization-engineering.md)

Private runtime logs are not included. The cases explain conditions, measurements, and limitations; this repository is not a publicly reproducible benchmark suite. The V100 short-test case reports syntax checks rather than executed-function correctness, and does not establish full application acceptance.

## What a completed deployment includes

- The real application/API entry point and start, stop, and reuse instructions.
- The configuration and build actually loaded by the service.
- Performance observations with timing boundaries and cache conditions.
- Results from your chosen tasks, plus failures, untested items, and limitations.
- A deployment note that lets another agent continue without duplicating work.

The guides distinguish observed behavior from unavailable internal metrics. An inaccessible counter stays unverified; it does not become an invented cache hit or prevent unrelated useful work.

## Feedback

For a useful issue, include hardware, model and quantization, backend version, workload, timing definition, cache conditions, and the observed result. Use public or redacted examples. Reports from additional hardware and execution agents are especially valuable.

Optional weekly research can track relevant models, drafts, kernels, and cache improvements. It is enabled only when requested and does not automatically change deployments.

## License

[MIT](LICENSE). Models, runtimes, and third-party code retain their own licenses.
