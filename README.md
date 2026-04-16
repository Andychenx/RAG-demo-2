# RAG-demo-2
# ReAct Agent 智能客服（扫地机器人领域）
## 📖 项目简介
本项目是一个基于 **`LangChain + LangGraph`** 构建的 **ReAct 架构智能客服 Agent**，专为扫地机器人 / 扫拖一体机产品提供专业问答和个性化使用报告生成服务。Agent 具备自主推理、工具调用、动态提示词切换能力，可同时处理开放式知识问答和结构化报告生成两种复杂任务。

## ✨ 核心特性
 - **ReAct 推理与行动**：遵循「思考 → 行动 → 观察 → 再思考」闭环，自主判断信息缺口并调用工具
 - **7 个专用工具**：
   - `rag_summarize`：从本地知识库检索专业资料
   - `get_weather`：实时天气查询[[wttr.in]]https://wttr.in/
   - `get_user_location` / `get_user_id` / `get_current_month`：获取用户上下文
   - `fetch_external_data`：读取 CSV 用户使用记录（清洁效率、耗材状态等）
   - `fill_context_for_report`：触发报告生成上下文切换

 - **动态提示词切换**：通过中间件根据调用工具自动切换“普通问答 Prompt”和“报告生成 Prompt”
 - **RAG 知识库**：支持 TXT/PDF 文档加载、MD5 去重、Chroma 向量存储、语义检索
 - **全链路可观测**：中间件记录工具调用入参、执行结果、耗时及模型输入输出
 - **个性化报告**：结合用户历史数据生成 Markdown 格式的使用报告与保养建议

## 🛠️ 技术栈
|类别	|技术|
|-------|----|
|Agent 框架	|LangChain, LangGraph (create_agent)|
|大模型	|通义千问 qwen3-max|
|嵌入模型|	DashScope text-embedding-v4|
|向量数据库|	Chroma (本地持久化)|
|文档加载|	PyPDFLoader, TextLoader|
|文本分割|	RecursiveCharacterTextSplitter|
|配置管理|	PyYAML|
|日志	|Python logging (控制台 + 按日期滚动文件)|
|外部 API|	[[wttr.in]]https://wttr.in/|

## 📦 安装与配置
1. 克隆仓库
```
git clone https://github.com/your-repo/react-agent-robot.git
cd react-agent-robot
```
2. 创建虚拟环境
```
python -m venv venv
source venv/bin/activate      # Linux/Mac
venv\Scripts\activate         # Windows
```

3. 安装依赖
```
pip install langchain langgraph langchain-community langchain-chroma chromadb dashscope pypdf pyyaml requests
```

4. 配置 API Key
**编辑 `config/rag.yml`，填入你的通义千问 API Key：**
```
api_key: "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

5. 准备知识库文档
**将扫地机器人相关的 TXT/PDF 文件放入 `data/` 目录，然后加载向量库：**
```
python -m rag.vector_store
```

6. 配置外部用户数据（报告功能）
编辑 `data/external/records.csv`，格式如下（示例已提供）：
```
"用户ID","特征","清洁效率","耗材","对比","时间"
"1001","65㎡公寓 | 单身 | 木地板","覆盖率:85%...","主刷寿命:剩余60天...","优于65%同面积用户...","2025-01"
...
```

## 🚀 快速开始
**运行 Agent 交互示例**
```
from agent.react_agent import ReactAgent

agent = ReactAgent()

# 普通问答
question = "扫地机器人在潮湿环境下如何保养？"
for chunk in agent.execute_stream(question):
    print(chunk, end="", flush=True)

# 生成使用报告
question = "生成我的使用报告"
for chunk in agent.execute_stream(question):
    print(chunk, end="", flush=True)
```

**命令行测试**
```
python -m agent.react_agent
```

## 📁 项目结构
```
.
├── agent/                      # Agent 核心模块
│   ├── react_agent.py          # ReAct Agent 主类
│   └── tools/                  # 自定义工具
│       ├── agent_tools.py      # 7 个工具实现
│       └── middleware.py       # 中间件（监控、日志、动态提示词）
├── rag/                        # RAG 模块
│   ├── rag_service.py          # RAG 总结服务
│   ├── vector_store.py         # 向量库管理（加载/检索）
│   └── __init__.py
├── model/                      # 模型工厂
│   └── factory.py              # 统一生成 LLM 和 Embeddings
├── utils/                      # 工具函数
│   ├── config_handler.py       # YAML 配置加载
│   ├── file_handler.py         # 文件 MD5、PDF/TXT 加载
│   ├── logger_handler.py       # 日志配置
│   ├── path_tool.py            # 绝对路径转换
│   └── prompt_loader.py        # 提示词加载
├── config/                     # 配置文件
│   ├── agent.yml               # Agent 配置（外部数据路径）
│   ├── chroma.yml              # 向量库参数
│   ├── prompts.yml             # 提示词文件路径
│   └── rag.yml                 # 模型 API 配置
├── prompts/                    # 提示词模板
│   ├── main_prompt.txt         # 主控 Prompt（ReAct 规则）
│   ├── rag_summarize.txt       # RAG 总结 Prompt
│   └── report_prompt.txt       # 报告生成 Prompt
├── data/                       # 数据目录
│   ├── external/               # 外部 CSV 数据（用户记录）
│   └── 扫地机器人*.txt/pdf     # 知识库原始文档
├── chroma_db/                  # 向量库持久化目录（自动生成）
├── logs/                       # 日志目录（自动生成）
├── md5.txt                     # 已加载文件的 MD5 记录
└── README.md
```

## 🧠 ReAct 思考流程示例
**用户问**：“扫地机器人在我所在的地区的气温下如何保养？”

1. **思考**：需要先知道用户所在城市 → 调用 get_user_location

2. **行动**：get_user_location() → 返回 “南京”

3. **观察**：已获得城市

4. **思考**：需要知道南京当前天气 → 调用 get_weather

5. **行动**：get_weather(city="南京") → 返回 “局部多云，19°C”

6. **观察**：已获得温度

7. **思考**：需要检索 19°C 下的保养建议 → 调用 rag_summarize

8. **行动**：rag_summarize(query="扫地机器人19度保养") → 返回专业建议

9. **最终回答**：整合信息生成自然语言回答

## 📊 报告生成强制流程
当用户请求生成使用报告时，Agent 严格按以下顺序执行：

1. `fill_context_for_report()` → 触发提示词切换（中间件设置 `context["report"]=True`）

1. `get_user_id()` → 获取用户 ID

3. `get_current_month()` → 获取当前月份（或使用用户指定月份）

4. `fetch_external_data(user_id, month)` → 拉取 CSV 记录

5. `rag_summarize(query="电池保养 边刷更换")` → 补充专业建议

6. 使用报告专用 Prompt 生成 Markdown 格式报告


## 🧩 工具详解
|工具名称	|功能	|入参	|说明|
|-----------|-------|-------|----|
|rag_summarize	|向量库检索	|query (str)	|从知识库获取专业回答|
|get_weather	|实时天气	|city (str)	|调用 [[wttr.in]]https://wttr.in/ |
|get_user_location	|用户城市|	无	|随机返回模拟城市|
|get_user_id	|用户 ID	|无	|随机返回 1001~1010|
|get_current_month	|当前月份|	无	|随机返回 2025-01 ~ 2025-12|
|fetch_external_data	|拉取使用记录|	user_id, month	|从 CSV 读取|
|fill_context_for_report|	触发报告模式|	无	|修改运行时上下文|

## ⚙️ 配置参数
**`config/chroma.yml`**
```
collection_name: agent          # 向量集合名
persist_directory: chroma_db    # 持久化目录
k: 3                            # 检索返回文档数
chunk_size: 200                 # 文本分块大小
chunk_overlap: 20               # 块重叠长度
```

**`config/agent.yml`**
```
external_data_path: data/external/records.csv
```

## 📝 日志系统
 - 日志目录：logs/
 - 文件命名：agent_YYYYMMDD.log
 - 同时输出到控制台（INFO 级别）和文件（DEBUG 级别）
 - 记录内容：工具调用名称/入参/结果、模型调用前消息状态、知识库加载详情、错误堆栈

## 🤝 贡献
**欢迎提交 Issue 和 Pull Request。**
**本项目来源：https://www.bilibili.com/video/BV1yjz5BLEoY?spm_id_from=333.788.videopod.episodes&vd_source=48803d014274dcfef4922462b7b0dbc6**
**黑马程序员B站主页：https://space.bilibili.com/37974444?spm_id_from=333.788.upinfo.detail.click**
