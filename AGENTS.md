# AGENTS.md — EchoSim.AI Reference Architecture

[English](#english) | [中文](#中文)

> **Scope of this document.** This is a documentation file, not software. It describes — at a high level — *how the EchoSim.AI agent interface is structured* and *how an AI agent would talk to it*. This repository **hosts no runnable service and ships no code**. The production scoring models, weighting schemes, simulation heuristics, and profile-inference rules that power [EchoSim.AI](https://echosim.ai) are **proprietary and intentionally not documented here**. The high-dimensional neural agents ("AI Echoes") that run in EchoSim.AI's production environment are a separate, closed system. For the conceptual overview, see [`WHITEPAPER.md`](WHITEPAPER.md).

---

## English

### 1. What this describes

EchoSim.AI's agent interface is modeled as a [Model Context Protocol](https://modelcontextprotocol.io) surface: a small, stable set of dating-psychology capabilities that an MCP-capable agent (Claude Desktop, Cursor, Windsurf, custom SDK clients) could invoke locally. This document describes that *public interface layer* — not the engine behind it. No implementation is distributed in this repository.

### 2. Architecture at a glance

```
┌──────────────────────────────────────────────────────────┐
│  AI Agent / Host  (Claude Desktop · Cursor · SDK client)  │
└───────────────────────────┬──────────────────────────────┘
                            │  MCP requests (JSON-RPC)
                            ▼
┌──────────────────────────────────────────────────────────┐
│                      echo-sim-mcp                         │
│                                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │  Transport Layer        (STDIO ⇆ JSON-RPC)          │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Tool Registry          (schema + dispatch)         │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Tool Handlers  ── compatibility · simulation ·     │  │
│  │                     profiling   (black-box logic)   │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  Domain Model           (psychological profile型)   │  │
│  └────────────────────────────────────────────────────┘  │
│                                                           │
│  All state in-memory · no network · no persistence        │
└──────────────────────────────────────────────────────────┘
```

The internals of the **Tool Handlers** — the actual compatibility math, simulation scripting, and inference rules — are treated as a black box in this document. They are part of EchoSim.AI's proprietary IP.

### 3. Layers

| Layer | Responsibility | Public? |
| :--- | :--- | :--- |
| **Transport** | Establishes the STDIO MCP channel; serializes JSON-RPC; keeps the process resilient to malformed input. | ✅ Standard MCP |
| **Tool Registry** | Declares tool names + input JSON Schemas; routes each call to its handler; wraps errors into safe tool responses. | ✅ Interface only |
| **Tool Handlers** | Turn a validated request into a human-readable report. | ⚠️ Interface published, logic not |
| **Domain Model** | Defines the shared *shape* of a psychological profile (attachment style, MBTI, OCEAN, communication/conflict descriptors). | ✅ Schema only |

### 4. Agent-facing contract (the only part agents depend on)

An agent integrates with three tools. Only their **names and I/O contracts** are public; *how* each result is computed is not.

| Tool | Input (shape) | Output (shape) |
| :--- | :--- | :--- |
| `calculate_compatibility` | two psychological profiles | a textual diagnostic report |
| `simulate_micro_date` | two profiles + a scenario key | a textual simulated-dialogue transcript |
| `generate_psychological_profile` | free-text behavioral description | a structured profile object |

Agents should treat outputs as opaque, advisory text. They must **not** assume any particular scoring scale, formula, threshold, or weighting — those are internal and may change without notice.

### 5. Design principles

- **Local-first / zero-trust.** Everything runs in the host machine's memory. No cloud calls, no logging of inputs, no persistence. Sensitive relational data never leaves the device.
- **Deterministic & side-effect-free.** Given the same input, a handler returns the same output. No external services, no randomness in the public surface.
- **Stable interface, private engine.** The tool names and schemas are a long-term contract; the logic behind them is free to evolve and is not part of the public contract.
- **Fail safe.** Bad input yields a structured error response, never a crash — global handlers keep the STDIO pipe alive.

### 6. What is *not* in this repository

To be explicit, the following live only inside EchoSim.AI's production systems and are **not** published here:

- The production compatibility / fatigue scoring models and their weights.
- The high-dimensional "AI Echo" neural agents and their memory architecture.
- Background large-scale Echo-vs-Echo collision and ranking infrastructure.
- Any real user data, datasets, or trained parameters.

For the production platform, see **[https://echosim.ai](https://echosim.ai)**.

---

## 中文

> **本文档边界。** 本文件只描述 **服务的架构** 以及 **AI 智能体如何与之交互**——仅限架构层面。驱动 [EchoSim.AI](https://echosim.ai) 的生产级评分模型、权重方案、仿真启发式与画像推断规则均为 **专有技术，刻意不在此公开**。本开源仓库提供的只是一层轻量的本地参考接口；EchoSim.AI 生产环境中运行的高维神经替身（AI Echo）是一套独立的闭源系统。

### 1. 这个组件是什么

`echo-sim-mcp` 是一个 [Model Context Protocol](https://modelcontextprotocol.io) 服务端。它对外暴露一小组稳定的恋爱心理学工具，任何支持 MCP 的智能体（Claude Desktop、Cursor、Windsurf、自定义 SDK 客户端）都能通过 STDIO 在本地调用。它是 EchoSim.AI 生态的 **公开接口层**，而非背后的引擎。

### 2. 架构总览

```
┌──────────────────────────────────────────────────────────┐
│   AI 智能体 / 宿主  (Claude Desktop · Cursor · SDK 客户端)  │
└───────────────────────────┬──────────────────────────────┘
                            │  MCP 请求 (JSON-RPC)
                            ▼
┌──────────────────────────────────────────────────────────┐
│                      echo-sim-mcp                         │
│                                                           │
│  ┌────────────────────────────────────────────────────┐  │
│  │  传输层        (STDIO ⇆ JSON-RPC)                    │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  工具注册表    (Schema 声明 + 调用分发)               │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  工具处理器 ── 兼容性 · 约会仿真 · 画像生成            │  │
│  │                (核心逻辑为黑盒)                       │  │
│  ├────────────────────────────────────────────────────┤  │
│  │  领域模型      (心理画像数据结构)                     │  │
│  └────────────────────────────────────────────────────┘  │
│                                                           │
│  全程内存运行 · 无网络 · 无持久化                          │
└──────────────────────────────────────────────────────────┘
```

**工具处理器** 的内部——真正的兼容性算法、仿真剧本与推断规则——在本文档中均视为黑盒，属于 EchoSim.AI 的专有知识产权。

### 3. 分层职责

| 层 | 职责 | 是否公开 |
| :--- | :--- | :--- |
| **传输层** | 建立 STDIO 的 MCP 通道；序列化 JSON-RPC；对异常输入保持进程稳健。 | ✅ 标准 MCP |
| **工具注册表** | 声明工具名称与输入 JSON Schema；将调用路由到对应处理器；把错误包装成安全响应。 | ✅ 仅接口 |
| **工具处理器** | 将校验后的请求转化为可读报告。 | ⚠️ 仅公开接口，逻辑不公开 |
| **领域模型** | 定义心理画像的共享 **数据结构**（依恋类型、MBTI、OCEAN、沟通/冲突描述）。 | ✅ 仅 Schema |

### 4. 面向智能体的契约（智能体唯一依赖的部分）

智能体对接三个工具。只有它们的 **名称与输入输出契约** 是公开的；每个结果 **如何计算** 不公开。

| 工具 | 输入（结构） | 输出（结构） |
| :--- | :--- | :--- |
| `calculate_compatibility` | 两份心理画像 | 一份文本诊断报告 |
| `simulate_micro_date` | 两份画像 + 场景标识 | 一段文本仿真对话 |
| `generate_psychological_profile` | 自由文本行为描述 | 一个结构化画像对象 |

智能体应将输出视为不透明的参考性文本，**不应** 假设任何特定的评分量纲、公式、阈值或权重——这些均为内部实现，可能随时变更。

### 5. 设计原则

- **本地优先 / 零信任。** 一切都在宿主机内存中运行。无云端调用、不记录输入、不持久化，敏感关系数据永不离开设备。
- **确定性且无副作用。** 相同输入返回相同输出。公开接口不依赖外部服务，也不引入随机性。
- **接口稳定，引擎私有。** 工具名称与 Schema 是长期契约；其背后的逻辑可自由演进，不属于公开契约。
- **安全失败。** 异常输入返回结构化错误响应而非崩溃——全局处理器保证 STDIO 管道不中断。

### 6. 本仓库 **不包含** 的内容

明确声明，以下内容仅存在于 EchoSim.AI 的生产系统中，**不在** 此公开：

- 生产级兼容性 / 内耗评分模型及其权重。
- 高维「AI Echo」神经替身及其记忆架构。
- 后台大规模 Echo 互相碰撞与排序的基础设施。
- 任何真实用户数据、数据集或训练参数。

了解生产平台，请访问 **[https://echosim.ai](https://echosim.ai)**。
