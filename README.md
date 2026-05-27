# DeepSearch Pro - 多智能体深度搜索系统

基于 **DeepAgents + LangChain + FastAPI** 构建的企业级多智能体协作系统，专为空调行业设计，支持网络搜索、数据库查询、知识库检索三大信息源的统一调度，并自动生成 Markdown/PDF 报告。

## 系统架构

```
┌─────────────────────────────────────────────────────────────────────┐
│                          前端客户端 (Browser)                        │
│                    通过 REST API 提交任务 / WebSocket 接收实时推送     │
└──────────────┬──────────────────────────────────┬───────────────────┘
               │ POST /api/task                   │ WebSocket /ws/{id}
               │ POST /api/upload                 │
               ▼                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      FastAPI Server (api/server.py)                  │
│  ┌──────────┐  ┌──────────────┐  ┌────────────────────────────┐    │
│  │ REST API │  │ File Upload  │  │  WebSocket ConnectionMgr   │    │
│  └────┬─────┘  └──────┬───────┘  └─────────────┬──────────────┘    │
│       │               │                        │                    │
│       ▼               ▼                        ▼                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Monitor (api/monitor.py) - 单例                 │   │
│  │    工具埋点 → 子智能体调用 → 最终结果 → 实时推送到前端         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │         Context (api/context.py) - ContextVar 协程隔离       │   │
│  │    session_dir / thread_id 的协程级上下文变量，防止多用户串台   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   Main Agent (agent/main_agent.py)                   │
│                DeepAgents 主智能体 - 团队协调者                       │
│                                                                     │
│  ┌─────────────────┐ ┌─────────────────┐ ┌──────────────────────┐  │
│  │ generate_markdown│ │convert_md_to_pdf│ │  read_file_content   │  │
│  │   Markdown生成   │ │   PDF转换       │ │   文件内容读取        │  │
│  └─────────────────┘ └─────────────────┘ └──────────────────────┘  │
│                                                                     │
│  ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┐ │
│    子智能体调度 (LangGraph task tool)                               │
│  └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─┘ │
└────┬──────────────────────┬──────────────────────┬──────────────────┘
     │                      │                      │
     ▼                      ▼                      ▼
┌─────────────┐   ┌──────────────────┐   ┌───────────────────┐
│ 网络搜索助手  │   │  数据库查询助手    │   │  RAGFlow 知识库助手 │
│  Tavily API  │   │   MySQL          │   │   RAGFlow SDK     │
│              │   │                  │   │                   │
│ internet_    │   │ list_sql_tables  │   │ get_assistant_list│
│   search     │   │ get_table_data   │   │ create_ask_delete │
│              │   │ execute_sql_query│   │                   │
└──────┬───────┘   └────────┬─────────┘   └────────┬──────────┘
       │                    │                      │
       ▼                    ▼                      ▼
┌─────────────┐   ┌──────────────────┐   ┌───────────────────┐
│  互联网公开   │   │  pharma_db       │   │  RAGFlow Server   │
│  信息资源     │   │  药品/库存/销售   │   │  企业内部知识文档   │
└─────────────┘   └──────────────────┘   └───────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                        输出层 (output/)                              │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  session_{id}/  ← 每个会话独立的输出目录                      │   │
│  │    ├── report.md       ← Markdown 报告                      │   │
│  │    ├── report.pdf      ← PDF 报告 (经 Word COM 转换)         │   │
│  │    └── uploaded_files  ← 用户上传文件的副本                   │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## 核心工作流

```
用户提问 → Main Agent 分析意图
              │
              ├─ 需要外部知识 → 调用「网络搜索助手」→ Tavily 搜索 → 返回结果
              │
              ├─ 需要业务数据 → 调用「数据库查询助手」→ MySQL 查询 → 返回结果
              │
              ├─ 需要内部文档 → 调用「RAGFlow助手」→ 知识库检索 → 返回结果
              │
              └─ 信息收集完毕 → 生成 Markdown → (可选) 转换 PDF → 推送结果到前端
```

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| Agent 框架 | DeepAgents + LangGraph | 多智能体编排与工具调度 |
| LLM | 通义千问 (qwen-max) | 通过 OpenAI 兼容接口调用 |
| Web 框架 | FastAPI + WebSocket | 异步 REST API + 实时双向通信 |
| 网络搜索 | Tavily API | 互联网公开信息检索 |
| 数据库 | MySQL (mysql-connector) | 企业药品/库存/销售数据查询 |
| 知识库 | RAGFlow SDK | 企业内部文档知识检索 |
| 文件处理 | pypdf / python-docx / pandas | 支持 PDF/Word/Excel 文件读取 |
| PDF 生成 | pywin32 (Word COM) | Markdown → HTML → PDF 转换 |
| 配置管理 | python-dotenv + PyYAML | 环境变量 + Prompt 配置 |

## 项目结构

```
deep_search_pro/
├── agent/                      # 智能体核心
│   ├── main_agent.py           # 主智能体定义与执行入口
│   ├── llm.py                  # LLM 模型初始化
│   ├── prompts.py              # YAML Prompt 配置加载器
│   └── subagents/              # 子智能体定义
│       ├── network_search_agent.py   # 网络搜索助手
│       ├── database_query_agent.py   # 数据库查询助手
│       └── knowledge_base_agent.py   # RAGFlow 知识库助手
├── api/                        # Web 服务层
│   ├── server.py               # FastAPI 主服务 (REST + WebSocket)
│   ├── monitor.py              # 实时监控与消息推送 (单例)
│   └── context.py              # ContextVar 协程级上下文隔离
├── tools/                      # Agent 可调用的工具
│   ├── tavily_tool.py          # Tavily 网络搜索工具
│   ├── db_tools.py             # MySQL 数据库查询工具
│   ├── ragflow_tools.py        # RAGFlow 知识库交互工具
│   ├── markdown_tools.py       # Markdown 文件生成工具
│   ├── pdf_tools.py            # Markdown 转 PDF 工具
│   └── upload_file_read_tool.py # 多格式文件读取工具
├── prompt/                     # Prompt 配置
│   └── prompts.yml             # 主智能体 + 子智能体的 Prompt 模板
├── ragflow/                    # RAGFlow 配置
│   └── rag_config.py           # RAGFlow 环境变量加载
├── utils/                      # 工具函数
│   ├── path_utils.py           # 统一路径解析 (虚拟路径清洗/会话隔离)
│   └── word_converter.py       # Word COM 引擎实现 MD→PDF
├── output/                     # 会话输出目录 (自动生成)
├── updated/                    # 用户上传文件暂存目录
├── .env                        # 环境变量配置
└── requirements.txt            # Python 依赖
```

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 配置环境变量

复制并编辑 `.env` 文件：

```env
# LLM 配置 (通义千问)
OPENAI_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1
OPENAI_API_KEY=your_api_key
LLM_QWEN_MAX=qwen-max

# Tavily 搜索
TAVILY_API_KEY=your_tavily_key

# RAGFlow 知识库
RAGFLOW_API_URL=http://your_ragflow_host
RAGFLOW_API_KEY=your_ragflow_key

# MySQL 数据库
MYSQL_HOST=localhost
MYSQL_PORT=3306
MYSQL_USER=root
MYSQL_PASSWORD=your_password
MYSQL_DATABASE=pharma_db
```

### 3. 启动服务

```bash
python -m api.server
```

服务将在 `http://localhost:8000` 启动，API 文档访问 `http://localhost:8000/docs`。

## API 接口

| 方法 | 路径 | 说明 |
|------|------|------|
| POST | `/api/task` | 提交搜索任务 `{query, thread_id?}` |
| POST | `/api/upload` | 上传文件到指定会话 |
| GET | `/api/files?path=` | 列出会话输出文件 |
| GET | `/api/download?path=` | 下载生成的文件 |
| WS | `/ws/{thread_id}` | WebSocket 实时消息推送 |

### WebSocket 消息格式

```json
{
  "type": "monitor_event",
  "event": "tool_start | assistant_call | task_result | session_created",
  "message": "事件描述",
  "data": {},
  "timestamp": "2026-05-27T..."
}
```

## 设计亮点

- **协程级上下文隔离**：使用 `ContextVar` 实现多用户并发场景下的会话隔离，避免数据串台
- **实时进度推送**：所有工具调用均埋入 Monitor 埋点，通过 WebSocket 实时推送到前端
- **会话文件隔离**：每个会话独立 `output/session_{id}/` 目录，互不干扰
- **智能路径解析**：自动清洗 LLM 生成的虚拟路径（如 `/workspace/`），统一解析到安全目录
- **Prompt 配置化**：所有智能体的 System Prompt 统一管理在 `prompts.yml`，便于调优
