# TechScout 技术栈与零搜索 API 费用方案

## 1. 成本原则

本项目的硬性约束是：**只允许大模型 Token 产生费用，其他组件和服务必须免费。**

这里的“免费”指软件许可免费、项目不调用按量收费的第三方 API。仍然需要用户已有的电脑、网络连接和本地运行资源；目标网站也可能限制自动访问。

以下方案不能作为项目默认依赖：

- 只有限时试用或有限免费额度、超额后可能收费的服务。
- 注册时要求开通自动扣费才能使用的服务。
- 把搜索、追踪、存储或 Agent 执行托管到计费云平台的方案。
- 无法确认许可证或计费规则的依赖。

如未来确实需要其中任何一项，必须由用户明确修改本约束后才能引入。

## 2. 技术栈与费用

| 分类 | 技术 | 用途 | 单独服务费 |
| --- | --- | --- | --- |
| 开发语言 | Python 3.11+ | 项目开发语言 | 无 |
| 依赖管理 | uv | 虚拟环境、安装依赖和锁文件 | 无 |
| Agent 框架 | deepagents | 规划、文件系统、子代理和上下文管理 | 无，MIT 开源 |
| 底层框架 | LangChain / LangGraph | 工具调用和 Agent 运行时，由依赖带入 | 无，开源 |
| 模型接入 | langchain-openai | 调用用户选择的大模型 | 仅模型 Token 费用 |
| 搜索发现 | DDGS | 无 API Key 的元搜索 | 无按请求 API 费用 |
| 网页读取 | httpx | 获取公开网页正文 | 无按请求 API 费用 |
| HTML 清理 | markdownify | 将 HTML 转成 Markdown | 无 |
| 配置 | pydantic-settings、python-dotenv | 环境变量读取和校验 | 无 |
| CLI | Typer、Rich | 命令行和进度展示 | 无 |
| 测试 | pytest、pytest-cov | 自动化测试 | 无 |
| 代码质量 | Ruff | 格式化和 Lint | 无 |
| 产物存储 | 本地 Markdown 文件 | brief、笔记和报告 | 无云存储费 |

首版不会启用 LangSmith 云端追踪、云端 Sandbox、托管部署或任何自动付费功能。

## 3. 联网是如何实现的

Deep Agents 不会自动拥有互联网访问能力。我们会把两个受限制的 Python 函数注册为工具：

```text
用户问题
   ↓
Deep Agent 制订调研计划
   ↓
调用 internet_search(query)
   ↓
DDGS 从公开搜索来源获得标题、摘要、URL
   ↓
按需调用 fetch_webpage(url)
   ↓
httpx 获取公开网页，markdownify 清理 HTML
   ↓
Deep Agent 分析证据并写入本地 report.md
```

模型只可以调用我们明确注册的工具。搜索次数、下载大小、超时和目标地址限制都由 Python 程序强制执行，而不是只写在提示词里。

## 4. 两个联网工具

### `internet_search`

职责是发现资料，而不是生成答案。

- 使用 `DDGS().text(...)` 搜索。
- 不需要注册账号或配置搜索 API Key。
- 每次最多返回 5 条结果。
- 输出统一为标题、URL、摘要。
- 捕获超时、限流和上游异常。

示意代码：

```python
from ddgs import DDGS


def internet_search(query: str, max_results: int = 5) -> list[dict[str, str]]:
    limit = min(max(max_results, 1), 5)
    results = DDGS(timeout=8).text(query, max_results=limit)
    return [
        {
            "title": item.get("title", ""),
            "url": item.get("href", ""),
            "summary": item.get("body", ""),
        }
        for item in results
    ]
```

### `fetch_webpage`

职责是核验某个搜索结果的正文。

1. 只接受 `http` 和 `https` URL。
2. 拒绝本机、内网和云元数据地址，降低 SSRF 风险。
3. 设置短超时、重定向上限、User-Agent 和响应体上限。
4. 只解析允许的文本内容类型。
5. 用 `markdownify` 转换 HTML，并截断过长内容。
6. 遵守网站访问限制；被拒绝时跳过，不尝试绕过。

## 5. 免费方案的限制

DDGS 不按请求收费，但不能理解为“无限且有 SLA”：

- 它依赖公开搜索来源，上游页面变化时可能暂时失效。
- 高频请求可能遇到验证码、限流或 IP 封禁。
- 搜索结果质量和地域覆盖可能不稳定。
- 某些网站禁止抓取或只能由浏览器访问。
- 应低频使用、限制并发，并缓存同一调研中的重复搜索。

这非常适合学习项目和个人低频使用，但不应直接承诺生产级可用性。

## 6. 为什么不使用 Tavily

Tavily 的 Python SDK 本身可以免费安装，但 Tavily Search 是商业 API。目前有每月免费额度，超出后存在付费选项。因此它不符合“除模型 Token 外不产生第三方服务账单”的严格约束，已从 `requirements.txt` 移除。

如果未来主动接受搜索费用，可以把 Tavily 作为可选 Provider；Agent 上层流程不需要重写。

## 7. 可选的完全自控方案

若 DDGS 不够稳定，可以自建 SearXNG：

- SearXNG 是自由开源的元搜索引擎，并提供 HTTP Search API。
- 自建实例不需要购买搜索 API。
- 但需要 Docker 或 Linux 环境、内存、网络和日常维护。
- 上游搜索来源仍可能限流，因此自建不等于绝对稳定。

为了保持第一个 Sprint 简单，当前不引入 SearXNG。

## 8. 配置方式

后续 `.env.example` 只需要模型相关配置：

```dotenv
MODEL_NAME=openai:your-model-name
OPENAI_API_KEY=replace-me
WORKSPACE_DIR=./workspace

# 明确关闭云端追踪，避免使用可计费的可观测服务
LANGSMITH_TRACING=false
```

不再需要 `TAVILY_API_KEY` 或其他搜索服务密钥。

## 9. 安装方式

```powershell
uv venv
uv pip install -r requirements.txt
```

`requirements.txt` 保存直接依赖范围。项目初始化后生成 `uv.lock`，锁定实际安装的完整版本。
