# MCP 学习示例：LangChain + FastMCP

用最小的例子跑通 MCP（Model Context Protocol）的完整链路：**MCP Server 暴露工具 → MCP Client 拉取工具 → 转成 LangChain Tool → 交给 LLM Agent 自然语言调用**。

项目里同一个 `get_today` 工具分别用三种传输方式实现（stdio / SSE / Streamable HTTP），方便对比它们的启动方式和适用场景。

## 项目结构

```
mcp/
├── mcp_server_stdio.py        # stdio 传输：本地子进程，由客户端拉起
├── mcp_server_sse.py          # SSE 传输：常驻服务，http://127.0.0.1:8000/sse
├── mcp_server_streamable.py   # Streamable HTTP：常驻服务，http://127.0.0.1:8000/mcp
├── mcp_client.py              # LangChain Agent 客户端（当前接的是 stdio 版）
└── .env                       # API Key 配置（不要提交到仓库）
```

## 环境要求

- Python >= 3.10
- 依赖：

```bash
pip install mcp langchain langgraph langchain-mcp-adapters langchain-openai python-dotenv
```

各依赖的作用：

| 包 | 作用 |
| --- | --- |
| `mcp` | MCP 官方 SDK，提供 `FastMCP` 服务端 |
| `langchain` | `create_agent` / `init_chat_model` |
| `langgraph` | `create_agent` 的运行时依赖 |
| `langchain-mcp-adapters` | `MultiServerMCPClient`，把 MCP 工具转成 LangChain Tool |
| `langchain-openai` | 走 OpenAI 兼容协议调用通义千问等模型 |
| `python-dotenv` | 读取 `.env` 中的 API Key |

## 配置

在项目根目录创建 `.env`：

```ini
DASHSCOPE_API_KEY=你的通义千问 API Key
DEEPSEEK_API_KEY=你的 DeepSeek API Key
OPENAI_API_KEY=你的 OpenAI API Key
```

`mcp_client.py` 里模型通过 OpenAI 兼容模式接入通义千问（`qwen-plus`），所以需要 `DASHSCOPE_API_KEY`。想换模型改 `init_chat_model` 的三个参数即可。

## 运行

### 1. stdio（客户端自动拉起服务，最简单）

```bash
python mcp_client.py
```

客户端通过 `MultiServerMCPClient` 以子进程方式启动 `mcp_server_stdio.py`，进程退出即结束，不需要额外开服务。

### 2. SSE

```bash
# 终端 1：启动服务
python mcp_server_sse.py

# 终端 2：把 mcp_client.py 里的连接配置换成下面这段，再运行
python mcp_client.py
```

```python
{
    "date": {
        "transport": "sse",
        "url": "http://127.0.0.1:8000/sse",
    }
}
```

### 3. Streamable HTTP

```bash
# 终端 1：启动服务
python mcp_server_streamable.py

# 终端 2：把 mcp_client.py 里的连接配置换成下面这段，再运行
python mcp_client.py
```

```python
{
    "date": {
        "transport": "streamable_http",
        "url": "http://127.0.0.1:8000/mcp",
    }
}
```

> 两种 HTTP 服务默认都占用 `8000` 端口，不要同时启动。

预期输出：

```
最终回答: 今天是2026年11月26日。
```

## 三种传输方式对比

| 传输 | 启动方式 | 连接地址 | 适用场景 |
| --- | --- | --- | --- |
| `stdio` | 客户端拉起子进程 | 无需地址 | 本地工具、单机脚本，最省事 |
| `sse` | 独立常驻进程 | `http://127.0.0.1:8000/sse` | 老版 HTTP 方案，兼容性好 |
| `streamable-http` | 独立常驻进程 | `http://127.0.0.1:8000/mcp` | MCP 新版推荐，支持无状态部署 |

## 踩坑记录

- **Windows 上必须用 `sys.executable`，不能写 `python3`。** Windows 没有 `python3` 命令，只会命中 Microsoft Store 的假 alias，子进程静默失败，看起来像“程序没反应”。
- **路径不要写死绝对路径。** 示例里用 `os.path.join(os.path.dirname(os.path.abspath(__file__)), "mcp_server_stdio.py")`，换机器也能跑。
- **stderr 里的 `INFO Processing request ...` 是正常日志**，来自 MCP 服务端，不影响结果。看到 `ListToolsRequest` 和 `CallToolRequest` 就说明工具确实被调用了。
- **`MultiServerMCPClient` 的 `transport` 字符串**：新版用的是 `streamable_http`（下划线），部分版本也接受 `streamable-http`（连字符），写错会直接报参数校验错误。
- **`.env` 不要提交。** 上传 GitHub 前确认里面是占位符或已加入 `.gitignore`。

## 参考

- [Model Context Protocol 官方文档](https://modelcontextprotocol.io)
- [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters)
