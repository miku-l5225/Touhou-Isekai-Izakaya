# 《东方异界食堂》技术设计文档 (Technical Design Document)

> 最后更新时间: 2025-12-25
> 状态: Production-Ready (Beta Phase 6)
> 关键词: Embodied AI, Multi-Agent, RAG, Web-based RPG

---

## 1. 项目概览 (Overview)

**《东方异界食堂》** 是一款基于《东方Project》世界观的 LLM 驱动型 Web 角色扮演与经营游戏。
本项目核心在于探索“具身智能 (Embodied AI)”在虚拟环境中的工程化落地，通过高度解耦的智能体架构实现一个拥有**感知 (Perception)、记忆 (Memory)、决策 (Decision)、执行 (Execution)** 完整闭环的智能系统。

### 1.1 核心设计哲学
*   **语义与数值对齐**：通过逻辑 Agent 将模糊的文学叙事转化为精确的数值状态。
*   **长效因果链**：利用基于 RAG 的记忆系统，使 NPC 的行为具备历史一致性。
*   **非入侵式交互**：在不中断玩家叙事流的前提下，实现复杂的系统触发（战斗、任务）。

---

## 2. 核心架构 (Core Architecture)

### 2.1 四重奏智能体协作模型 (The Quartet Architecture)

系统将每一轮交互拆解为四个独立运行的智能体，模拟人类大脑的功能分区：

| Agent | 角色 | 核心职责 | 输入 (Input) | 输出 (Output) |
| :--- | :--- | :--- | :--- | :--- |
| **LLM #1** | **Storyteller** | 叙事驱动 & NPC 扮演 | 玩家输入 + 历史记忆 | 剧情文本 + XML Triggers |
| **LLM #2** | **Game Master** | 逻辑解析 & 状态提取 | 剧情文本 + 游戏快照 | 结构化 JSON 指令 |
| **LLM #3** | **Scribe** | 记忆加工 & RAG 管理 | 原始对话 + 逻辑变更 | 结构化记忆快照 |
| **LLM #4** | **Misc/Tool** | 特化判定 (战斗/生成) | 任务特定上下文 | 判定结果/生成数据 |

### 2.2 逻辑处理流水线 (Logic Processing Pipeline)

为了解决大模型输出的非确定性（Non-determinism），`LogicService` 实现了工业级的鲁棒性方案：

#### 2.2.1 容错与重试策略 (Retry Strategy)
针对 API 失败或 JSON 语法错误，系统采用 **3 次指数退避重试 (Exponential Backoff)**：
```typescript
// 指数退避重试实现片段 (logic.ts)
for (let attempt = 1; attempt <= maxRetries; attempt++) {
  try {
    return await this._executeLogicRequest(userContent, storyContent, gameState);
  } catch (e: any) {
    const delay = attempt * 1000; // 1s, 2s, 3s...
    await new Promise(r => setTimeout(r, delay));
  }
}
```

#### 2.2.2 数据清洗 (Data Sanitization)
利用正则表达式修复 LLM 常见的格式缺陷，如：
- 移除多余的 Markdown 代码块标记 (` ```json `)。
- 修复非法转义字符和控制符。
- 自动平衡缺失的括号。

---

## 3. 记忆检索系统 (Scribe-Retrieval System)

项目采用 **"Agentic RAG"** 架构，旨在解决传统对话模型在长周期游戏中的“失忆”问题。

### 3.1 混合检索模型 (Hybrid Retrieval)

为了兼顾性能与深度，系统实现了两阶段过滤：

1.  **Level 1 粗筛 (Heuristic Filter)**：
    - **关键词匹配**：基于玩家输入的词云匹配。
    - **时间衰减 (Time Decay)**：$Score = \frac{1}{\sqrt{CurrentTurn - MemoryTurn}}$。
    - **重要性加权**：由 Scribe Agent 在记录时预先评估（1-5分）。
2.  **Level 2 精选 (Semantic Refinement)**：
    - 调用高智力模型对前 50 条粗筛结果进行语义评估，输出最终注入 Context Window 的记忆 IDs。

### 3.2 记忆分类 (Memory Schema)
| 类型 | 存储格式 | 检索触发条件 |
| :--- | :--- | :--- |
| **Summary** | 自然语言摘要 | 语义相关性/关键词 |
| **Facility** | 设施属性 JSON | 地点匹配/经营行为 |
| **Variable** | 数值变动 Diff | 经济行为/物品交互 |
| **Alliance** | 关系契约文本 | 关键角色入场 |

---

## 4. 具身智能与物理对齐 (Physical Alignment)

### 4.1 程序化地图生成 (Procedural Map Generation)
利用 LLM 生成空间语义，再由确定性算法完成物理填充。
- **语义层**：LLM 规划“厨房”、“储物间”、“用餐区”的逻辑连接。
- **物理层**：`ZonePopulator` 计算瓦片连通性，自动生成吧台、楼梯及装饰物。

### 4.2 战斗与交互逻辑 (Combat & Interaction)
- **非阻塞触发 (Non-blocking Triggers)**：系统检测到冲突信号后，在 UI 侧生成悬浮图标。玩家可自由决定介入时间，避免了传统阻塞弹窗对叙事体验的割裂。
- **嘴遁判定 (Persuasion Logic)**：战斗中实时调用 LLM #4 评估玩家文本输入的逻辑合理性与情感冲击力，从而动态修改敌方战意。

---

## 5. 数据模型设计 (Data Models)

```typescript
// 核心状态模型 (GameState)
interface GameState {
  player: PlayerStatus;        // 包含 hp, money, power, location 等
  npcs: Record<string, NPCStatus>; // NPC 好感度、衣着、心理、姿态
  system: GameSystemState;     // 包含 turn_count, active_quests, current_scene
  flags: Record<string, any>;  // 全局剧情开关
}

// 逻辑指令模型 (GameAction)
interface GameAction {
  type: 'UPDATE_PLAYER' | 'UPDATE_NPC' | 'INVENTORY' | 'SCENE';
  op: 'add' | 'subtract' | 'set' | 'push';
  field: string;
  value: any;
}
```

---

## 6. 技术栈 (Tech Stack)

*   **Runtime**: Node.js 18+ / Browser (Modern)
*   **Frontend**: Vue 3 + Vite + TypeScript
*   **State**: Pinia (持久化游戏核心状态)
*   **Database**: Dexie.js (IndexedDB 封装，用于本地存储万级记忆片段)
*   **Styling**: Tailwind CSS v4 (高性能和风 UI 方案)
*   **Audio**: Web Audio API (动态环境音效合成)

---

## 7. 未来路线图 (Roadmap)

- [x] **Quartet Architecture**: 四重 Agent 协作模型落地。
- [x] **Agentic RAG**: 实现基于 IndexedDB 的长效记忆系统。
- [x] **Embodied UI**: 非阻塞任务/战斗触发系统。
- [ ] **Multimodal Alignment**: 结合 Stable Diffusion 实现 NPC 形象的实时生成。
- [ ] **Autonomous Behavior**: 引入 NPC 的独立时间轴与自主寻路。
- [ ] **Multi-language Support**: 全球化叙事适配。

---
> *The boundary between Gensokyo and Reality is guarded by code.*
