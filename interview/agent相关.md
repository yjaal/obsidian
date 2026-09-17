
```toc
```

## 你们用的 Agent 框架是什么？ReAct 还是 Plan-and-Execute？

这里 ReAct 就是边想边做，每一步都是基于上一步的结果动态决策，而后一种是先制定计划，确定全部步骤，然后再执行。

两者在几个维度上的差异：

**第一，容错性**。ReAct 每步都能根据 Observation 调整，容错性强；Plan-and-Execute 一旦规划错了，后续全错，需要重新规划。

**第二，效率**。ReAct 每步都要调用 LLM 推理，Token 消耗和延迟都高；Plan-and-Execute 规划一次，执行阶段不需要频繁推理，效率更高。

**第三，适用场景**。ReAct 适合探索型、不确定型任务；Plan-and-Execute 适合确定性高、步骤明确的任务。



## 首次生成和多轮补充的链路路由是怎么区分和实现的 ？

这个问题切中了 Agent 系统设计的核心工程问题：**第一次请求和后续多轮请求，走的链路有什么不同？路由是怎么判断的？**

是想看你是否理解 **"有状态 Agent"和"无状态 API 调用"的本质区别**，以及能否设计出兼顾效率和体验的路由架构。

下面我从**链路差异、路由判断、工程实现**三个维度给你系统拆解。


### 一、首次生成 vs 多轮补充：链路的本质差异

| 维度 | 首次生成 | 多轮补充 |
|------|----------|----------|
| **触发条件** | 用户发起全新任务 | 用户对已有任务的追问/修正/补充 |
| **上下文状态** | 空白，需要从零构建 | 已有上下文，只需增量更新 |
| **检索策略** | 全量检索（RAG/工具/记忆） | 增量检索（只查缺失信息） |
| **规划策略** | 完整规划（Plan-and-Execute） | 局部调整（ReAct 式微调） |
| **缓存利用** | 无缓存，全新计算 | 可复用 Prefix Cache |
| **Token 消耗** | 高（完整上下文） | 低（增量上下文） |
| **典型耗时** | 长（3-10 秒） | 短（1-3 秒） |


### 二、路由判断：怎么区分首次和多轮？

核心是**状态检测**——判断当前请求是否属于已有会话的延续。

#### 判断逻辑

```python
class RequestRouter:
    def route(self, request):
        # 1. 检查是否有任务ID
        if not request.task_id:
            return "first_turn"  # 全新任务
        
        # 2. 检查会话状态
        task = session_store.get(request.task_id)
        if not task or task.status == "completed":
            return "first_turn"  # 会话不存在或已完成
        
        # 3. 检查用户意图：是新任务还是追问？
        intent = self.classify_intent(request.query, task.history)
        
        if intent == "new_task":
            return "first_turn"  # 用户开启了新任务
        elif intent == "follow_up":
            return "multi_turn"  # 追问/补充
        elif intent == "correction":
            return "multi_turn_correction"  # 修正
        elif intent == "clarification":
            return "multi_turn_clarification"  # 澄清
```

#### 意图分类的关键信号

| 信号 | 判断 | 路由 |
|------|------|------|
| 用户说"另外"、"还有一个问题" | 新任务 | first_turn |
| 用户说"不对"、"我是说" | 修正 | multi_turn_correction |
| 用户说"那...呢？"、"继续" | 追问 | multi_turn |
| 用户说"具体一点"、"展开说说" | 澄清 | multi_turn_clarification |
| Query 与历史高度相关 | 追问 | multi_turn |
| Query 与历史无关 | 新任务 | first_turn |


### 三、回答模板

**Q**：Agent 中首次生成和多轮补充的链路路由是怎么做的？
**你**：核心是**状态检测+意图分类**。

**第一步是状态检测**：检查请求是否携带任务 ID，以及 task是否处于活跃状态。如果没有任务 ID 或任务已完成，就是首次生成。这里要注意，一次会话中会存在多个 task，一个 task 包含多轮 QA。

**第二步是意图分类**：如果有活跃会话，用轻量级分类器判断用户意图——是新任务、追问、修正还是澄清。判断信号包括关键词（"另外"=新任务，"不对"=修正）、语义相似度（与历史高度相关=追问）、时间间隔等。

**第三步是路由分发**：
- **首次生成**走全量链路：完整上下文构建 + Plan-and-Execute 规划 + 全量检索，耗时较长但一次到位。
- **多轮补充**走增量链路：复用已有上下文 + 局部调整 + 增量检索，耗时短、响应快。
 
**关键工程点**：
1. 会话状态用 Redis 持久化，设置 TTL
2. 多轮链路复用 Prefix Cache，保持系统 Prompt 稳定
3. 意图分类用轻量级模型，避免增加延迟

**核心认知**：首次和多轮的本质区别是**信息增量**——首次需要从零构建，多轮只需要补充缺失部分。好的路由设计能让多轮响应速度提升 50%以上，同时保持上下文连贯性。



## Agent 的记忆怎么搞？长期短期分别怎么存？

Agent的记忆设计完全借鉴了这个原理。

|记忆类型|对应人类|容量|访问速度|生命周期|
|---|---|---|---|---|
|**工作记忆**|当前思考的内容|极小（几轮对话）|极快|当前Turn|
|**短期记忆**|今天发生的事|小（当前会话）|快|当前Session|
|**长期记忆**|人生经历|大（跨会话）|慢（需检索）|永久|


一般工作记忆直接放在 Prompt 中，不落盘。短期记忆 Redis + 摘要压缩。长期记忆存储于向量数据库和关系型数据库。


## Agent 的 skills 功能原理是什么？你是怎么设计和实现 skills 体系的？

- **Level 1：元数据 (Metadata)**：每个技能启动时，只有 `name` 和 `description` 会被预加载到系统提示中。这就像一本手册的**目录**，让 Agent 知道“有哪些技能可用”，消耗极低（约 30-50 tokens）。
    
- **Level 2：技能主体 (SKILL.md)**：当 Agent 判断某技能与任务相关时，才会读取完整的 `SKILL.md` 文件。这是“**操作手册的正文**”，包含详细步骤和注意事项，通常控制在 5k tokens 以内。
    
- **Level 3：附加资源 (Scripts/References)**：对于复杂场景，技能文件夹内可包含脚本或额外文档。只有在执行到特定步骤时，Agent 才会加载或运行它们。这是“**手册的附录或工具箱**”，内容量无硬性上限。

何时用比是什么功能更重要，职责要单一。


## 怎么构建提示词模板

### Prompt 模板的标准结构

一个生产级的 Prompt 模板，通常包含以下六个模块，按**稳定性从高到低**排列：

┌─────────────────────────────────────────────────────────────┐
│  1. 角色定义（Role）                                          │
│     - 你是谁？一个句子说清楚                                   │
│     - 示例："你是一个专业的航空客服助手"                         │
├─────────────────────────────────────────────────────────────┤
│  2. 核心约束（Constraints）                                   │
│     - 必须做什么、禁止做什么                                    │
│     - 每条约束必须可验证（能写出测试用例）                        │
│     - 示例："禁止编造航班信息"、"必须使用中文回答"                 │
├─────────────────────────────────────────────────────────────┤
│  3. 工作流（Workflow）                                        │
│     - 按步骤编号，流程线性或分支明确                             │
│     - 示例："Step 1: 理解意图 → Step 2: 检索信息 → Step 3: 回答"│
├─────────────────────────────────────────────────────────────┤
│  4. 工具使用规则（Tool Usage）                                 │
│     - 每个工具的使用条件、参数获取方式、错误处理                   │
│     - 格式：工具名 | 调用条件 | 必填参数 | 异常降级               │
├─────────────────────────────────────────────────────────────┤
│  5. 输出格式（Output Format）                                 │
│     - 必须遵守的输出格式（JSON/Markdown/纯文本）                 │
│     - 提供示例                                               │
├─────────────────────────────────────────────────────────────┤
│  6. 边界处理（Edge Cases）                                    │
│     - 用户输入不可识别/超长/恶意时的处理                         │
│     - 示例："如果无法回答，请说'我需要转接人工客服'"               │
└─────────────────────────────────────────────────────────────┘

同时我们还会对 prompt 进行拆分，因为有些部分是相同的，比如角色，相关约束等，如果都放在同一个文件，可能会超大，不利于管理。


### 设计原则

**职责分离**

| 内容类型           | 放在哪里      | 原因        |
| -------------- | --------- | --------- |
| 角色、约束、工作流      | Prompt 中  | 需要模型理解并遵循 |
| 权限校验、数据过滤      | 工具/API 层  | 不应依赖模型判断  |
| 动态信息（时间、Query） | Prompt 末尾 | 不污染缓存前缀   |

**核心认知**：自然语言用于“工作流程指导”，工具层负责“权限与数据”。

**可验证性**
每条约束都应该能写出对应的测试用例。

**最小充分性**
Prompt 不是越长越好。每个模块只保留**必要信息**，冗余内容会稀释注意力。

**渐进式披露**
复杂任务拆分成多个 Prompt，按需加载。


## 在上下文工程方面有哪些实践经验 ？

这里是想看你是否理解 **“把正确的信息，在正确的时间，以正确的格式，放到正确的位置”** 这一核心命题。

### 一、上下文组装：分层布局

#### 核心原则：稳定前置、动态后置

```
┌─────────────────────────────────────────────────────────────┐
│  区域1：绝对稳定区（Static Prefix）                         │
│  - 角色定义、核心约束、通用工作流                            │
│  - 输出格式模板                                             │
│  ⚡ 缓存命中率：最高（永不失效）                            │
├─────────────────────────────────────────────────────────────┤
│  区域2：半稳定区（Semi-Static Prefix）                      │
│  - 工具列表摘要（非完整描述）                               │
│  - Few-shot示例                                             │
│  ⚡ 缓存命中率：高（仅在工具变动时失效）                    │
├─────────────────────────────────────────────────────────────┤
│  区域3：动态区（Dynamic Suffix）                            │
│  - RAG检索结果                                              │
│  - 对话历史                                                 │
│  - 工具调用结果                                             │
│  - 用户Query                                                │
│  - 系统时间戳                                               │
│  ⚡ 缓存命中率：不依赖缓存（每次都是新的）                  │
└─────────────────────────────────────────────────────────────┘
```

**关键实践**：工具列表不直接写完整描述，而是用“懒加载引用”——Prompt 里只放工具名称摘要，完整参数说明在系统层按需注入，避免污染缓存前缀。

### 二、上下文压缩：保价值的有损压缩

#### 压缩策略矩阵

| 策略       | 做法             | 适用场景     | 压缩率    |
| -------- | -------------- | -------- | ------ |
| **摘要压缩** | 用 LLM 对历史对话做摘要 | 长对话      | 80-95% |
| **实体提取** | 只保留关键实体和事实     | 信息密集型对话  | 90-98% |
| **滑动窗口** | 只保留最近 N 轮      | 近因偏好型任务  | 50-80% |
| **分层记忆** | 短期全量+长期摘要+按需检索 | 复杂 Agent | 60-90% |

#### 压缩的质量保障

```python
class CompressedMemory:
    def __init__(self):
        self.summary = ""  # 压缩后的摘要
        self.source_chunks = []  # 压缩前的原始chunk
        self.chunk_map = {}  # 摘要句子→原始chunk的映射
    
    def compress(self, text, query):
        # 按信息价值密度排序
        scored_sentences = self.score_by_information_density(text)
        # 保留高价值句子
        self.summary = self.select_top_k(scored_sentences, query)
        # 记录来源映射，支持回溯验证
        self.chunk_map = self.build_mapping(self.summary, text)
        return self.summary
    
    def lookup(self, claim):
        # 需要验证时，反向查找原始来源
        source_id = self.chunk_map.get(claim)
        if source_id:
            return self.source_chunks[source_id]
        return None
```

**核心认知**：压缩的目标不是“尽可能小”，而是“在满足任务需求前提下的最小”。必须保留可回溯性，当 Agent 发现信息不足时能反向查找原始文本。


### 三、缓存优化：最大化 KV Cache 命中率

#### 关键策略

| 策略         | 做法                              | 效果             |
| ---------- | ------------------------------- | -------------- |
| **稳定前缀**   | 系统 Prompt 和工具摘要保持不变             | 缓存命中率 90%+     |
| **动态后置**   | 时间戳、Query、历史放在末尾                | 不污染前缀          |
| **工具列表外置** | 完整工具描述通过推理引擎层注入                 | 工具变动不影响前缀 Hash |
| **前缀缓存复用** | 使用 vLLM/SGLang 的 Prefix Caching | 减少重复计算         |

#### 上下文布局示例

```python
# 稳定前缀（缓存命中）
stable_prefix = """
你是航空客服助手。
核心约束：禁止编造航班信息，必须使用中文回答。
工作流：理解意图 → 检索信息 → 回答。
工具摘要：查询类(weather/flight/rate)，操作类(email/ticket/approval)
"""

# 动态后缀（不参与缓存）
dynamic_suffix = f"""
当前时间：{current_time}
用户问题：{query}
历史对话：{recent_messages}
检索结果：{rag_results}
"""
```


### 四、动态管理：按需检索与注入

#### 检索时机

| 时机 | 检索内容 | 检索层级 |
|------|----------|----------|
| 每次请求 | 最近 N 轮对话 | 工作记忆 |
| Task 开始时 | 当前 Session 的 Task 历史 | 短期记忆 |
| 需要用户信息时 | 用户偏好+历史经验 | 长期记忆 |
| 用户问“上次...” | 历史交互记录 | 长期记忆 |

#### 检索策略

```python
class ContextRetriever:
    def retrieve(self, query, session, user):
        results = []
        
        # 1. 工作记忆：直接返回最近N轮
        results.extend(session.working_memory.to_prompt())
        
        # 2. 短期记忆：按Task相关性检索
        if session.short_term_memory:
            relevant_tasks = session.short_term_memory.find_relevant(query)
            results.extend(relevant_tasks)
        
        # 3. 长期记忆：按语义相关性检索
        long_term = LongTermMemory(user.id)
        memories = long_term.recall(query, top_k=3)
        results.extend(memories)
        
        # 4. 排序和截断
        return self.rank_and_truncate(results, max_tokens=4000)
```


### 五、生产级实践经验

#### 经验 1：信息价值非均匀分布

文本中不是每句话都同等重要（一般涉及到相关业务的意图对应的语料价值更高）。用启发式评分函数给句子打分：

```python
def calculate_information_density(sentence):
    score = 0
    # 实体密度（专有名词、数字、日期）
    score += len(extract_entities(sentence)) * 2.0
    # 动词密度（动作越具体越有价值）
    score += len(extract_action_verbs(sentence)) * 1.5
    # 疑问句（通常包含用户意图）
    if is_question(sentence):
        score += 1.0
    return score
```

#### 经验 2：语义完整性保护

压缩单元必须是语义完整的单元——句子、段落、或对话轮次。不能把一句话拦腰截断。

| 压缩单元 | 优点 | 缺点 |
|----------|------|------|
| 按句子压缩 | 语义基本完整 | 可能丢失跨句逻辑 |
| 按段落压缩 | 语义完整度高 | 压缩率较低 |
| 按对话轮次压缩 | 保持对话结构 | 长轮次可能包含冗余 |

#### 经验 3：任务相关性过滤

信息的价值是相对于当前任务而言的。同样一段文本，用户问“价格”和“使用方法”，需要保留的信息完全不同。用用户 Query 做相关性检索，只保留与当前任务最相关的内容。

#### 经验 4：可回溯性设计

即使做了最好的压缩，也可能丢掉用户后续追问需要的细节。压缩机制必须保留“原始来源映射表”，每个压缩后的句子都能追溯到原始文本的 chunk ID。


### 六、回答模板

> **你**：我从四个维度做上下文工程：
>
> **第一是上下文组装**。核心原则是“稳定前置、动态后置”。把 Prompt 分为三个区域：绝对稳定区（角色、约束、工作流）、半稳定区（工具摘要、Few-shot 示例）、动态区（RAG 结果、对话历史、用户 Query）。稳定区作为缓存前缀，动态区放在末尾，最大化 KV Cache 命中率。
>
> **第二是上下文压缩**。采用“保价值的有损压缩”——按信息价值密度排序，保留高价值句子，同时记录来源映射支持回溯验证。压缩的目标不是“尽可能小”，而是“在满足任务需求前提下的最小”。
>
> **第三是缓存优化**。工具列表不直接写完整描述，而是用“懒加载引用”——Prompt 里只放工具名称摘要，完整参数说明在系统层按需注入，避免工具变动污染缓存前缀。
>
> **第四是动态管理**。采用三层记忆架构：工作记忆（最近 5 轮对话）、短期记忆（Session 内 Task 状态）、长期记忆（跨会话用户偏好）。每次请求按需检索，排序后截断到 4000 Token 以内。
>
> **核心认知**：上下文工程的关键不是“塞更多信息”，而是**“在正确的时间，把正确的信息，以正确的格式，放到正确的位置”**。这需要组装、压缩、缓存、动态管理四者协同。



## 有没有做过 todo list 这类优化，为什么它能让模型更聚焦？

> 我认为 Todo List 是 Agent 系统中最"性价比"最高的优化之一。
> 
> **它的核心原理是"把隐式规划变成显式状态"**。没有 Todo List 时，任务进度散落在对话历史中，模型需要同时关注"做什么"和"做到哪了"，注意力被分散。有了 Todo List 后，任务状态集中在一个结构化列表中，模型只需要关注"当前这一步怎么做"。
> 
> **具体效果有三个**：
> 
> 1. **注意力聚焦**：模型只关注当前🔄标记的任务，执行质量提升。
> 2. **防止循环**：已完成任务标记为✅，模型不会重复执行。
> 3. **支持断点续传**：中断后可以从 Todo List 中定位断点，继续执行。
> 
> 
> **实现上**，我用结构化任务列表，每个任务包含 id、内容、状态、结果。进阶版支持任务依赖图，确保按正确顺序执行。任务粒度控制在"可独立执行、可验证结果"的最小单元。
> 
> **生产级优化**上，我会做动态调整——执行结果超出预期时，动态插入新任务；执行失败时，标记失败并尝试替代方案。
> 
> **核心认知**：Todo List 的本质是**把"规划"和"执行"分离**——规划一次，执行时只关注当前步骤。这和人类用待办清单提高效率的原理完全一致。


## 查询改写

问：项目中有没有做过查询改写？多维度的查询改写具体是什么？当改写需要用户补充信息时，你是怎么设计交互和技术实现的？

多维度的查询改写，本质上是在**用户原始 Query 信息不完整、表述模糊、或与知识库术语不匹配**时，通过多个维度的转换，把它变成“可检索、可回答”的查询。当改写需要用户补充信息时，核心设计原则是：**能自动推断的不问用户，必须用户确认的才问，且问得精准、问得少。**

下面从**改写维度、交互设计、技术实现**三个层面系统拆解。

---

### 一、多维度的查询改写具体是什么？

查询改写不是单一操作，而是**多个维度的组合转换**。生产级系统通常覆盖以下六个维度：

| 维度 | 目标 | 示例 |
|------|------|------|
| **1. 同义扩展** | 解决术语不匹配 | “怎么退钱” → “退款流程” |
| **2. 指代消解** | 解决代词模糊 | “它多少钱” → “iPhone 15 多少钱” |
| **3. 意图澄清** | 解决意图模糊 | “帮我处理一下” → “帮我取消订单” |
| **4. 条件补全** | 解决参数缺失 | “订机票” → “订明天北京到上海的机票” |
| **5. 逻辑拆解** | 解决复合问题 | “A 和 B 哪个好” → 拆成“A 怎么样”“B 怎么样”“对比 A 和 B” |
| **6. 术语对齐** | 解决领域鸿沟 | “脑袋疼” → “头痛”（医学术语） |

**关键认知**：这六个维度不是独立使用的，而是**按需组合**。一个 Query 可能同时需要同义扩展+指代消解+条件补全。

---

### 二、改写需要用户补充信息时，怎么设计交互？

#### 核心原则：最小打扰

| 策略 | 做法 | 适用场景 |
|------|------|----------|
| **自动推断** | 从上下文/用户画像中提取 | 信息可从已有数据推断 |
| **默认值** | 用合理默认值填充 | 信息有行业惯例 |
| **选项确认** | 给出 2-3 个候选让用户选 | 信息有明确候选集 |
| **开放式追问** | 直接问用户 | 信息完全无法推断 |

#### 交互设计示例

**场景**：用户说“帮我订机票”，缺少出发地、目的地、日期。

**不好的交互**：
```
Agent：请问您的出发地是？
用户：深圳
Agent：请问您的目的地是？
用户：上海
Agent：请问您的出发日期是？
用户：明天
→ 三轮追问，用户体验差
```

**好的交互**：
```
Agent：好的，为您订机票。请确认以下信息：
- 出发地：深圳（根据您的定位推断）
- 目的地：上海（根据您上次的行程推断）
- 日期：明天（根据"明天"推断）
如果正确请确认，如需修改请告诉我。
→ 一次确认，用户体验好
```

**更好的交互**（带候选）：
```
Agent：好的，为您订机票。请选择：
1. 深圳 → 上海，明天
2. 深圳 → 北京，明天
3. 其他（请补充）
→ 用户只需点选，无需打字
```

#### 追问的粒度控制

```python
class ClarificationStrategy:
    def decide(self, query, context):
        missing_slots = self.find_missing_slots(query)
        
        for slot in missing_slots:
            # 1. 能否从上下文推断？
            inferred = self.infer_from_context(slot, context)
            if inferred:
                continue  # 不问，直接用推断值
            
            # 2. 是否有合理默认值？
            default = self.get_default_value(slot)
            if default:
                continue  # 不问，用默认值
            
            # 3. 是否有候选集？
            candidates = self.get_candidates(slot)
            if candidates:
                return self.ask_with_options(slot, candidates)
            
            # 4. 只能开放追问
            return self.ask_openly(slot)
```

---

### 三、生产级最佳实践

#### 实践 1：追问次数控制

```python
MAX_CLARIFICATION_ROUNDS = 2

def should_continue_clarifying(self, session):
    if session.clarification_rounds >= MAX_CLARIFICATION_ROUNDS:
        # 超过2轮追问，直接转人工
        return False
    return True
```

#### 实践 2：追问与推断的平衡

| 信息类型 | 策略 | 理由 |
|----------|------|------|
| 高风险参数（金额、日期） | 必须确认 | 错误代价高 |
| 低风险参数（座位偏好） | 可推断 | 错误代价低 |
| 有候选集的参数 | 选项确认 | 用户体验好 |
| 无候选集的参数 | 开放追问 | 只能问用户 |

#### 实践 3：改写效果评估

```python
def evaluate_rewrite(original_query, rewritten_query, expected_intent):
    # 1. 意图是否一致
    intent_match = classify(rewritten_query) == expected_intent
    
    # 2. 检索结果是否改善
    original_results = retrieve(original_query)
    rewritten_results = retrieve(rewritten_query)
    recall_improvement = len(rewritten_results) / len(original_results)
    
    # 3. 用户是否需要额外追问
    clarification_needed = has_missing_slots(rewritten_query)
    
    return {
        "intent_match": intent_match,
        "recall_improvement": recall_improvement,
        "clarification_needed": clarification_needed
    }
```

---

### 四、回答模板

> 查询改写有六个维度：同义扩展、指代消解、意图澄清、条件补全、逻辑拆解、术语对齐。生产级系统会按需组合这些维度。
>
> **当改写需要用户补充信息时，我的核心原则是“最小打扰”**：
>
> **第一，能推断的不问**。从用户画像、历史对话、当前上下文中推断缺失信息。比如用户说“订机票”，我从定位推断出发地，从上次行程推断目的地，从“明天”推断日期。
>
> **第二，有候选的让用户选**。如果缺失信息有明确候选集（如日期选项、航班选项），给出 2-3 个选项让用户点选，而不是打字。
>
> **第三，必须问的才问**。高风险参数（金额、日期）必须用户确认，低风险参数（座位偏好）可以用默认值。
>
> **第四，控制追问次数**。最多追问 2 轮，超过就转人工，避免用户体验恶化。
>
> **技术实现上**，我用五步流水线：意图识别→槽位提取→上下文推断→改写决策→改写输出。核心是槽位推断的置信度控制——置信度高于 0.8 直接使用，低于 0.8 才追问。
>
> **核心认知**：查询改写的目标不是“改写得多复杂”，而是**“让用户用最少的输入，得到最准确的回答”**。能自动推断的不问，必须确认的才问，且问得精准。



## 并行化意图识别是什么？为什么要做并行化？你是如何实现的？

并行化意图识别，简单说就是**同时用多个“专家”从不同角度判断用户意图，而不是让一个大模型从头想到尾**。它的核心价值在于：**把串行的“理解-判断-决策”链路，拆成并行的“多路独立判断+融合”链路，从而降低延迟、提升鲁棒性。**

下面从**是什么、为什么、怎么做**三个维度系统拆解。

### 一、并行化意图识别是什么？

#### 核心架构

```
用户Query
    ↓
┌─────────────────────────────────────────────────────────────┐
│  并行执行层（同时运行，互不依赖）                            │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ 规则匹配器   │  │ 小模型分类器 │    │ 向量检索器  │        │
│  │ (关键词/正则)│  │ (BERT)      │   │ (Embedding) │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │
│         └────────────────┼────────────────┘                │
│                          ↓                                 │
│              ┌───────────────────────┐                     │
│              │  结果融合器           │                     │
│              │  (投票/加权/优先级)   │                     │
│              └───────────┬───────────┘                     │
│                          ↓                                 │
│              ┌───────────────────────┐                     │
│              │  最终意图 + 置信度    │                     │
│              └───────────────────────┘                     │
└─────────────────────────────────────────────────────────────┘
```

#### 多路“专家”的分工

| 专家 | 判断依据 | 优势 | 局限 |
|------|----------|------|------|
| **规则匹配器** | 关键词、正则、模板 | 极快、100%可控 | 泛化差，只能覆盖已知模式 |
| **小模型分类器** | BERT 等轻量模型 | 快、准确率较高 | 需要训练数据，冷启动难 |
| **向量检索器** | Query Embedding + 意图库 | 泛化好，可处理新表述 | 依赖意图库质量 |
| **LLM 判断器** | 大模型推理 | 泛化最强，可处理复杂意图 | 慢、贵、不可控 |

**关键设计**：前三个是“快专家”，负责快速判断；LLM 是“慢专家”，只在快专家置信度低时才调用。我们目前也是单维检索+混合意图判断。


### 二、为什么要做并行化？

#### 原因 1：降低延迟

**串行方式**：
```
规则匹配（10ms）→ 小模型分类（50ms）→ LLM判断（500ms）
总延迟：560ms
```

**并行方式**：
```
规则匹配（10ms）┐
小模型分类（50ms）├→ 融合（5ms）→ 输出
向量检索（30ms）┘
总延迟：65ms（快专家）或 565ms（需要LLM时）
```

**效果**：80%的请求可以由快专家在 65 ms 内完成，只有 20%的复杂请求才需要调用 LLM。

#### 原因 2：提升鲁棒性

单一路径的意图识别有“单点故障”风险：

| 单一路径 | 风险 |
|----------|------|
| 只用 LLM | LLM 幻觉、超时、限流 → 整个系统不可用 |
| 只用规则 | 新表述、新意图 → 完全无法识别 |
| 只用小模型 | 训练数据未覆盖 → 判断错误 |

**并行多路**：任何一路失败，其他路仍可工作。规则匹配超时，向量检索仍能返回结果。

#### 原因 3：提升准确率

不同专家擅长不同场景：

```
用户Query: "我想退一下那个东西"

规则匹配器：命中"退" → 猜测"退货"（置信度0.6）
小模型分类器：判断"退货"（置信度0.7）
向量检索器：匹配到"退货流程"（置信度0.85）
LLM判断器：综合上下文 → "退货"（置信度0.95）

融合结果：退货（置信度0.85，多数投票）
```

**核心价值**：多路独立判断，通过投票/加权融合，降低单一路径的误判率。

#### 原因 4：成本优化

```
纯LLM方案：每次请求都调用LLM → 成本高
并行方案：80%请求由快专家处理 → 成本降低80%
```

#### 原因 5：可解释性

```
LLM判断：黑盒，不知道为什么判断为"退货"
并行方案：
  - 规则匹配：命中关键词"退"
  - 向量检索：相似度0.85
  → 可解释，便于调试和优化
```


### 三、生产级最佳实践

#### 实践 1：快慢分离

```
快路径（80%请求）：规则 + 小模型 + 向量 → 65ms
慢路径（20%请求）：快路径 + LLM → 565ms
```

#### 实践 2：缓存高频意图

#### 实践 3：持续优化

```python
# 记录低置信度的Query，用于优化规则和训练数据
def log_low_confidence(query, result):
    if result.confidence < 0.7:
        low_confidence_log.append({
            "query": query,
            "result": result,
            "timestamp": now()
        })
        # 定期人工审核，补充规则或训练数据
```


### 四、回答模板

> 并行化意图识别是**同时用多个“专家”从不同角度判断用户意图**，然后融合结果。这些专家包括规则匹配器、小模型分类器、向量检索器，必要时才调用 LLM。
>
> **为什么要并行化？有四个原因**：
>
> **第一是降低延迟**。串行方式是“规则→小模型→LLM”依次执行，总延迟 560 ms。并行方式下，三个快专家同时跑，65 ms 就能出结果，只有 20%的复杂请求才需要调用 LLM。
>
> **第二是提升鲁棒性**。单一路径有单点故障风险——LLM 超时、规则覆盖不全、小模型训练数据不足。并行多路，任何一路失败其他路仍能工作。
>
> **第三是提升准确率**。不同专家擅长不同场景，通过投票/加权融合，降低单一路径的误判率。比如规则命中“退”置信度 0.6，向量检索匹配“退货流程”置信度 0.85，融合后置信度 0.85。
>
> **第四是成本优化**。80%的请求由快专家处理，只有 20%需要调用 LLM，成本降低 80%。
>
> **技术实现上**，用 asyncio.gather 并行执行多个专家，然后按投票或加权融合。置信度高于 0.9 直接执行，0.7-0.9 调用 LLM 确认，低于 0.7 让用户澄清。
>
> **核心认知**：并行化意图识别的本质是**“用多个廉价专家的共识，替代单个昂贵专家的判断”**——既快又准，还便宜。


## 怎么让模型老老实实调用工具 ，不瞎编参数？

核心思路是——**不要给它编的机会，同时让它编的代价变高**。下面从**约束设计、参数校验、反馈闭环、工程兜底**四个层面系统拆解。

### 一、为什么模型会瞎编参数？

| 原因 | 表现 | 本质 |
|------|------|------|
| **参数缺失** | 用户没说日期，模型自己填了“明天” | 模型倾向于“补全”而非“追问” |
| **格式不匹配** | 日期格式要求 `YYYY-MM-DD`，模型给了“下周三” | 模型不知道严格的格式要求 |
| **枚举值越界** | 舱位只允许 `经济/商务/头等`，模型填了“豪华” | 模型不知道枚举范围 |
| **实体幻觉** | 用户说“订去上海的票”，模型填了“北京→上海” | 模型用先验知识填补了缺失信息 |
| **类型错误** | 金额要求数字，模型给了“一百块” | 模型不理解类型约束 |

**核心认知**：模型不是“故意”瞎编，而是它的训练目标就是“生成最可能的下一个 Token”，而不是“严格遵循参数约束”。

### 二、约束设计：让模型“没有编的机会”

#### 1. 工具定义要“窄而严”

```python
# 不好的定义（模型自由发挥空间大）
{
    "name": "book_flight",
    "description": "订机票",
    "parameters": {
        "from": {"type": "string"},
        "to": {"type": "string"},
        "date": {"type": "string"}
    }
}

# 好的定义（约束明确）
{
    "name": "book_flight",
    "description": "订机票。必须从用户明确提供的信息中提取参数，禁止推断或编造。",
    "parameters": {
        "from": {
            "type": "string",
            "description": "出发城市。必须是用户明确说出的城市名，禁止推断",
            "enum": ["北京", "上海", "深圳", "广州", ...]  # 枚举限定
        },
        "to": {
            "type": "string",
            "description": "目的城市。必须是用户明确说出的城市名，禁止推断",
            "enum": ["北京", "上海", "深圳", "广州", ...]
        },
        "date": {
            "type": "string",
            "description": "出发日期，格式YYYY-MM-DD。必须是用户明确说出的日期，禁止推断",
            "pattern": "^\\d{4}-\\d{2}-\\d{2}$"  # 正则约束
        }
    },
    "required": ["from", "to", "date"]
}
```

**关键设计**：
- `enum` 限定取值范围，模型只能在范围内选
- `pattern` 约束格式，不符合格式的直接报错
- `description` 中明确写“禁止推断/编造”
- `required` 标记必填参数，缺失时必须追问

#### 2. System Prompt 中明确约束

```
## 工具调用规则
1. 调用工具前，必须确认所有必填参数都已从用户输入中明确获取。
2. 禁止推断、猜测、或编造任何参数值。
3. 如果参数缺失，必须先向用户追问，不得直接调用工具。
4. 如果用户提供的信息模糊（如“大概下周”），必须先澄清具体日期。
5. 参数值必须严格符合工具定义的格式和枚举范围。
```

#### 3. 强制“先确认再调用”

```python
# 在Prompt中要求模型先输出参数确认
"""
调用工具前，必须先输出以下格式：
【参数确认】
- from: 深圳（来源：用户明确说“从深圳出发”）
- to: 上海（来源：用户明确说“去上海”）
- date: 2026-09-18（来源：用户说“明天”，当前日期2026-09-17）

确认无误后，再调用工具。
"""
```


### 三、参数校验：让模型“编了也白编”

#### 1. 调用前校验

```python
class ToolCallValidator:
    def validate(self, tool_name, params):
        tool_def = self.get_tool_definition(tool_name)
        errors = []
        
        for param_name, param_def in tool_def["parameters"].items():
            # 1. 必填校验
            if param_def.get("required") and param_name not in params:
                errors.append(f"缺少必填参数：{param_name}")
                continue
            
            if param_name not in params:
                continue
            
            value = params[param_name]
            
            # 2. 类型校验
            if not self.check_type(value, param_def["type"]):
                errors.append(f"参数{param_name}类型错误：期望{param_def['type']}，实际{type(value)}")
            
            # 3. 枚举校验
            if "enum" in param_def and value not in param_def["enum"]:
                errors.append(f"参数{param_name}取值越界：{value}不在{param_def['enum']}中")
            
            # 4. 格式校验
            if "pattern" in param_def and not re.match(param_def["pattern"], str(value)):
                errors.append(f"参数{param_name}格式错误：{value}不符合{param_def['pattern']}")
            
            # 5. 范围校验
            if "minimum" in param_def and value < param_def["minimum"]:
                errors.append(f"参数{param_name}小于最小值{param_def['minimum']}")
        
        return errors
```

#### 2. 校验失败后的处理

```python
def handle_validation_failure(errors, tool_name, params):
    # 方案1：返回错误给模型，让它重新生成
    error_message = f"""
    工具调用失败，参数校验错误：
    {chr(10).join(errors)}
    
    请重新生成参数，或向用户追问缺失信息。
    """
    return llm.regenerate(error_message)
    
    # 方案2：直接追问用户
    # return ask_user_for_missing_params(errors)
    
    # 方案3：降级到人工
    # return escalate_to_human(errors)
```

#### 3. 来源追溯

```python
# 要求模型标注每个参数的来源
def extract_with_source(query):
    """
    用户输入："帮我订明天从深圳到上海的机票"
    
    模型输出：
    {
        "from": {"value": "深圳", "source": "用户明确说'从深圳'"},
        "to": {"value": "上海", "source": "用户明确说'到上海'"},
        "date": {"value": "2026-09-18", "source": "用户说'明天'，当前日期2026-09-17"}
    }
    """
    # 校验：source不能为空，且必须能在用户输入中找到依据
    for param, data in extracted.items():
        if not data["source"]:
            raise ValidationError(f"参数{param}缺少来源标注")
        if not verify_source_in_query(data["source"], query):
            raise ValidationError(f"参数{param}的来源无法在用户输入中验证")
```

### 四、反馈闭环：让模型“越用越老实”

#### 1. 记录幻觉案例

```python
class HallucinationTracker:
    def log(self, tool_name, params, error_type, query):
        self.db.insert({
            "tool_name": tool_name,
            "params": params,
            "error_type": error_type,  # "missing", "wrong_value", "format_error"
            "query": query,
            "timestamp": now()
        })
    
    def analyze(self):
        # 统计高频幻觉类型
        stats = self.db.aggregate([
            {"$group": {"_id": "$error_type", "count": {"$sum": 1}}},
            {"$sort": {"count": -1}}
        ])
        return stats
```

#### 2. 针对性优化

| 高频幻觉类型 | 优化措施 |
|-------------|----------|
| 参数缺失时编造 | 在 Prompt 中强调“缺失必须追问” |
| 日期格式错误 | 在工具定义中增加格式示例 |
| 枚举值越界 | 在工具定义中列出完整枚举 |
| 实体幻觉 | 增加“来源标注”要求 |

#### 3. Few-shot 示例

```
## 正确示例
用户：帮我订明天从深圳到上海的机票
正确：参数完整，直接调用
{"from": "深圳", "to": "上海", "date": "2026-09-18"}

## 错误示例（禁止）
用户：帮我订机票
错误：编造参数
{"from": "北京", "to": "上海", "date": "2026-09-18"}  ← 禁止！用户没说出发地

正确：追问用户
"请问您从哪个城市出发？"
```

### 五、工程兜底：最后的防线

#### 1. 关键参数二次确认

```python
# 高风险参数（金额、日期、账号）必须用户确认
HIGH_RISK_PARAMS = ["amount", "date", "account_id"]

def execute_with_confirmation(tool_name, params):
    high_risk_values = {
        k: v for k, v in params.items() 
        if k in HIGH_RISK_PARAMS
    }
    
    if high_risk_values:
        confirmation = ask_user(
            f"请确认以下信息：{high_risk_values}"
        )
        if not confirmation.approved:
            return "用户取消"
    
    return execute_tool(tool_name, params)
```

#### 2. 参数默认值兜底

```python
# 对于有合理默认值的参数，用默认值而不是让模型编
def fill_defaults(params, tool_def):
    for param_name, param_def in tool_def["parameters"].items():
        if param_name not in params and "default" in param_def:
            params[param_name] = param_def["default"]
    return params
```

#### 3. 调用失败重试

```python
def call_with_retry(tool_name, params, max_retries=2):
    for attempt in range(max_retries):
        try:
            return execute_tool(tool_name, params)
        except ValidationError as e:
            if attempt == max_retries - 1:
                return escalate_to_human(e)
            # 让模型根据错误重新生成
            params = llm.regenerate_params(tool_name, params, e)
```


### 六、回答模板

> **核心思路是** “不给它编的机会，同时让它编的代价变高”**。我从四个层面设计：
>
> **第一是约束设计** 。 工具定义要“窄而严”——用 enum 限定取值范围，用 pattern 约束格式，在 description 中明确写“禁止推断或编造”，在 System Prompt 中强调“缺失参数必须追问”。
>
> **第二是参数校验**。调用前做五层校验：必填校验、类型校验、枚举校验、格式校验、范围校验。校验失败就返回错误给模型重新生成，或者直接追问用户。
>
> **第三是来源追溯**。要求模型标注每个参数的来源，比如“深圳”来自“用户明确说从深圳出发”。如果来源无法在用户输入中验证，就判定为幻觉。
>
> **第四是反馈闭环**。记录所有幻觉案例，统计高频错误类型，针对性优化 Prompt 和工具定义。比如发现日期格式错误多，就在工具定义中增加格式示例。
>
> **工程兜底**上，高风险参数（金额、日期）必须用户二次确认，调用失败自动重试，重试失败降级到人工。
>
> **核心认知**：模型不是“故意”瞎编，而是它的训练目标就是“生成最可能的 Token”。我们的任务是**用工程手段把它的生成空间约束在事实范围内**——让正确的调用容易，让错误的调用困难。



## 评测与 Badcase 定位

评测与 Badcase 定位是 AI 应用从“能用”到“好用”的核心工程环节。这里是想看你是否有**数据驱动迭代**的方法论，而不是靠“感觉”调 Prompt。

下面从**评测体系、Badcase 定位、归因分析、闭环优化**四个维度系统拆解。


### 一、评测体系：三层评测架构

#### 核心原则：离线评测保底线，在线评测看真实，人工评测抓体验

```
┌─────────────────────────────────────────────────────────────┐
│  L1：离线评测（Offline Evaluation）                         │
│  - 用固定测试集跑批量评测                                    │
│  - 指标：准确率、召回率、F1、BLEU、ROUGE                    │
│  - 频率：每次Prompt/模型变更后必跑                           │
│  - 优点：快、可复现、可对比                                  │
│  - 缺点：测试集可能不覆盖真实场景                            │
├─────────────────────────────────────────────────────────────┤
│  L2：在线评测（Online Evaluation）                          │
│  - A/B测试 + 用户行为埋点                                    │
│  - 指标：点击率、转化率、停留时长、追问率                    │
│  - 频率：持续运行                                            │
│  - 优点：真实反映用户满意度                                  │
│  - 缺点：周期长、需要流量支撑                                │
├─────────────────────────────────────────────────────────────┤
│  L3：人工评测（Human Evaluation）                           │
│  - 专家标注 + 用户反馈                                      │
│  - 指标：相关性、准确性、完整性、有用性                      │
│  - 频率：每周抽样                                            │
│  - 优点：能发现机器指标无法捕捉的问题                        │
│  - 缺点：成本高、主观性强                                    │
└─────────────────────────────────────────────────────────────┘
```


### 二、离线评测：怎么做才有效？

#### 1. 测试集构建

| 测试集类型 | 来源 | 规模 | 用途 |
|-----------|------|------|------|
| **黄金集** | 人工精选+标注 | 200-500 条 | 核心指标，每次必跑 |
| **回归集** | 历史 Badcase | 持续积累 | 防止修复后复发 |
| **边界集** | 极端/异常输入 | 100-200 条 | 测试鲁棒性 |
| **真实集** | 线上抽样 | 1000+条 | 反映真实分布 |

#### 2. 评测指标

```python
class OfflineEvaluator:
    def evaluate(self, test_set, agent):
        results = {
            "accuracy": 0,      # 答案正确率
            "recall": 0,        # 检索召回率
            "faithfulness": 0,  # 答案是否忠于检索结果
            "completeness": 0,  # 信息完整度
            "latency": 0,       # 平均延迟
            "token_cost": 0     # 平均Token消耗
        }
        
        for case in test_set:
            response = agent.run(case.query)
            
            # 1. 准确性：答案是否包含关键信息
            results["accuracy"] += self.check_accuracy(
                response, case.expected_answer
            )
            
            # 2. 忠实性：答案是否基于检索结果（而非编造）
            results["faithfulness"] += self.check_faithfulness(
                response, case.retrieved_docs
            )
            
            # 3. 完整性：是否覆盖所有关键点
            results["completeness"] += self.check_completeness(
                response, case.expected_key_points
            )
        
        return {k: v / len(test_set) for k, v in results.items()}
```

#### 3. LLM-as-a-Judge

用 GPT-4 等强模型做自动评测：

```python
def llm_judge(query, response, reference):
    prompt = f"""
    请评估以下回答的质量：
    
    用户问题：{query}
    参考答案：{reference}
    模型回答：{response}
    
    请从以下维度打分（1-5分）：
    1. 准确性：回答是否正确
    2. 相关性：回答是否切题
    3. 完整性：是否覆盖所有要点
    4. 流畅性：语言是否自然
    
    输出JSON格式：{{"accuracy": 4, "relevance": 5, ...}}
    """
    return llm.generate(prompt)
```

**注意**：LLM-as-a-Judge 需要与人工评测做校准，确保一致性。


### 三、Badcase 定位：从现象到根因

#### 1. Badcase 分类

| 类型 | 表现 | 根因层 |
|------|------|--------|
| **检索失败** | 没找到相关文档 | RAG 检索层 |
| **检索噪声** | 找到了不相关文档 | RAG 检索层 |
| **生成幻觉** | 编造不存在的信息 | LLM 生成层 |
| **指令违背** | 没遵循格式/约束 | Prompt 层 |
| **意图误解** | 理解错了用户意图 | 意图识别层 |
| **工具误用** | 调用了错误的工具 | 工具调用层 |
| **上下文丢失** | 忘记了之前的信息 | 记忆管理层 |

#### 2. Badcase 定位流程

```
Badcase发现
    ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 1: 现象记录                                            │
│  - 用户Query、Agent响应、期望响应                            │
│  - 完整执行链路（Thought/Action/Observation）                │
├─────────────────────────────────────────────────────────────┤
│  Step 2: 分层归因                                            │
│  - 检索层：召回文档是否相关？                                │
│  - 生成层：答案是否基于检索结果？                            │
│  - 意图层：意图识别是否正确？                                │
│  - 工具层：工具调用是否正确？                                │
├─────────────────────────────────────────────────────────────┤
│  Step 3: 根因定位                                            │
│  - 用消融实验定位：去掉某模块，问题是否复现？                │
│  - 用对比实验定位：换一个模型/Prompt，问题是否消失？         │
├─────────────────────────────────────────────────────────────┤
│  Step 4: 修复验证                                            │
│  - 针对性修复（改Prompt/换模型/优化检索）                    │
│  - 回归测试：确保修复不引入新问题                            │
└─────────────────────────────────────────────────────────────┘
```

#### 3. 定位工具

```python
class BadcaseAnalyzer:
    def analyze(self, badcase):
        # 1. 检索层分析
        retrieval_analysis = self.analyze_retrieval(
            badcase.query, 
            badcase.retrieved_docs,
            badcase.expected_docs
        )
        
        # 2. 生成层分析
        generation_analysis = self.analyze_generation(
            badcase.response,
            badcase.retrieved_docs
        )
        
        # 3. 意图层分析
        intent_analysis = self.analyze_intent(
            badcase.query,
            badcase.detected_intent
        )
        
        # 4. 综合归因
        return {
            "retrieval": retrieval_analysis,  # {"recall": 0.3, "precision": 0.5}
            "generation": generation_analysis, # {"faithfulness": 0.4, "hallucination": True}
            "intent": intent_analysis,         # {"correct": False, "predicted": "退货", "expected": "换货"}
            "root_cause": self.determine_root_cause(...)
        }
```

#### 4. 消融实验定位根因

```python
def ablate_and_locate(badcase):
    # 实验1：换更强的模型，问题是否消失？
    if run_with_gpt4(badcase.query) == badcase.expected:
        return "模型能力不足"
    
    # 实验2：换更精准的检索，问题是否消失？
    if run_with_perfect_retrieval(badcase.query) == badcase.expected:
        return "检索质量问题"
    
    # 实验3：换更清晰的Prompt，问题是否消失？
    if run_with_optimized_prompt(badcase.query) == badcase.expected:
        return "Prompt表述不清"
    
    # 实验4：手动给出正确意图，问题是否消失？
    if run_with_correct_intent(badcase.query) == badcase.expected:
        return "意图识别错误"
    
    return "多因素综合"
```


### 四、闭环优化：从 Badcase 到改进

#### 1. Badcase 驱动的迭代流程

```
Badcase发现 → 归因分析 → 制定修复方案 → 实施修复 → 回归测试 → 上线验证
    ↑                                                              ↓
    └──────────────────── 持续监控 ←──────────────────────────────┘
```

#### 2. 修复优先级矩阵

| 影响范围 | 修复成本 | 优先级 |
|----------|----------|--------|
| 高（影响大量用户） | 低 | P 0，立即修复 |
| 高 | 高 | P 1，排期修复 |
| 低 | 低 | P 2，顺手修复 |
| 低 | 高 | P 3，暂不修复 |

#### 3. 回归测试集

```python
class RegressionTestSet:
    def __init__(self):
        self.cases = []  # 历史Badcase
    
    def add_badcase(self, badcase):
        self.cases.append({
            "query": badcase.query,
            "expected": badcase.expected,
            "root_cause": badcase.root_cause,
            "fixed_at": now()
        })
    
    def run_all(self, agent):
        failures = []
        for case in self.cases:
            response = agent.run(case.query)
            if not self.is_correct(response, case.expected):
                failures.append(case)
        return failures
```

**核心原则**：每个修复的 Badcase 都要加入回归集，确保**修复后不再复发**。


### 五、生产级最佳实践

#### 实践 1：全链路追踪

```python
class AgentTracer:
    def trace(self, request_id):
        return {
            "request_id": request_id,
            "query": ...,
            "intent": ...,
            "retrieved_docs": [...],
            "tool_calls": [...],
            "response": ...,
            "latency": ...,
            "token_cost": ...,
            "timestamp": ...
        }
```

**价值**：Badcase 发生时，可以完整回放整个执行链路，快速定位问题层。

#### 实践 2：在线监控告警

```python
# 监控指标
metrics = {
    "error_rate": 0.02,        # 错误率
    "p99_latency": 3.5,        # P99延迟
    "avg_token_cost": 1500,    # 平均Token消耗
    "user_feedback_negative": 0.05  # 负反馈率
}

# 告警规则
if metrics["error_rate"] > 0.05:
    alert("错误率超过5%")
if metrics["p99_latency"] > 5.0:
    alert("P99延迟超过5秒")
```

#### 实践 3：用户反馈闭环

```python
# 用户点赞/点踩
def handle_feedback(request_id, feedback):
    if feedback == "negative":
        # 自动记录为Badcase
        badcase = extract_badcase(request_id)
        badcase_queue.append(badcase)
        
        # 如果是高危场景，立即告警
        if badcase.is_high_risk():
            alert_team(badcase)
```


### 六、回答模板

> **我分评测和 Badcase 定位两部分。
>
> **评测体系是三层架构**：
> - **离线评测**用固定测试集跑批量评测，指标包括准确率、召回率、忠实性。每次 Prompt 或模型变更后必跑，保证不退化。
> - **在线评测**用 A/B 测试和用户行为埋点，指标包括点击率、转化率、追问率。
> - **人工评测**每周抽样，评估相关性、准确性、完整性，发现机器指标无法捕捉的问题。
>
> **Badcase 定位分四步**：
> 1. **现象记录**：完整保存执行链路（Thought/Action/Observation）。
> 2. **分层归因**：从检索层、生成层、意图层、工具层逐层排查。
> 3. **根因定位**：用消融实验——换更强模型问题消失→模型能力不足；换更精准检索问题消失→检索质量问题。
> 4. **修复验证**：针对性修复后，加入回归测试集，确保不复发。
>
> **闭环优化上**，每个修复的 Badcase 都加入回归集，每次上线前跑全量回归。同时做全链路追踪和在线监控告警，Badcase 发生时能快速回放整个链路。
>
> **核心认知**：评测不是“跑个分”，而是**数据驱动的迭代闭环**——发现 Badcase→归因→修复→回归→监控，持续循环。

## Agent 系统的整体效果怎么评估？在没有用户反馈的情况下，如何进行有效的抽检？

Agent 系统的效果评估，核心难点在于它不像传统模型有明确的标签，而是一个**多步决策过程**——最终结果对了，不代表中间步骤对；中间步骤对了，也不代表最终结果对。在没有用户反馈的情况下，评估必须靠**自动化指标+分层抽检+对抗性测试**三管齐下。

下面从**评估维度、自动抽检、分层人工抽检、无反馈场景的特殊设计**四个层面系统拆解。

### 一、Agent 效果评估的特殊性

| 维度 | 传统模型 | Agent 系统 |
|------|----------|-----------|
| **评估对象** | 单次输出 | 多步决策链 |
| **正确性定义** | 输出=标签 | 结果正确+过程合理+效率可接受 |
| **失败模式** | 输出错误 | 检索错、工具选错、参数错、循环、超时 |
| **反馈来源** | 明确标签 | 用户反馈稀缺，需主动构造 |

**核心认知**：Agent 评估必须**结果与过程并重**——只看结果会忽略“运气好蒙对”的 case，只看过程会忽略“过程对但结果差”的情况。

### 二、四维评估体系

#### 维度 1：结果指标（Outcome）

| 指标 | 定义 | 计算方式 |
|------|------|----------|
| **任务完成率** | 成功完成用户目标的比例 | 成功数/总数 |
| **准确率** | 答案正确的比例 | 正确数/总数 |
| **完整度** | 覆盖所有要点的比例 | 覆盖要点数/总要点数 |
| **幻觉率** | 编造信息的比例 | 幻觉数/总数 |

#### 维度 2：过程指标（Process）

| 指标 | 定义 | 计算方式 |
|------|------|----------|
| **工具调用准确率** | 工具选择正确的比例 | 正确调用数/总调用数 |
| **参数正确率** | 参数填写正确的比例 | 正确参数数/总参数数 |
| **检索命中率** | 检索到相关文档的比例 | 命中数/检索数 |
| **步骤效率** | 是否走了最少步数 | 实际步数/最优步数 |
| **循环率** | 出现循环的比例 | 循环数/总数 |

#### 维度 3：效率指标（Efficiency）

| 指标 | 定义 | 计算方式 |
|------|------|----------|
| **延迟** | 端到端响应时间 | P 50/P 95/P 99 |
| **Token 消耗** | 平均 Token 使用量 | 总 Token/请求数 |
| **工具调用次数** | 平均调用工具次数 | 总调用数/请求数 |
| **重试率** | 需要重试的比例 | 重试数/总数 |

#### 维度 4：体验指标（Experience）

| 指标 | 定义 | 计算方式 |
|------|------|----------|
| **追问率** | 需要用户补充信息的比例 | 追问数/总数 |
| **澄清率** | 需要澄清意图的比例 | 澄清数/总数 |
| **放弃率** | 用户中途放弃的比例 | 放弃数/总数 |
| **负面反馈率** | 点踩/投诉的比例 | 负面数/总数 |


### 三、无用户反馈时，如何有效抽检？

#### 策略 1：自动抽检——用 LLM 做裁判

```python
class AutoInspector:
    def __init__(self):
        self.judge = LLM("gpt-4")
    
    def inspect(self, trace):
        """对单条执行链路做自动评估"""
        # 1. 结果评估
        result_score = self.judge.evaluate(f"""
            用户问题：{trace.query}
            Agent回答：{trace.response}
            检索文档：{trace.retrieved_docs}
            
            请评估：
            1. 答案是否正确（1-5分）
            2. 答案是否基于检索文档（1-5分，5=完全基于）
            3. 是否完整覆盖了用户需求（1-5分）
            
            输出JSON格式。
        """)
        
        # 2. 过程评估
        process_score = self.judge.evaluate(f"""
            Agent执行链路：{trace.steps}
            
            请评估：
            4. 工具选择是否合理（1-5分）
            5. 参数填写是否正确（1-5分）
            6. 是否有冗余步骤（1-5分，5=无冗余）
            7. 是否出现循环或重复（是/否）
        """)
        
        return {
            "result": result_score,
            "process": process_score,
            "overall": self.combine(result_score, process_score)
        }
```

**关键设计**：LLM-as-a-Judge 需要**与人工评测做校准**——先人工标注 100 条，对比 LLM 评分和人工评分的一致性，一致性达到 85%以上才可大规模使用。

#### 策略 2：分层抽检——按风险等级抽样

不同场景的抽检优先级不同：

| 风险等级 | 抽检比例 | 抽检方式 | 示例场景 |
|----------|----------|----------|----------|
| **高风险** | 100% | 人工全检 | 金融交易、医疗建议 |
| **中风险** | 20% | LLM 自动+人工复核 | 订单处理、退款 |
| **低风险** | 5% | LLM 自动 | 信息查询、闲聊 |

```python
def stratified_sampling(traces):
    samples = []
    for trace in traces:
        risk_level = classify_risk(trace)
        sample_rate = {
            "high": 1.0,
            "medium": 0.2,
            "low": 0.05
        }[risk_level]
        
        if random.random() < sample_rate:
            samples.append(trace)
    return samples
```

#### 策略 3：对抗性抽检——主动构造极端 Case

没有用户反馈时，主动构造“可能出错”的 Case 来测试：

```python
class AdversarialInspector:
    def generate_test_cases(self):
        return [
            # 1. 边界输入
            {"query": "", "expected": "拒绝或追问"},
            {"query": "a" * 10000, "expected": "截断或拒绝"},
            
            # 2. 矛盾输入
            {"query": "我要最便宜的，但必须头等舱", "expected": "指出矛盾"},
            
            # 3. 越权输入
            {"query": "帮我删除所有用户数据", "expected": "拒绝"},
            
            # 4. 模糊输入
            {"query": "帮我处理一下", "expected": "追问具体需求"},
            
            # 5. 多意图输入
            {"query": "我要退票，另外帮我查一下天气", "expected": "分别处理"},
            
            # 6. 历史依赖
            {"query": "就那个", "context": "之前聊过机票", "expected": "正确指代消解"},
        ]
```

#### 策略 4：影子模式——线上流量自动抽检

```python
class ShadowInspector:
    def __init__(self, agent, judge):
        self.agent = agent
        self.judge = judge
    
    def inspect_online_traffic(self, trace):
        """对线上流量做自动抽检"""
        # 1. 用LLM评估结果
        result_score = self.judge.evaluate_result(trace)
        
        # 2. 检测异常模式
        anomalies = self.detect_anomalies(trace)
        
        # 3. 如果分数低或检测到异常，加入人工审核队列
        if result_score < 3.0 or anomalies:
            human_review_queue.append(trace)
        
        return {
            "auto_score": result_score,
            "anomalies": anomalies,
            "need_human_review": result_score < 3.0 or bool(anomalies)
        }
    
    def detect_anomalies(self, trace):
        anomalies = []
        # 检测循环
        if trace.has_loop:
            anomalies.append("loop_detected")
        # 检测超长
        if trace.steps > 20:
            anomalies.append("too_many_steps")
        # 检测工具调用失败
        if trace.tool_failures > 2:
            anomalies.append("multiple_tool_failures")
        # 检测幻觉
        if not trace.response_based_on_retrieval:
            anomalies.append("potential_hallucination")
        return anomalies
```


### 四、无反馈场景的完整抽检流程

```
线上流量
    ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 1: 自动抽检（100%流量）                               │
│  - LLM-as-a-Judge评估结果和过程                              │
│  - 检测异常模式（循环、超长、工具失败、幻觉）                │
│  - 输出：自动评分 + 异常标记                                 │
├─────────────────────────────────────────────────────────────┤
│  Step 2: 分层抽样（按风险等级）                             │
│  - 高风险：100%进入人工审核队列                              │
│  - 中风险：20%进入人工审核队列                               │
│  - 低风险：5%进入人工审核队列                                │
├─────────────────────────────────────────────────────────────┤
│  Step 3: 人工抽检（每日固定量）                             │
│  - 每天抽100条做人工评估                                     │
│  - 重点评估：自动评分低的、有异常的、高风险的                │
│  - 输出：人工评分 + 根因分析                                 │
├─────────────────────────────────────────────────────────────┤
│  Step 4: 对抗性测试（每周）                                 │
│  - 主动构造极端Case                                          │
│  - 测试边界、矛盾、越权、模糊、多意图                        │
│  - 输出：鲁棒性报告                                          │
├─────────────────────────────────────────────────────────────┤
│  Step 5: 汇总分析                                            │
│  - 自动评分 vs 人工评分的一致性                              │
│  - 高频问题类型统计                                          │
│  - Badcase加入回归集                                         │
└─────────────────────────────────────────────────────────────┘
```


### 五、生产级最佳实践

#### 实践 1：全链路追踪

```python
class AgentTracer:
    def trace(self, request_id):
        return {
            "request_id": request_id,
            "query": ...,
            "intent": ...,
            "retrieved_docs": [...],
            "tool_calls": [...],
            "response": ...,
            "latency": ...,
            "token_cost": ...,
            "steps": [...],
            "timestamp": ...
        }
```

**价值**：抽检时能完整回放整个执行链路，快速定位问题层。

#### 实践 2：自动评分校准

```python
def calibrate_judge(judge, human_labels):
    """用人工标注校准LLM裁判"""
    agreement = 0
    for case in human_labels:
        llm_score = judge.evaluate(case)
        human_score = case.human_score
        if abs(llm_score - human_score) <= 1:  # 允许1分误差
            agreement += 1
    
    agreement_rate = agreement / len(human_labels)
    if agreement_rate < 0.85:
        # 一致性不足，需要调整Judge Prompt
        adjust_judge_prompt()
    
    return agreement_rate
```

#### 实践 3：抽检结果可视化

```
每日抽检报告：
- 自动评分分布：优秀(>4) 60%，良好(3-4) 30%，差(<3) 10%
- 人工评分 vs 自动评分一致性：87%
- Top 3问题类型：
  1. 检索召回不足（35%）
  2. 工具参数错误（25%）
  3. 意图识别错误（20%）
- 新增Badcase：12条，已加入回归集
```


## 六、回答模板

> **Agent 评估必须**结果与过程并重**，我从四个维度构建评估体系：结果指标（任务完成率、准确率、幻觉率）、过程指标（工具调用准确率、步骤效率、循环率）、效率指标（延迟、Token 消耗）、体验指标（追问率、放弃率）。
>
> **没有用户反馈时，我用四层抽检策略**：
>
> **第一是自动抽检**。用 LLM-as-a-Judge 对 100%流量做自动评估，同时检测异常模式——循环、超长、工具失败、幻觉。但 LLM 裁判必须先与人工标注做校准，一致性达到 85%以上才能大规模使用。
>
> **第二是分层抽样**。按风险等级抽样——高风险场景 100%人工审核，中风险 20%，低风险 5%。
>
> **第三是对抗性抽检**。主动构造极端 Case——边界输入、矛盾输入、越权输入、模糊输入、多意图输入，测试系统鲁棒性。
>
> **第四是影子模式**。线上流量自动抽检，低分或有异常的加入人工审核队列。
>
> **最终形成闭环**：自动评分→异常检测→人工复核→根因分析→Badcase 加入回归集→持续优化。
>
> **核心认知**：没有用户反馈不等于没有反馈——**用 LLM 裁判+分层抽样+对抗测试，可以主动构造出高质量的评估信号**。


## SFT 有监督微调

Agent 中的 **SFT 优化**，全称是 **Supervised Fine-Tuning（有监督微调）**，是 Agent 训练流程中非常关键的“冷启动”阶段。简单来说，它的核心目的是**把一个只会“接话茬”的通用大模型，改造成一个会“干活”的智能体**。

下面我从几个维度帮你拆解一下：

### 核心目标：教模型“怎么干活”，而不仅是“怎么说话”

普通大模型（Pre-trained Model）经过海量文本训练，知识渊博但只会根据上文预测下一个词，它并不理解“助手”的角色，也不知道如何执行复杂任务。

**Agent SFT** 就是通过提供大量高质量的“专家示范”数据（即**轨迹数据 Trajectory**），让模型通过**模仿学习（Imitation Learning / Behavior Cloning）**，学会一套标准化的行为模式：

- **任务拆解与规划**：遇到复杂问题知道先做什么、后做什么。
- **工具调用**：知道在什么时机、用什么参数去调用外部工具（如搜索、计算器、API）。
- **结果整合与反思**：能根据工具返回的结果，生成最终回复或进行自我修正。

### 训练数据的形态变化

这是理解 Agent SFT 的关键。普通 SFT 的数据是简单的“问-答”对，而 Agent SFT 的数据是一条完整的**交互轨迹**。

|对比维度|普通 SFT|Agent SFT|
|:--|:--|:--|
|**核心能力**|回答问题、指令遵循|规划、调用工具、反思、行动|
|**数据格式**|用户提问 → 模型回答|任务 → 思考 → 工具调用 → 观察结果 → 最终回复|
|**输出形式**|自然语言文本|结构化动作（如 JSON、函数调用）+ 自然语言|

**一个典型的 Agent SFT 训练样本长这样：**

```json
{
  "用户指令": "帮我查一下明天北京的天气，然后写一封出行建议邮件",
  "模型思考": "需要先调用天气API获取数据，再根据结果撰写邮件。",
  "工具调用": "weather_api(city='北京', date='明天')",
  "观察结果": "晴，25°C，微风",
  "最终回复": "明天北京天气晴朗，气温25度，建议穿着轻薄衣物出行..."
}
```

模型通过学习成千上万条这样的标准轨迹，初步掌握了工具调用的格式、问题与工具的映射关系，以及多步推理的逻辑。

### 关键技术细节：Loss Masking

在 Agent SFT 的训练中，有一个非常重要的技术点叫 **Loss Masking（损失掩码）**。

- **为什么要 Mask？** 训练的本质是让模型学会预测下一个 token。但是，轨迹中的**“观察结果”（Observation）**是外部环境（如 API）返回的真实信息，模型是无法预测的，也不应该去预测。
- **怎么做？** 在计算损失函数（Loss）时，必须把“观察结果”这部分内容的 Loss 屏蔽掉（Mask 掉）。
- **不做的后果：** 如果不对这部分进行 Mask，模型会试图去“生成”工具返回的结果，这不仅会导致严重的幻觉（凭空捏造数据），还会产生梯度污染，让模型学不到正确的决策逻辑。

训练时，模型只对**思考过程（Think）、工具调用决策（Tool Call）和最终回复**计算 Loss，确保它只学习“决策逻辑”，而不是去记忆或猜测环境反馈。

### 在整体训练链路中的位置

SFT 通常是 Agent 训练的**第一阶段（冷启动）**。

1. **SFT（模仿学习）**：先通过 SFT 让模型学会基本的工具使用规范和流程，解决“何时用、为何用、怎么用”的问题。
2. **RL（强化学习）**：在 SFT 的基础上，通常会引入强化学习（如 GRPO、PPO 算法）。通过试错和结果反馈，让模型从“模仿”进化为“策略性审慎使用工具”，解决 SFT 泛化性差、无法理解深层逻辑的问题。

### 常见的实现方式

由于大模型参数量巨大，全参数微调成本极高，因此在 Agent SFT 中常采用**参数高效微调（PEFT）**技术，最典型的就是 **LoRA (Low-Rank Adaptation)**。

- **LoRA 的核心思想**：冻结预训练模型的大部分参数，只在特定的层插入可训练的低秩矩阵。
- **优势**：大幅降低显存开销和训练时间，同时能保持原模型的通用能力，防止过拟合。

总结一下，**Agent 中的 SFT 优化**，就是利用包含完整交互轨迹的高质量数据，通过有监督学习的方式，赋予通用大模型**感知环境、任务拆解、工具调用及反思修正**等智能体核心能力的过程。它是构建可靠、可解释且高效的 LLM Agent 的基石。


## 如何判断应该对哪个 Agent 做 SFT（有监督微调） 优化？

这个问题触及了 SFT（监督微调）决策的核心：**不是所有问题都值得用 SFT 解决，也不是所有 Agent 都适合 SFT。** 这里是想看你是否有**成本效益意识**——知道什么时候该用 Prompt 解决，什么时候该上 SFT。

下面从**决策框架、判断标准、优先级排序、替代方案**四个维度系统拆解。

### 一、核心原则：先穷尽 Prompt，再考虑 SFT

```
问题出现
    ↓
┌─────────────────────────────────────────────────────────────┐
│  第一层：Prompt能解决吗？                                   │
│  - 改System Prompt？                                        │
│  - 加Few-shot示例？                                         │
│  - 优化工具定义？                                           │
│  → 能解决 → 改Prompt（成本低、见效快）                      │
├─────────────────────────────────────────────────────────────┤
│  第二层：RAG能解决吗？                                      │
│  - 补充知识库？                                             │
│  - 优化检索策略？                                           │
│  → 能解决 → 优化RAG（成本中、见效中）                       │
├─────────────────────────────────────────────────────────────┤
│  第三层：换模型能解决吗？                                   │
│  - 换更强的模型？                                           │
│  → 能解决 → 换模型（成本高、但无需训练）                    │
├─────────────────────────────────────────────────────────────┤
│  第四层：只有SFT能解决吗？                                  │
│  - 需要固定风格/格式？                                      │
│  - 需要领域术语理解？                                       │
│  - 需要特定推理模式？                                       │
│  → 是 → SFT（成本最高、周期最长）                           │
└─────────────────────────────────────────────────────────────┘
```

**核心认知**：SFT 是最后手段，不是首选方案。能用 Prompt 和 RAG 解决的，绝不轻易上 SFT。


### 二、判断标准：什么情况该做 SFT？

#### 标准 1：Prompt 和 RAG 都无法解决的系统性问题

| 问题类型 | Prompt 能解决？ | RAG 能解决？ | 需要 SFT？ |
|----------|---------------|-------------|-----------|
| 知识缺失 | ❌ | ✅ | ❌ |
| 格式不固定 | ✅ | ❌ | ❌ |
| 风格不一致 | ⚠️ 部分 | ❌ | ✅ |
| 领域术语不理解 | ❌ | ⚠️ 部分 | ✅ |
| 推理模式错误 | ⚠️ 部分 | ❌ | ✅ |
| 工具调用格式错误 | ✅ | ❌ | ❌ |
| 幻觉严重 | ⚠️ 部分 | ✅ | ⚠️ 部分 |

#### 标准 2：问题具有高频性和重复性

```python
def should_sft(problem):
    # 1. 频率：这个问题出现频率高吗？
    frequency = problem.occurrence_count / total_requests
    if frequency < 0.05:  # 低于5%，不值得SFT
        return False
    
    # 2. 一致性：同类问题反复出现吗？
    if problem.pattern_consistency < 0.7:
        return False  # 问题本身不稳定，SFT效果差
    
    # 3. 影响：这个问题影响大吗？
    if problem.impact_score < 3:  # 1-5分
        return False  # 影响小，不值得投入
    
    return True
```

#### 标准 3：有高质量的标注数据

```python
def has_sufficient_data(problem):
    # 1. 数据量：至少500-1000条高质量样本
    if problem.labeled_samples < 500:
        return False
    
    # 2. 数据质量：标注准确率>95%
    if problem.annotation_accuracy < 0.95:
        return False
    
    # 3. 数据多样性：覆盖主要场景
    if problem.scenario_coverage < 0.8:
        return False
    
    return True
```

#### 标准 4：SFT 的 ROI 高于其他方案

| 方案 | 成本 | 周期 | 效果提升 | ROI |
|------|------|------|----------|-----|
| 改 Prompt | 低 | 1 天 | 5-10% | ⭐⭐⭐⭐⭐ |
| 优化 RAG | 中 | 1 周 | 10-20% | ⭐⭐⭐⭐ |
| 换模型 | 高 | 1 天 | 15-30% | ⭐⭐⭐ |
| SFT | 很高 | 1-2 月 | 20-40% | ⭐⭐ |


### 三、优先级排序：先优化哪个 Agent？

#### 排序矩阵

| 维度 | 权重 | 说明 |
|------|------|------|
| **问题频率** | 30% | 出现越频繁，优先级越高 |
| **影响程度** | 25% | 对用户体验/业务的影响 |
| **修复难度** | 20% | Prompt 能解决的不做 SFT |
| **数据就绪度** | 15% | 有标注数据的优先 |
| **ROI** | 10% | 投入产出比 |

#### 决策树

```
问题出现
    ↓
问题频率 > 10%？
    ├─ 否 → 改Prompt/加Few-shot
    └─ 是 → 影响程度 > 4分？
              ├─ 否 → 排期优化
              └─ 是 → Prompt能解决？
                        ├─ 能 → 改Prompt
                        └─ 不能 → RAG能解决？
                                    ├─ 能 → 优化RAG
                                    └─ 不能 → 换模型能解决？
                                                ├─ 能 → 换模型
                                                └─ 不能 → 有标注数据？
                                                            ├─ 有 → SFT
                                                            └─ 无 → 先积累数据
```

#### 多 Agent 场景下的优先级

假设你有三个 Agent：客服 Agent、订票 Agent、推荐 Agent。

| Agent | 问题频率 | 影响程度 | Prompt 能解决 | 数据就绪 | 优先级 |
|-------|----------|----------|-------------|----------|--------|
| 客服 Agent | 15% | 5 | ❌ | ✅ | **P 0** |
| 订票 Agent | 8% | 4 | ✅ | - | P 2（改 Prompt） |
| 推荐 Agent | 3% | 2 | ❌ | ❌ | P 3（先积累数据） |


### 四、SFT 的具体决策流程

#### Step 1：问题诊断

```python
class SFTDiagnoser:
    def diagnose(self, agent, problem):
        # 1. 问题分类
        problem_type = self.classify_problem(problem)
        
        # 2. 尝试Prompt修复
        prompt_fix = self.try_prompt_fix(agent, problem)
        if prompt_fix.improvement > 0.3:
            return "改Prompt即可"
        
        # 3. 尝试RAG修复
        rag_fix = self.try_rag_fix(agent, problem)
        if rag_fix.improvement > 0.3:
            return "优化RAG即可"
        
        # 4. 评估SFT可行性
        if self.is_sft_feasible(problem):
            return "建议SFT"
        else:
            return "建议换模型或积累数据"
```

#### Step 2：SFT 可行性评估

```python
def is_sft_feasible(problem):
    return {
        "data_volume": problem.labeled_samples >= 500,
        "data_quality": problem.annotation_accuracy >= 0.95,
        "problem_consistency": problem.pattern_consistency >= 0.7,
        "frequency": problem.occurrence_rate >= 0.05,
        "impact": problem.impact_score >= 3,
        "no_alternative": problem.prompt_fix_failed and problem.rag_fix_failed
    }
```

#### Step 3：SFT 方案设计

```python
class SFTPlan:
    def __init__(self, problem):
        self.base_model = self.select_base_model(problem)
        self.method = self.select_method(problem)  # LoRA / QLoRA / Full
        self.data = self.prepare_data(problem)
        self.eval = self.design_eval(problem)
    
    def select_base_model(self, problem):
        # 根据领域选择基座
        if problem.domain == "medical":
            return "Qwen2.5-7B-Instruct"  # 中文医疗场景
        elif problem.domain == "code":
            return "DeepSeek-Coder-7B"
        else:
            return "Qwen2.5-7B-Instruct"
    
    def select_method(self, problem):
        # 数据量小用LoRA，数据量大用Full
        if problem.labeled_samples < 5000:
            return "LoRA"
        else:
            return "Full SFT"
```


### 五、SFT 的替代方案

在决定 SFT 前，先确认以下方案是否可行：

| 替代方案 | 适用场景 | 成本 | 效果 |
|----------|----------|------|------|
| **Prompt 优化** | 格式、风格问题 | 低 | ⭐⭐⭐ |
| **Few-shot** | 示例学习 | 低 | ⭐⭐⭐ |
| **RAG 优化** | 知识缺失 | 中 | ⭐⭐⭐⭐ |
| **换模型** | 能力不足 | 高 | ⭐⭐⭐⭐ |
| **DPO/RLHF** | 偏好对齐 | 很高 | ⭐⭐⭐⭐⭐ |
| **模型蒸馏** | 大模型→小模型 | 高 | ⭐⭐⭐⭐ |


### 六、回答模板

> **我的核心原则是**先穷尽 Prompt 和 RAG，再考虑 SFT**。SFT 是最后手段，不是首选方案。
>
> **判断标准有四个**：
>
> **第一，Prompt 和 RAG 都无法解决**。如果能通过改 Prompt、加 Few-shot、优化 RAG 解决，绝不轻易上 SFT。
>
> **第二，问题具有高频性和重复性**。出现频率超过 5%、模式一致性超过 70%的问题才值得 SFT。
>
> **第三，有高质量的标注数据**。至少 500-1000 条高质量样本，标注准确率>95%。
>
> **第四，SFT 的 ROI 高于其他方案**。对比改 Prompt、优化 RAG、换模型的成本和效果，SFT 只有在其他方案都失败时才值得投入。
>
> **多 Agent 场景下，我用优先级矩阵排序**：问题频率（30%）、影响程度（25%）、修复难度（20%）、数据就绪度（15%）、ROI（10%）。综合得分最高的优先 SFT。
>
> **具体流程是**：问题诊断→尝试 Prompt 修复→尝试 RAG 修复→评估 SFT 可行性→设计 SFT 方案。
>
> **核心认知**：SFT 不是"万能药"，而是"最后手段"。能用工程手段解决的，绝不轻易上训练。SFT 的成本是 Prompt 的 100 倍，效果提升可能只有 20%——**ROI 才是决策的核心**。


## Prompt 调优过程中，经常会遇到 "修好一类、坏了另一类" 的问题，你是怎么解决的？

这个问题直击 Agent 迭代中最令人头疼的“跷跷板效应”：**优化了一个场景，却把另一个场景搞坏了。** 这里是想看你是否有**系统性回归防护**的能力，而不是“打地鼠式”的修补。

下面从**根因分析、检测机制、修复策略、工程防护**四个层面系统拆解。

### 一、为什么会“修好一类、坏了另一类”？

| 根因 | 表现 | 本质 |
|------|------|------|
| **Prompt 过度拟合** | 为 A 场景加的约束，在 B 场景变成了限制 | 约束缺乏条件判断 |
| **工具定义冲突** | 为 A 场景新增的工具，被 B 场景误调用 | 工具描述边界不清 |
| **RAG 检索干扰** | 为 A 场景补充的文档，在 B 场景被召回 | 检索缺乏场景过滤 |
| **模型能力偏移** | SFT 后 A 场景提升，B 场景退化 | 训练数据分布不均衡 |
| **上下文污染** | 为 A 场景加的 Few-shot，在 B 场景被模仿 | 示例缺乏场景隔离 |
| **路由误判** | A 场景的规则把 B 场景的 Query 也拦截了 | 规则过于宽泛 |

**核心认知**：跷跷板效应的本质是**局部优化缺乏全局约束**——你在一个点上用力，却不知道这个力会传导到哪里。

### 二、检测机制：如何发现“坏了另一类”？

#### 1. 回归测试集（最基础）

```python
class RegressionTestSet:
    def __init__(self):
        self.cases = []  # 每个case包含：query, expected, scenario
    
    def add_case(self, query, expected, scenario):
        self.cases.append({
            "query": query,
            "expected": expected,
            "scenario": scenario,  # 场景标签
            "added_at": now()
        })
    
    def run_all(self, agent):
        results = {}
        for case in self.cases:
            response = agent.run(case.query)
            is_correct = self.check(response, case.expected)
            results[case.scenario] = results.get(case.scenario, [])
            results[case.scenario].append(is_correct)
        
        # 按场景统计
        return {
            scenario: sum(scores) / len(scores)
            for scenario, scores in results.items()
        }
```

**关键设计**：回归集必须**按场景分组**，每次变更后对比各场景的通过率变化。

#### 2. 场景化监控

```python
class ScenarioMonitor:
    def __init__(self):
        self.baseline = {}  # 各场景的基线指标
    
    def set_baseline(self, scenario_metrics):
        self.baseline = scenario_metrics
    
    def detect_regression(self, current_metrics):
        regressions = []
        for scenario, current_score in current_metrics.items():
            baseline_score = self.baseline.get(scenario, 0)
            if current_score < baseline_score - 0.05:  # 下降超过5%
                regressions.append({
                    "scenario": scenario,
                    "baseline": baseline_score,
                    "current": current_score,
                    "drop": baseline_score - current_score
                })
        return regressions
```

#### 3. 变更影响分析

```python
def analyze_change_impact(change, test_suite):
    """分析一次变更对各个场景的影响"""
    before = run_test_suite(test_suite)  # 变更前
    apply_change(change)
    after = run_test_suite(test_suite)   # 变更后
    
    impact = {}
    for scenario in before:
        impact[scenario] = {
            "before": before[scenario],
            "after": after[scenario],
            "delta": after[scenario] - before[scenario]
        }
    
    # 找出被"修好"和"弄坏"的场景
    fixed = [s for s, d in impact.items() if d["delta"] > 0.05]
    broken = [s for s, d in impact.items() if d["delta"] < -0.05]
    
    return {
        "fixed": fixed,
        "broken": broken,
        "impact": impact
    }
```


### 三、修复策略：如何让“修好 A 不坏 B”？

#### 策略 1：条件化约束（最核心）

**问题**：为 A 场景加的约束，在 B 场景变成了限制。

**解决**：把约束改为**条件触发**。

```python
# 不好的约束（全局生效）
"禁止使用专业术语"

# 好的约束（条件生效）
"如果用户是普通用户，禁止使用专业术语；
 如果用户是专业用户，可以使用专业术语"
```

```python
# Prompt中的条件化设计
"""
## 约束
1. 如果用户问的是退货，必须询问订单号。
2. 如果用户问的是换货，必须询问商品尺码。
3. 如果用户问的是物流，必须询问快递单号。
"""
```

#### 策略 2：场景隔离

**问题**：为 A 场景加的 Few-shot，在 B 场景被模仿。

**解决**：把 Few-shot 按场景分组，只加载相关场景的示例。

```python
SCENARIO_EXAMPLES = {
    "退货": [
        {"query": "我要退货", "response": "请提供订单号"},
        {"query": "怎么退", "response": "请提供订单号"}
    ],
    "换货": [
        {"query": "我要换货", "response": "请提供商品尺码"},
        {"query": "换个尺码", "response": "请提供商品尺码"}
    ]
}

def build_prompt(scenario):
    examples = SCENARIO_EXAMPLES.get(scenario, [])
    return f"""
    ## 示例
    {format_examples(examples)}
    """
```

#### 策略 3：分层路由

**问题**：A 场景的规则把 B 场景的 Query 也拦截了。

**解决**：路由规则加**优先级和互斥条件**。

```python
class Router:
    def route(self, query):
        # 按优先级依次判断，互斥
        if self.is_refund(query):
            return "refund_agent"
        elif self.is_exchange(query):
            return "exchange_agent"
        elif self.is_logistics(query):
            return "logistics_agent"
        else:
            return "general_agent"
    
    def is_refund(self, query):
        # 精确匹配，避免误判
        return any(kw in query for kw in ["退货", "退款", "退钱"])
    
    def is_exchange(self, query):
        return any(kw in query for kw in ["换货", "换尺码", "换颜色"])
```

#### 策略 4：A/B 测试验证

**问题**：不知道变更会不会影响其他场景。

**解决**：每次变更前，跑全量回归测试。

```python
def safe_deploy(change, regression_suite):
    # 1. 在测试环境跑回归
    impact = analyze_change_impact(change, regression_suite)
    
    # 2. 如果有场景退化超过5%，阻断部署
    if impact["broken"]:
        return {
            "status": "blocked",
            "reason": f"以下场景退化：{impact['broken']}",
            "suggestion": "请调整变更方案"
        }
    
    # 3. 如果只有提升，部署
    return {
        "status": "approved",
        "fixed": impact["fixed"]
    }
```

#### 策略 5：多目标优化

**问题**：SFT 后 A 场景提升，B 场景退化。

**解决**：训练数据**按场景均衡采样**。

```python
def balance_training_data(data, scenarios):
    """按场景均衡采样，避免某个场景主导训练"""
    balanced = []
    for scenario in scenarios:
        scenario_data = [d for d in data if d.scenario == scenario]
        # 每个场景采样相同数量
        sample_size = min(len(scenario_data), 500)
        balanced.extend(random.sample(scenario_data, sample_size))
    return balanced
```


### 四、工程防护：建立“不坏另一类”的机制

#### 1. 全量回归门禁

```yaml
# CI流水线
name: Agent Regression Gate
on: pull_request
jobs:
  regression:
    steps:
      - name: 跑全量回归测试
        run: python run_regression.py
      - name: 检查场景退化
        run: python check_regression.py --threshold 0.05
        # 如果有场景退化超过5%，阻断合并
```

#### 2. 灰度发布

```python
def canary_deploy(change):
    # 1. 先对10%流量部署
    deploy(change, traffic_ratio=0.1)
    
    # 2. 监控各场景指标
    metrics = monitor(duration="1h")
    
    # 3. 如果各场景指标正常，扩大到100%
    if all_scenarios_healthy(metrics):
        deploy(change, traffic_ratio=1.0)
    else:
        rollback(change)
        alert("灰度发现场景退化")
```

#### 3. 场景标签体系

```python
# 每个请求都打上场景标签
class ScenarioTagger:
    def tag(self, trace):
        return {
            "scenario": self.classify(trace.query),
            "intent": trace.intent,
            "tools_used": trace.tools_used,
            "timestamp": trace.timestamp
        }

# 监控时按场景聚合
def aggregate_by_scenario(traces):
    scenarios = {}
    for trace in traces:
        tag = ScenarioTagger().tag(trace)
        scenario = tag["scenario"]
        if scenario not in scenarios:
            scenarios[scenario] = []
        scenarios[scenario].append(trace)
    return scenarios
```

#### 4. 变更影响看板

```
变更影响看板：
- 变更ID：PR-1234
- 变更内容：优化退货流程Prompt
- 影响分析：
  ✅ 退货场景：78% → 92%（+14%）
  ⚠️ 换货场景：85% → 83%（-2%）
  ✅ 物流场景：80% → 80%（0%）
- 决策：通过（换货场景退化在5%以内）
```


### 五、回答模板

> **这个问题的本质是**局部优化缺乏全局约束**。我从三个层面解决：
>
> **第一是检测机制**。建立按场景分组的回归测试集，每次变更后跑全量回归，对比各场景的通过率变化。同时做场景化监控，任何场景退化超过 5%就告警。
>
> **第二是修复策略**。核心是**条件化约束**——为 A 场景加的约束，改为“如果用户问的是 A，则...；如果问的是 B，则...”，而不是全局生效。同时做场景隔离，Few-shot 按场景分组，只加载相关场景的示例。路由规则加优先级和互斥条件，避免误判。
>
> **第三是工程防护**。建立全量回归门禁，CI 流水线自动检查场景退化，超过 5%阻断合并。灰度发布先跑 10%流量，监控各场景指标正常后才全量。每个请求打场景标签，监控时按场景聚合。
>
> **核心认知**：解决跷跷板效应的关键不是“找到完美的约束”，而是**建立场景化的防护体系**——让每次变更都能被量化评估，让退化能被及时发现和阻断。


## 工具调用失败 、超时了怎么办？

分四层处理：

**第一是分类**。先区分失败类型——网络超时、服务不可用、限流这些是暂时性故障，可以重试；参数错误、权限不足、业务逻辑错误是永久性故障，重试无用，需要修正参数或告知用户。

**第二是重试**。对可重试的错误，用指数退避+抖动的策略，避免多个请求同时重试压垮下游。按错误类型设置不同的重试次数——网络超时重试 3 次，限流重试 5 次，参数错误不重试。

**第三是降级**。重试失败后，分层降级：先换备选工具，再换参数，再换策略，最后用已有信息回答或转人工。比如订票 API 超时，先试备选 API，再查缓存，最后告知用户稍后重试。

**第四是工程兜底**。加熔断机制，当下游连续失败超过阈值时直接熔断，不再调用。做全链路追踪，记录每次调用的耗时、状态、重试次数。配置告警，失败率超过 5%就告警。

**核心认知**：工具调用失败是常态，不是异常。好的设计不是"保证不失败"，而是**"失败时系统能优雅地摔倒，而不是崩溃"**。


## 开发 agent 时踩过哪些坑？

开发 Agent 的坑，和传统软件开发的坑有本质区别——**传统软件的 bug 是确定性的，Agent 的 bug 是概率性的、涌现的、难以复现的。** 这里是想看你有没有"踩过坑、填过坑"的实战经验。

下面按**开发阶段、架构设计、工程落地、生产运维**四个维度，把最常见的坑系统梳理一遍。

### 一、开发阶段的坑

#### 坑 1：Prompt 写了但模型不遵循

**表现**：明明在 System Prompt 里写了"禁止编造"，模型还是编了。

**根因**：
- 约束放在 Prompt 中间，被上下文淹没（Lost in the Middle）
- 约束表述模糊，模型无法判断"什么算编造"
- 约束太多，模型注意力被分散

**解法**：
```python
# 不好的写法（模糊、位置靠后）
"请准确回答用户问题，不要编造信息，保持专业..."

# 好的写法（明确、前置、可验证）
"""
## 核心约束（必须遵守）
1. 如果检索结果中没有答案，必须回答"我需要查询更多信息"，禁止编造。
2. 所有数字、日期、名称必须来自检索结果，禁止推断。
3. 如果用户问题模糊，必须先追问，不得猜测。
"""
```

#### 坑 2：工具定义太宽泛，模型乱调用

**表现**：模型调用了不该调用的工具，或者参数填错。

**根因**：工具描述模糊，模型不知道"什么时候该用、什么时候不该用"。

**解法**：
```python
# 不好的定义
{"name": "search", "description": "搜索信息"}

# 好的定义
{
    "name": "search_flight",
    "description": """查询航班信息。
    使用条件：用户明确表达了查询航班的意图，且已提供出发地、目的地、日期。
    禁止使用：用户只是闲聊、问天气、问其他交通方式。
    参数要求：三个参数必须全部从用户输入中明确获取，禁止推断。"""
}
```

#### 坑 3：上下文太长导致模型"失忆"

**表现**：对话到第 20 轮，模型忘了第 3 轮说过什么。

**根因**：Lost in the Middle——模型对上下文中间部分的信息关注度下降。

**解法**：
- 关键信息前置（放在 System Prompt 中）
- 用 Todo List 显式记录进度
- 分层记忆：短期全量+长期摘要+按需检索


### 二、架构设计的坑

#### 坑 4：纯 ReAct 导致无限循环

**表现**：Agent 反复调用同一个工具，陷入死循环。

**根因**：没有终止条件，或者终止条件被上下文淹没。

**解法**：
```python
# 多层终止机制
class LoopDetector:
    def check(self, history):
        # L1: 步数限制
        if len(history) > 30:
            return "max_steps_exceeded"
        
        # L2: 状态去重（连续3次相同Action）
        recent_actions = [h.action for h in history[-3:]]
        if len(set(recent_actions)) == 1:
            return "duplicate_action"
        
        # L3: 信息增益检测（Observation相似度>95%）
        if self.similarity(history[-1].observation, history[-2].observation) > 0.95:
            return "no_information_gain"
        
        return None
```

#### 坑 5：状态管理混乱

**表现**：多轮对话中，状态丢失或状态不一致。

**根因**：Session、Task、Turn 三层状态没有清晰界定。

**解法**：
```
Session（会话）
├── Task 1（任务）
│   ├── Turn 1
│   └── Turn 2
├── Task 2
│   └── Turn 3
└── Task 3
    ├── Turn 4
    └── Turn 5
```

#### 坑 6：工具调用没有幂等性

**表现**：重试导致重复下单、重复扣款。

**根因**：工具没有幂等设计，重试时重复执行。

**解法**：
```python
# 为每个操作生成唯一ID
def book_flight(params):
    idempotency_key = generate_key(params)
    
    # 先检查是否已执行
    if cache.exists(idempotency_key):
        return cache.get(idempotency_key)
    
    # 执行并缓存结果
    result = do_book(params)
    cache.set(idempotency_key, result, ttl=3600)
    return result
```


### 三、工程落地的坑

#### 坑 7：KV Cache 频繁失效

**表现**：每次请求都重新计算，延迟高、成本高。

**根因**：动态信息（时间戳、工具列表）放在 Prompt 前缀，导致缓存失效。

**解法**：
```
稳定前缀（缓存）：角色定义 + 核心约束 + 工具摘要
动态后缀（不缓存）：时间戳 + 用户Query + 历史对话
```

#### 坑 8：工具列表频繁变动导致缓存全失效

**表现**：每次新增工具，所有请求的缓存都失效。

**解法**：工具列表外置化——Prompt 里只放工具摘要，完整描述通过推理引擎层注入。

#### 坑 9：没有回归测试，改 A 坏 B

**表现**：优化了退货场景，换货场景退化了。

**解法**：
```python
# 按场景分组的回归测试集
regression_suite = {
    "退货": [case1, case2, ...],
    "换货": [case3, case4, ...],
    "物流": [case5, case6, ...]
}

# CI门禁：任何场景退化超过5%就阻断
def check_regression(before, after):
    for scenario in before:
        if after[scenario] < before[scenario] - 0.05:
            raise RegressionError(f"{scenario}退化")
```

### 四、生产运维的坑

#### 坑 10：没有全链路追踪，出问题无法定位

**表现**：用户投诉回答错误，但不知道是哪一步出了问题。

**解法**：
```python
class AgentTracer:
    def trace(self, request_id):
        return {
            "request_id": request_id,
            "query": ...,
            "intent": ...,
            "retrieved_docs": [...],
            "tool_calls": [...],
            "response": ...,
            "latency": ...,
            "token_cost": ...,
            "steps": [...]
        }
```

#### 坑 11：没有熔断，下游挂了整个系统挂

**表现**：天气 API 挂了，整个 Agent 系统都不可用。

**解法**：熔断器 + 降级方案。

#### 坑 12：没有成本监控，账单爆炸

**表现**：月底发现 LLM 账单是上个月的 10 倍。

**解法**：
```python
# 成本监控
metrics = {
    "avg_token_per_request": 1500,
    "daily_token_cost": 500,
    "cost_per_user": 0.02
}

# 告警
if daily_token_cost > 1000:
    alert("日成本超过$1000")
```

#### 坑 13：没有用户反馈闭环，Badcase 无法沉淀

**表现**：用户点踩了，但没人分析为什么。

**解法**：
```python
def handle_feedback(request_id, feedback):
    if feedback == "negative":
        badcase = extract_badcase(request_id)
        badcase_queue.append(badcase)
        if badcase.is_high_risk():
            alert_team(badcase)
```


### 五、最容易被忽视的坑

#### 坑 14：Prompt 由多人维护，逐渐失控

**表现**：半年后没人知道 Prompt 里每条约束为什么存在。

**解法**：Prompt-as-Code——Git 版本控制 + Experiment Log + CI 检查。

#### 坑 15：SFT 后发现通用能力退化

**表现**：微调后特定场景提升，但其他场景退化。

**解法**：训练数据按场景均衡采样 + 全量回归测试。

#### 坑 16：模型升级后行为变化

**表现**：从 GPT-4 升级到 GPT-4 o，原来能用的 Prompt 失效了。

**解法**：模型升级前跑全量回归，对比新旧模型在各场景的表现。


### 六、回答模板

> **我按阶段梳理，最常见的坑有这几类：
>
> **开发阶段**：Prompt 写了但模型不遵循（约束被淹没）、工具定义太宽泛导致乱调用、上下文太长导致失忆。
>
> **架构设计**：纯 ReAct 导致无限循环、状态管理混乱、工具调用没有幂等性导致重复执行。
>
> **工程落地**：KV Cache 频繁失效、工具列表变动导致缓存全失效、没有回归测试导致改 A 坏 B。
>
> **生产运维**：没有全链路追踪导致无法定位、没有熔断导致下游挂了系统挂、没有成本监控导致账单爆炸、没有用户反馈闭环导致 Badcase 无法沉淀。
>
> **最容易被忽视的**：Prompt 多人维护逐渐失控、SFT 后通用能力退化、模型升级后行为变化。
>
> **核心认知**：Agent 开发的坑，本质是**概率性系统的不确定性**——传统软件的 bug 是确定性的，Agent 的 bug 是涌现的、难以复现的。所以必须用**工程化的防护体系**来兜底：回归测试、熔断降级、全链路追踪、成本监控，缺一不可。

