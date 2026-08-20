# AnySearch MCP Server

> [English](./README.md) | 简体中文

统一实时搜索 MCP 服务器，支持通用网页搜索、垂直领域搜索、并行批量搜索和整页 URL 内容提取。

## 功能

- **通用网页搜索** —— 开放式自然语言查询
- **垂直领域搜索** —— 支持金融、学术、安全、法律、代码等领域的结构化查询
- **并行批量搜索** —— 在一次调用中执行多个独立查询
- **URL 内容提取** —— 获取并将完整页面内容提取为 Markdown
- **匿名访问** —— 无需 API key 即可使用（速率限制较低）

## API Key 配置

API key **是可选项，但建议配置**。即使没有 key，所有功能仍可通过匿名访问使用，但速率限制较低。

### 注册获取 API Key（推荐）

智能体可以在**一次调用**中完成用户注册并获取 API key —— 无需验证码，无需手动注册。向用户索取一个**真实邮箱地址**：它将作为账户用户名。

```bash
curl -s -X POST "https://api.anysearch.com/v1/auth/email/register" \
  -H "Content-Type: application/json" \
  -d '{"email": "you@example.com"}'
```

成功响应（`code: 0`）会返回账户信息和仅显示一次的明文 API key：

```json
{
  "code": 0,
  "message": "success",
  "data": {
    "username": "you@example.com",
    "email": "you@example.com",
    "login_url": "https://www.anysearch.com/login",
    "api_key": {
      "id": "key_xxxxxxxx",
      "key": "as_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
      "key_prefix": "as_sk_xxxxxx...",
      "name": "default",
      "rate_limit": 100,
      "quota_limit": 0,
      "expires_at": null,
      "created_at": "2026-06-23T10:23:00Z"
    }
  }
}
```

成功后，智能体**必须**：

1. 将 `data.api_key.key` 写入 MCP 服务器配置，作为 `Authorization: Bearer <key>` 的值（和/或设置 `ANYSEARCH_API_KEY`）—— 它只显示一次（之后也可从控制台重新获取）。
2. 告知用户其用户名（= 邮箱）和 `login_url`。

错误处理（出错时 `code` 始终为 `-1`；根据 `message` 字符串分支处理）：

| message | 处理方式 |
| ------- | -------- |
| `Invalid email address.` | 请用户重新输入邮箱 |
| `email_already_registered` | 邮箱已被注册 —— 告知用户在 `login_url` 登录；**不要**重试 |
| 包含 `Rate limited` | 从 message 中读取重试秒数（例如 `"Rate limited, retry after 300 seconds."`），等待后重试 |
| 以 `Key creation failed.` 开头 | 账户已创建，但 key 创建失败 —— 从 message 中提取邮箱和 URL（`"Key creation failed. Your account <email> was created; sign in at <url>."`），告知用户在该地址登录并手动创建 key |
| `Internal server error.` | 稍后重试，或回退到匿名访问 |

> 该邮箱**必须真实且可接收邮件**。

### 获取 API Key

也可访问 https://anysearch.com/console/api-keys 创建免费 API key。

### Key 优先级

| 优先级 | 来源 |
| -------- | ---- |
| 1（最高） | `--api_key` CLI 标志 / `Authorization` Header |
| 2 | 环境变量 `ANYSEARCH_API_KEY` |
| 3 | `.env` 文件（`ANYSEARCH_API_KEY=<key>`） |
| 4 | 匿名访问（速率限制较低） |

### Key 行为

| 场景 | 行为 |
| ---- | ---- |
| 无 key | 使用匿名访问（速率限制较低） |
| 有 key | 通过 `Authorization: Bearer <key>` Header 发送，享有更高的速率限制 |
| Key 额度耗尽，返回自动注册的 key | 智能体应请求用户确认，然后持久化保存新 key |
| Key 额度耗尽，未返回新 key | 告知用户，并建议配置新 API key |

## MCP 传输

AnySearch MCP 服务器**原生支持 Streamable HTTP** 传输（MCP 规范 2025-03-26）。SSE 和 stdio 客户端可通过代理连接。

| 传输方式 | 原生支持？ | 适用对象 |
| -------- | -------- | -------- |
| **Streamable HTTP** | 是 | OpenCode、Claude Desktop (2025.6+)、Web 客户端 |
| **SSE** | 通过代理 | Cursor、Windsurf |
| **stdio** | 通过代理 | Claude Desktop（旧版）、VS Code Copilot、Cline |

## 安装

### Streamable HTTP（推荐 —— 无需代理）

适用于支持 Streamable HTTP 传输（MCP 规范 2025-03-26+）的智能体：

**OpenCode** (v1.x+ / v0.1.x+)：

配置文件位置取决于 OpenCode 版本。运行 `opencode -v` 查看版本。

| 版本 | 全局配置路径 | 项目配置路径 |
| ---- | ------------ | ------------ |
| **1.x+**（当前版本） | `~/.config/opencode/opencode.json` | `opencode.json` 或 `.opencode/opencode.json` |
| **0.1.x ~ 0.15.x** | `~/.config/opencode/opencode.json` | `opencode.json` |
| **0.0.x**（旧版 Go） | `~/.opencode.json` | `.opencode.json` |

> **Windows**：将 `~/.config/opencode/` 替换为 `%USERPROFILE%\.config\opencode\`。

对于 v1.x+ 和 v0.1.x+（MCP key：`mcp`）：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "anysearch": {
      "type": "remote",
      "url": "https://api.anysearch.com/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

<details>
<summary>旧版 Go (0.0.x) —— MCP key：<code>mcpServers</code></summary>

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "sse",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

> 旧版 Go 不原生支持 Streamable HTTP。请通过代理使用 SSE 或 stdio。

</details>

**Claude Desktop** (2025.6+, `claude_desktop_config.json`)：

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "streamable-http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

> 如果没有 API key，只删除 `Authorization` 这一行，但**保留** `X-Anysearch-Client`。服务器将自动使用匿名访问。

### stdio（通过代理）

适用于仅支持 stdio 传输的智能体。有两种代理方案：

#### 方案 A：mcp-remote（推荐）

[`mcp-remote`](https://github.com/geelen/mcp-remote) —— 可自动检测 Streamable HTTP，配置最简单：

**Claude Desktop** (`claude_desktop_config.json`)：

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

**VS Code Copilot** (`.vscode/mcp.json`)：

```json
{
  "servers": {
    "anysearch": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

**Cline**（VS Code 设置）：

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--header",
        "Authorization: Bearer ${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

> 如果没有 API key，只省略 `"Authorization: Bearer ..."` 这组 `--header` 参数；**保留** `X-Anysearch-Client` 的 `--header`。

#### 方案 B：supergateway

[`supergateway`](https://github.com/supercorp-ai/supergateway) —— 提供更多传输选项，支持 SSE 输出：

**Claude Desktop** (`claude_desktop_config.json`)：

```json
{
  "mcpServers": {
    "anysearch": {
      "command": "npx",
      "args": [
        "-y",
        "supergateway",
        "--streamableHttp",
        "https://api.anysearch.com/mcp",
        "--header",
        "X-Anysearch-Client: mcp/1.0.0",
        "--oauth2Bearer",
        "${ANYSEARCH_API_KEY}"
      ]
    }
  }
}
```

> 如果没有 API key，省略 `"--oauth2Bearer"` 和 key 参数。

### SSE（通过代理）

适用于仅支持 SSE 传输的智能体（Cursor、Windsurf）。需要运行本地 SSE 代理服务器：

#### 启动代理

```bash
npx -y supergateway \
  --streamableHttp https://api.anysearch.com/mcp \
  --outputTransport sse \
  --port 8000 \
  --header "X-Anysearch-Client: mcp/1.0.0" \
  --oauth2Bearer <your_api_key>
```

> 如果没有 API key，省略 `--oauth2Bearer` 标志。

然后配置你的智能体：

**Cursor** (`.cursor/mcp.json`)：

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "sse",
      "url": "http://localhost:8000/sse"
    }
  }
}
```

**Windsurf** (`~/.codeium/windsurf/mcp_config.json`)：

```json
{
  "mcpServers": {
    "anysearch": {
      "serverUrl": "http://localhost:8000/sse"
    }
  }
}
```

> 智能体运行期间，SSE 代理必须保持运行。可考虑将其作为后台服务运行。

## 智能体速查表

| 智能体 | 传输方式 | 配置位置 | 需要代理？ | 代理工具 |
| ------- | -------- | -------- | -------- | -------- |
| OpenCode (v1.x+) | Streamable HTTP | `~/.config/opencode/opencode.json` 或项目中的 `opencode.json` | 否 | — |
| Claude Desktop (2025.6+) | Streamable HTTP | `claude_desktop_config.json` | 否 | — |
| Claude Desktop（旧版） | stdio | `claude_desktop_config.json` | 是 | `mcp-remote` |
| Cursor | SSE | `.cursor/mcp.json` | 是 | `supergateway` |
| VS Code Copilot | stdio | `.vscode/mcp.json` | 是 | `mcp-remote` |
| Windsurf | SSE | `mcp_config.json` | 是 | `supergateway` |
| Cline | stdio | VS Code 设置 | 是 | `mcp-remote` |

## 可用工具

### `search`

执行通用或垂直领域搜索查询。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `query` | string | 是 | 自然语言搜索查询。每次调用只包含一个意图 |
| `domain` | string | 否 | 垂直领域（例如 `finance`、`academic`、`security`）。必须来自 `get_sub_domains` 枚举 |
| `sub_domain` | string | 否 | 子领域路由 key（例如 `finance.us_stock`）。必须来自 `get_sub_domains` 输出 |
| `sub_domain_params` | object | 否 | 来自 `get_sub_domains` params 列的结构化参数。**绝不要**臆造参数值 |
| `max_results` | integer | 否 | 1–10，默认为 10 |

### `get_sub_domains`

查询垂直领域目录。**使用 domain 进行任何搜索前都必须先调用** —— 返回有效的 sub_domains 及其参数 schema。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `domain` | string | 二选一 | 要查询的单个领域 |
| `domains` | string[] | 二选一 | 批量查询最多 5 个领域（推荐 —— 覆盖范围更广） |

返回 Markdown 表格：`sub_domain | description | params`

### `batch_search`

并行执行 1–5 个独立搜索查询。单个查询失败不会阻塞其他查询。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `queries` | object[] | 是 | 1–5 个查询对象，每个对象的字段与 `search` 相同 |

### `extract`

从 URL 获取完整页面内容并以 Markdown 返回。内容超过 50,000 个字符时会被截断。仅支持 HTML 页面。

| 参数 | 类型 | 必填 | 说明 |
| ---- | ---- | ---- | ---- |
| `url` | string | 是 | 目标 URL（`http://` 或 `https://`） |
