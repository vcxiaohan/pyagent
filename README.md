# pyagent

基于 LangChain ReAct Agent 的扫地机器人智能客服 Demo。Agent 可以自主决定调用哪些工具：从本地知识库检索专业资料（RAG）、查询天气、读取用户的使用记录，并能在识别到"生成使用报告"的需求时，自动切换到报告写作模式，输出一份 Markdown 格式的使用报告与保养建议。

## 技术栈

- **LangChain / LangGraph**：`create_agent` 构建 ReAct Agent，通过中间件做日志监控和提示词切换
- **Chroma**：本地向量数据库
- **通义千问（DashScope）**：`text-embedding-v4` 做向量化，`qwen3-max` 负责推理和生成
- **Streamlit**：聊天页面

## 工作流程

### Agent 循环

Agent 会多次调用大模型，每一轮由大模型决定是直接回答，还是先调用工具：

```
用户提问
  └─> 调用大模型 ──> 需要工具？──否──> 输出最终回答
          ▲              │
          │              是
          │              ▼
          └──── 工具结果追加到消息记录 <── 执行工具
```

### 报告模式切换

普通问答使用 `main_prompt`。当大模型判断用户想要使用报告时，会按以下顺序调用工具：

```
get_user_id ──> get_current_month ──> fill_context_for_report ──> fetch_external_data ──> 写报告
                                              │
                          中间件把 context["report"] 置为 True，
                          之后每次调用大模型都改用 report_prompt
```

`fill_context_for_report` 本身不做任何事，它的作用是给大模型一个"开关"：中间件 `monitor_tool` 监测到它被调用后打上标记，`report_prompt_switch` 在每次调用大模型前读取标记，决定这次使用哪份系统提示词。每次提问都会重置为普通模式。

### 知识库入库

```
data/ 下的 txt、pdf ──> 计算 MD5 去重 ──> 读取 ──> 文本切分 ──> 向量化 ──> 存入 Chroma
```

## 目录结构

```
pyagent/
├── app.py                    # Streamlit 聊天页面（启动入口）
├── agent/
│   ├── react_agent.py        # 创建 Agent，提供流式调用方法
│   └── tools/
│       ├── agent_tools.py    # Agent 可调用的工具
│       └── middleware.py     # 中间件：工具监控、模型调用日志、提示词切换
├── rag/
│   ├── vector_store.py       # 向量库连接、知识文档入库
│   └── rag_service.py        # 检索 + 大模型总结（供 rag_summarize 工具使用）
├── model/
│   └── factory.py            # 创建对话模型和向量模型，加载 .env
├── utils/
│   ├── path_tool.py          # 基于项目根目录计算绝对路径
│   ├── config_handler.py     # 读取 config/ 下的 yml 配置
│   ├── prompt_loader.py      # 读取 prompts/ 下的提示词
│   ├── file_handler.py       # 文件 MD5、目录筛选、txt/pdf 加载
│   └── logger_handler.py     # 日志：控制台 INFO 以上，文件 DEBUG 以上
├── config/
│   ├── rag.yml               # 模型名称
│   ├── chroma.yml            # 向量库、切分、检索参数
│   ├── prompts.yml           # 各提示词文件路径
│   └── agent.yml             # 外部数据文件路径
├── prompts/
│   ├── main_prompt.txt       # 智能客服系统提示词
│   ├── report_prompt.txt     # 报告写作系统提示词
│   └── rag_summarize.txt     # RAG 总结提示词
├── data/
│   ├── *.txt / *.pdf         # 知识库文档
│   └── external/records.csv  # 模拟的用户使用记录
├── requirements.txt
└── .env.example
```

运行后会自动生成以下内容（已在 `.gitignore` 中忽略）：

- `chroma_db/`：向量库数据
- `md5.txt`：已入库文件的 MD5 记录，用于避免重复入库
- `logs/`：按名称和日期生成的日志文件，如 `agent_20260930.log`

## 快速开始

以下命令都在 `pyagent/` 目录下执行。

### 1. 创建虚拟环境并安装依赖

需要 Python 3.10 及以上（开发时使用 3.13）。

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 2. 配置 API Key

```bash
cp .env.example .env
```

在 `.env` 中填入阿里云百炼的 API Key：

```
DASHSCOPE_API_KEY=你的key
```

### 3. 导入知识库

```bash
python -m rag.vector_store
```

会把 `data/` 下所有 txt、pdf 切分后存入向量库，并做一次检索测试。已经入库的文件会根据 MD5 自动跳过，新增文档后重新执行即可。

### 4. 启动智能客服

```bash
streamlit run app.py
```

可以试试这些问题：

- 「小户型适合哪些扫地机器人？」（走知识库检索）
- 「我这边天气适合用拖地功能吗？」（获取所在城市，再查天气）
- 「给我生成我的使用报告」（进入报告模式）

## 工具列表

| 工具 | 作用 | 说明 |
|---|---|---|
| `rag_summarize` | 从知识库检索资料并总结 | 内部会额外调用一次大模型 |
| `get_weather` | 查询城市天气 | 模拟数据，固定返回晴天 |
| `get_user_location` | 获取用户所在城市 | 模拟数据，随机返回 |
| `get_user_id` | 获取用户 ID | 模拟数据，从 1001–1010 中随机返回 |
| `get_current_month` | 获取当前月份 | 模拟数据，从 2025 年 12 个月中随机返回 |
| `fetch_external_data` | 查询用户某月的使用记录 | 读取 `data/external/records.csv` |
| `fill_context_for_report` | 触发报告模式 | 无实际逻辑，由中间件监测 |

## 中间件

| 中间件 | 类型 | 执行时机 | 作用 |
|---|---|---|---|
| `monitor_tool` | `@wrap_tool_call` | 每次调用工具 | 记录工具名、参数和结果；遇到 `fill_context_for_report` 时打上报告标记 |
| `log_before_model` | `@before_model` | 每次调用大模型前 | 记录当前消息条数和最新一条消息 |
| `report_prompt_switch` | `@dynamic_prompt` | 每次调用大模型前 | 根据报告标记返回 `main_prompt` 或 `report_prompt` |

## 配置说明

| 文件 | 配置项 | 说明 |
|---|---|---|
| `rag.yml` | `chat_model_name` | 对话模型 |
| | `embedding_model_name` | 向量化模型 |
| `chroma.yml` | `collection_name` | 向量库集合名 |
| | `persist_directory` | 向量库存放目录 |
| | `k` | 每次检索返回的片段数量 |
| | `data_path` | 知识库文档目录 |
| | `allow_knowledge_file_type` | 允许入库的文件类型 |
| | `chunk_size` / `chunk_overlap` | 切分长度和重叠字符数 |
| | `separators` | 切分时优先使用的分隔符，从前往后依次尝试 |
| `prompts.yml` | `*_prompt_path` | 各提示词文件路径（相对项目根目录） |
| `agent.yml` | `external_data_path` | 用户使用记录文件路径 |

## 注意事项

- **在 `pyagent/` 目录下启动。** 代码中使用 `from model.factory import ...` 这类顶层导入，需要项目根目录在 Python 的搜索路径里。另外 `persist_directory` 目前是相对当前工作目录的路径，从别的目录启动会在别处新建一个空向量库。
- `chroma_db/` 和 `md5.txt` 要一起删除或一起保留。只删向量库、保留 `md5.txt`，重新入库时会误判为"已存在"而跳过，导致检索不到任何内容。
- 更换 `embedding_model_name` 后需要同时清空 `chroma_db/` 和 `md5.txt` 重新入库，不同模型生成的向量不能混用。

## 已知限制

- **Agent 没有多轮记忆。** 页面上会显示历史消息，但每次提问 Agent 都从零开始，不记得上一个问题。可以通过给 `create_agent` 配置 `checkpointer` 并传入 `thread_id` 实现。
- **流式输出是按步骤的。** 目前使用 `stream_mode="values"`，大模型每完成一步才输出一整条消息，页面上的逐字效果是 `app.py` 里加延时模拟的。改为真正的逐字输出需要使用 `stream_mode="messages"`，并给 `ChatTongyi` 开启 `streaming=True`。
- 除 `rag_summarize` 和 `fetch_external_data` 外，其余工具返回的都是模拟数据。
