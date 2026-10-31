# 在 ThinkRail 中管理 MCP 服务器

> 中文译文，原文见 [`SKILL.md`](./SKILL.md)。英文原文是唯一权威来源，二者不一致时以原文为准。

ThinkRail 使用 pi 内置的 MCP 支持，但没有 `pi` 可执行文件。绝不运行 `pi mcp add`、`pi mcp login`
或其他 `pi` 命令；改动配置文件，并把一切需要界面的操作指向 **Settings › MCP servers**。

## 服务器配置在哪里（Where servers are configured）

- **用户级** —— pi 的 agent 目录，通常是 `~/.pi/agent/mcp.json`：在每个项目中都可用。
- **本仓库** —— 工作树中的 `.pi/mcp.json`，由 git 跟踪。在这里定义的服务器在用户在 Settings 中批准
  之前不会运行，之后对其条目的任何改动都需要再次批准。只有当用户信任该项目时它才会加载。

两个文件都使用 pi 的格式：

```json
{
  "mcpServers": {
    "docs": { "url": "https://example.com/mcp" },
    "browser": { "command": "npx", "args": ["-y", "@playwright/mcp@0.0.83"] }
  }
}
```

- 在 `args` 中钉住包版本；绝不用 `@latest`。
- 绝不把密钥写进这两个文件中的任何一个。用 `${ENV_VAR}`（取自宿主环境）或 `!command`
  （命令的输出，例如 `!gh auth token`）来引用它，或者让用户自行登录。
- `exposure`：`deferred`（这里的默认值 —— 先用 `tool_search` 加载该服务器的工具）、
  `direct`（少数始终有用的工具），或 `hidden`。`codemode` 按 `deferred` 处理。
- 个人服务器优先放在用户级文件里；只有当用户希望它与仓库共享时，才把服务器放进 `.pi/mcp.json`。

## GitHub

GitHub 是一份配方，而不是预设：它的托管服务器，用用户的 GitHub CLI 登录来鉴权。把它加到用户级文件
（Settings › MCP servers 在 Paste JSON 下提供同样的条目）：

```json
{
  "mcpServers": {
    "github": {
      "url": "https://api.githubcopilot.com/mcp/readonly",
      "headers": { "Authorization": "!t=$(gh auth token) && echo \"Bearer $t\"" }
    }
  }
}
```

- `/readonly` 是 GitHub 文档中给出的只读模式（等同于一个 `X-MCP-Readonly: true` 头）。
  只有当用户要求能改动数据的工具时，才用 `https://api.githubcopilot.com/mcp/`。
- 该头命令在每次聊天连接时运行。当 `gh` 处于登出状态时它会失败，而不是发送一个空 token；
  用户需要在终端里运行 `gh auth login`。
- 未验证：GitHub 是否对每个工具集都接受 GitHub CLI token 的 scope。如果服务器拒绝它，
  就让用户在某个环境变量中提供 token：`"Authorization": "Bearer ${GITHUB_TOKEN}"`。

## 编辑之后（After editing）

宿主会自行重载空闲的聊天，忙碌的聊天会在空闲时接收这次改动。当某一步属于用户时，告诉他们：
批准一个仓库级服务器，或为一个使用 OAuth 的服务器点 **Sign in**（两者都在 Settings › MCP servers）。
不要让他们重启 ThinkRail。

## 使用 MCP 工具（Using MCP tools）

- 工具的命名形式是 `mcp__<server>__<tool>`。对于 `deferred` 服务器，调用 `tool_search` 来加载它们。
- 子 agent 无法调用 MCP 工具：自己调用它们，并把子 agent 需要的东西放进它的任务里。
- 对于可能改动数据的调用，ThinkRail 会先询问用户。被拒绝或被取消的调用对该次调用是终局的；
  未经先询问用户，不要重试它。
