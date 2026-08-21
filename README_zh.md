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

线上服务端点为：

```text
https://api.anysearch.com/mcp
```

该端点原生使用 **Streamable HTTP**。当前版本的 OpenCode、Claude Code、Cursor、VS Code、Windsurf 和 Cline 均可直接连接，无需 SSE 或 stdio 代理。以下配置均依据各客户端当前官方文档整理。

## 安装

API key 是可选项。对于支持自定义 Header 的客户端，下面默认给出推荐的鉴权配置。如需匿名访问，只删除 `Authorization` 项，并保留 `X-Anysearch-Client`。

### OpenCode

官方文档：[MCP servers](https://opencode.ai/docs/mcp-servers/) 和 [配置文件位置](https://opencode.ai/docs/config/)。

全局配置使用 `~/.config/opencode/opencode.json`，项目配置使用项目根目录下的 `opencode.json`。Windows 全局路径为 `%USERPROFILE%\.config\opencode\opencode.json`。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "anysearch": {
      "type": "remote",
      "url": "https://api.anysearch.com/mcp",
      "enabled": true,
      "oauth": false,
      "headers": {
        "Authorization": "Bearer {env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

OpenCode 使用 `{env:NAME}` 语法引用环境变量。`"oauth": false` 可避免 API key 鉴权服务触发不必要的 OAuth 探测。

### Claude Code

官方文档：[通过 MCP 将 Claude Code 连接到工具](https://code.claude.com/docs/en/mcp)。

如需仅供当前用户使用、并在本机所有项目中生效，运行：

```bash
claude mcp add --transport http anysearch https://api.anysearch.com/mcp --scope user --header "Authorization: Bearer <your_api_key>" --header "X-Anysearch-Client: mcp/1.0.0"
```

`user` scope 会将服务写入 `~/.claude.json`，并在本机所有项目中提供。匿名访问时，省略 `Authorization` 对应的 `--header` 参数。

如需共享项目配置，在项目根目录创建 `.mcp.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Claude Code 会在 `.mcp.json` 中展开 `${VAR}` 和 `${VAR:-default}`，包括 `url` 与 `headers`。启动 Claude Code 前需设置 `ANYSEARCH_API_KEY`。首次以交互方式打开项目级 MCP 服务时需要审批。

使用以下命令验证连接：

```bash
claude mcp get anysearch
claude mcp list
```

在 Claude Code 内运行 `/mcp` 可查看服务状态和可用工具。

### Cursor

官方文档：[Model Context Protocol](https://cursor.com/docs/mcp)。

项目配置使用 `.cursor/mcp.json`，全局配置使用 `~/.cursor/mcp.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Cursor 会自动识别远程 HTTP 端点，并支持在 `url` 和 `headers` 中使用 `${env:NAME}` 环境变量插值。

### VS Code

官方文档：[MCP 配置参考](https://code.visualstudio.com/docs/agents/reference/mcp-configuration)。

运行 **MCP: Open User Configuration** 可配置全局服务，也可在工作区创建 `.vscode/mcp.json`。下面使用密码输入变量，首次启动时提示输入 key 并安全保存，避免将其提交到仓库：

```json
{
  "inputs": [
    {
      "type": "promptString",
      "id": "anysearch-api-key",
      "description": "AnySearch API key",
      "password": true
    }
  ],
  "servers": {
    "anysearch": {
      "type": "http",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${input:anysearch-api-key}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

如需匿名访问，删除 `Authorization` 项和整个 `inputs` 数组。

### Windsurf

官方文档：[Cascade MCP 集成](https://docs.devin.ai/windsurf/plugins/cascade/mcp)。

打开 **Settings > Tools > Windsurf Settings > Add Server**，或通过 **View Raw Config** 编辑 `~/.codeium/mcp_config.json`：

```json
{
  "mcpServers": {
    "anysearch": {
      "serverUrl": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer ${env:ANYSEARCH_API_KEY}",
        "X-Anysearch-Client": "mcp/1.0.0"
      }
    }
  }
}
```

Windsurf 支持在 `serverUrl`、`url` 和 `headers` 中插值环境变量。保存后刷新 MCP 服务列表。

### Cline

官方文档：[MCP](https://docs.cline.bot/mcp/mcp-overview)。

在 Cline 面板中打开 **MCP Servers > Configure > Configure MCP Servers**，也可从 **Remote Servers** 标签页添加托管端点。必须显式使用 `streamableHttp` 类型；省略时会为兼容旧配置而回退到 SSE。

```json
{
  "mcpServers": {
    "anysearch": {
      "type": "streamableHttp",
      "url": "https://api.anysearch.com/mcp",
      "headers": {
        "Authorization": "Bearer <your_api_key>",
        "X-Anysearch-Client": "mcp/1.0.0"
      },
      "disabled": false,
      "autoApprove": []
    }
  }
}
```

Cline CLI 的配置文件是 `~/.cline/mcp.json`；运行 `cline mcp` 可打开交互式 MCP 向导。

## 客户端速查表

| 客户端 | 官方接入方式 | 配置位置 | 可直连 Streamable HTTP？ |
| ------ | ------------ | -------- | ------------------------ |
| OpenCode | 远程 MCP 配置 | `~/.config/opencode/opencode.json` 或项目 `opencode.json` | 是 |
| Claude Code | 远程 HTTP MCP | 用户级 `~/.claude.json` 或项目级 `.mcp.json` | 是 |
| Cursor | 远程 MCP 配置 | `.cursor/mcp.json` 或 `~/.cursor/mcp.json` | 是 |
| VS Code | HTTP MCP 配置 | 用户 MCP 配置或 `.vscode/mcp.json` | 是 |
| Windsurf | 远程 HTTP MCP | `~/.codeium/mcp_config.json` | 是 |
| Cline | 远程 Streamable HTTP | Cline MCP 设置或 `~/.cline/mcp.json` | 是 |

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
