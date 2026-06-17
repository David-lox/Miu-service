# 小miu-Service-Agent: 电商客服Agent系统

---

## 项目架构 

### 当前架构

```
ecom-service-agent/
├── main.py                        # CLI 入口（支持单 Agent / Multi-Agent 模式切换 + memory/skills 命令）
├── requirements.txt
├── .env.example
│
├── app/                           # 主 Bot 全部代码 + 数据
│   ├── config/
│   │   └── settings.py            # 配置管理（从 .env 读取，含 MCP / RAG / Multi-Agent / Memory / Skill / Evaluation 配置）
│   ├── prompts/
│   │   ├── customer_service.py    # 电商客服 system prompt（含工具使用指南 + 记忆能力）
│   │   ├── summarizer.py          # 历史摘要 prompt
│   │   ├── agents.py              # Multi-Agent 子 Agent prompt（售前/售后/投诉 + Router）
│   │   ├── memory.py              # 记忆提取 prompt（短期 STM / 长期 LTM 事实抽取）
│   │   └── evaluation.py          # LLM-as-judge prompt（回答质量 / 幻觉 / 过程合理性）
│   ├── schemas/
│   │   └── response.py            # 结构化输出 schema（Pydantic）
│   ├── agent/                     # Agent 核心实现 + 全部 Agent 技术栈（tools / rag / skills）
│   │   ├── chat.py                # 核心 ReAct 循环（集成 MemoryManager + SkillManager）
│   │   ├── summarizer.py          # LLM 自我压缩老对话（支持工具消息）
│   │   ├── storage.py             # 会话 JSON 持久化（含短期记忆）
│   │   ├── memory/                # 记忆系统
│   │   │   ├── __init__.py        # 导出 MemoryManager / ShortTermMemory / LongTermMemory
│   │   │   ├── manager.py         # MemoryManager：统一管理短期 + 长期记忆
│   │   │   ├── short_term.py      # 短期记忆：会话内事实提取
│   │   │   ├── long_term.py       # 长期记忆：跨会话持久化（JSON per user）
│   │   │   └── extraction.py      # LLM 事实提取（共用模块）
│   │   ├── skills/                # Skill 模块：代码 + 技能内容分层
│   │   │   ├── __init__.py        # 导出 SkillManager / SkillMeta
│   │   │   ├── loader.py          # SkillManager：扫描、发现、加载 SKILL.md（渐进式披露）
│   │   │   └── definitions/       # 技能内容（遵循 Agent Skills 开放标准，每个一个 SKILL.md）
│   │   │       ├── process-return/
│   │   │       │   └── SKILL.md   # 退货退款处理技能（确认订单→校验资格→退款→告知进度）
│   │   │       ├── track-order/
│   │   │       │   └── SKILL.md   # 订单物流跟踪技能（查单→查物流→综合建议）
│   │   │       └── product-recommend/
│   │   │           └── SKILL.md   # 商品推荐技能（了解需求→查偏好→搜索→推荐）
│   │   ├── strategies/            # (upcoming) Agent 执行策略
│   │   ├── tools/                 # 电商工具集（Function Calling）
│   │   │   ├── mock_data.py       # Mock 数据：订单、商品、物流
│   │   │   ├── registry.py        # 本地工具注册表 + OpenAI schema + 分发执行
│   │   │   ├── manager.py         # ToolManager：统一管理本地 + MCP 工具（支持 allowed_tools 过滤）
│   │   │   ├── order.py           # 查询订单详情
│   │   │   ├── product.py         # 搜索商品信息
│   │   │   ├── logistics.py       # 查询物流轨迹
│   │   │   ├── refund.py          # 申请退款
│   │   │   ├── knowledge.py       # search_knowledge：RAG 政策/FAQ 检索
│   │   │   ├── memory_tool.py     # recall_user_memory：查询用户记忆
│   │   │   └── skill_tool.py      # load_skill：按需加载技能指令
│   │   └── rag/                   # RAG 模块
│   │       ├── chunker.py         # Markdown → Chunk（按二级标题切分）
│   │       ├── embedder.py        # OpenAI Embeddings 封装
│   │       ├── retriever.py       # KnowledgeRetriever：query → 向量检索
│   │       ├── backends/          # 向量后端
│   │       │   ├── base.py        # VectorBackend 抽象接口
│   │       │   ├
│   │       │   └── chroma_backend.py  # Chroma 嵌入式向量数据库
│   │       └── knowledge/         # 知识库源文档（markdown，RAG 数据源）
│   │           ├── 退换货政策.md
│   │           ├── 配送说明.md
│   │           ├── 会员权益.md
│   │           └── 常见问题FAQ.md
│   ├── mcp_client/                # MCP Client（同步封装）
│   │   ├── client.py              # MCPClient：后台线程管理异步连接
│   │   └── converter.py           # MCP Tool schema → OpenAI function calling 格式
│   ├── evaluation/                # Agent 评估体系
│   │   ├── __init__.py            # 导出 EvalCase / Sandbox / Evaluator / RunTrace 等
│   │   ├── dataset.py             # EvalCase 数据结构 + load_dataset
│   │   ├── trace.py               # RunTrace：沙箱采集的过程+结果载体
│   │   ├── sandbox.py             # Sandbox：隔离环境 + 共享 client 插桩 + 采集
│   │   ├── metrics.py             # 过程/结果双层指标（代码规则 + LLM judge）
│   │   ├── evaluator.py           # Evaluator：跑用例 → 双层评分 → 聚合报告
│   │   └── cases.json             # 黄金测试集（~10 条，引用 mock 数据）
│   ├── multi_agent/               # Multi-Agent 协作
│   │   ├── router.py              # 意图路由器（LLM 分类 → 子 Agent）
│   │   ├── agents.py              # SubAgent 子 Agent 类 + 配置
│   │   └── orchestrator.py        # 编排器：路由 → 执行 → 结构化提取（集成 MemoryManager + SkillManager）
│   ├── scripts/
│   │   ├── build_kb_index.py      # 离线构建知识库索引（--backend numpy/chroma）
│   │   └── run_eval.py            # 离线运行评估（--mode single/multi · --judge/--no-judge · --output）
│   └── sessions/                  # 运行时生成，已 .gitignore
│       ├── session.json           # 当前会话快照
│       ├── kb_index.json          # NumpyBackend 索引
│       ├── chroma/                # ChromaBackend 持久化目录
│       └── memory/                # 长期记忆存储（按 user_id 分文件）
│           └── {user_id}.json
│
├── mcp_server/                    # MCP Server（独立微服务）
│   └── server.py                  # FastMCP + Streamable HTTP，暴露电商工具
│
└── tests/                         # 全部测试
    ├── test_agent.py              # 结构化输出 + 多轮 + reset
    ├── test_conversation_management.py  # 多轮对话管理
    ├── test_react_agent.py        # ReAct Agent + Function Calling
    ├── test_mcp.py                # MCP 集成
    ├── test_rag.py                # RAG 知识库检索
    ├── test_multi_agent.py        # Multi-Agent 协作
    ├── test_memory.py             # Memory 短期记忆 & 长期记忆
    ├── test_skills.py             # Skill 可复用能力模块
    └── test_evaluation.py         # Agent 评估体系（沙箱 + 双层测评）
```
## 计划

**已实现**
- ReAct 范式的 Agent
- 工具调用 / Function Calling（查订单、查库存等）
- MCP（Model Context Protocol）集成
- RAG 检索增强生成（接入商品库、FAQ、退换货政策等）
- Multi-Agent 协作（客服路由、售前售后分流）
- 多轮对话管理
- Memory：短期记忆 & 长期记忆
- Skill：可复用的能力模块（退货处理、订单跟踪等标准化流程）✅
- Agent 评估体系 ✅

**计划篇**
- Guardrails 安全护栏（Prompt Injection 检测、输出幻觉校验、敏感信息过滤、意图越界拦截）
- Human-in-the-Loop 人机协作（置信度评估与自动转人工、Agent↔真人客服交接协议、上下文传递）
- Agent Observability 可观测性（调用链 Trace、Token/延迟指标采集、工具成功率看板、异常告警）
  
## 运行界面

### 售前
<img width="1086" height="518" alt="售前1" src="https://github.com/user-attachments/assets/a7bd32c4-cb48-426a-96eb-bec93951c758" />
<img width="1090" height="509" alt="售前2" src="https://github.com/user-attachments/assets/d2c19dce-692c-43fd-835c-4113ab2fdb3f" />

### 售后
<img width="1095" height="461" alt="售后1" src="https://github.com/user-attachments/assets/ed7fde88-574e-4192-ad9f-0cd58e475194" />
<img width="782" height="511" alt="售后1 5" src="https://github.com/user-attachments/assets/e6a72f18-e0fe-4168-9da7-2521f5021fc7" />
<img width="768" height="493" alt="售后2" src="https://github.com/user-attachments/assets/7cf4840e-1879-4752-ada7-aa24d0d3ba01" />
<img width="1095" height="325" alt="售后3" src="https://github.com/user-attachments/assets/d62b568e-a654-437a-9fda-26651210542b" />
<img width="987" height="364" alt="售后4" src="https://github.com/user-attachments/assets/a37ca4cd-a1ca-4a25-b0c1-f8e5347a01ea" />
<img width="1062" height="345" alt="售后5" src="https://github.com/user-attachments/assets/9f07b365-23f8-444e-aec4-71794081a56b" />

### 投诉
<img width="950" height="343" alt="投诉1" src="https://github.com/user-attachments/assets/06fd3f57-a951-4113-b79d-19ed3551622f" />
<img width="938" height="305" alt="投诉2" src="https://github.com/user-attachments/assets/a871c092-bb61-4b12-a942-29fc4c2e06d5" />
<img width="956" height="331" alt="投诉3" src="https://github.com/user-attachments/assets/328b4bd7-6d16-48e3-bd21-be3756efdc17" />
<img width="957" height="327" alt="投诉4" src="https://github.com/user-attachments/assets/282fddfc-1cd8-4c99-86be-fbb8aac6a546" />








