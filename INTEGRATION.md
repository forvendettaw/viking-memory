---
name: openviking-integration
description: 将 OpenViking 集成到 OpenClaw 作为向量化记忆系统的完整指南
---

# OpenViking + OpenClaw 集成指南

将 OpenViking 作为 OpenClaw 的长期记忆系统，实现语义向量搜索。

## 背景

| 组件 | 角色 |
|------|------|
| OpenClaw | AI Agent 框架 |
| OpenViking | 向量化记忆数据库 (支持豆包 embedding) |
| viking-memory skill | 两者桥梁 |

## 步骤 1: 安装 OpenViking

```bash
# 创建虚拟环境
python3 -m venv ~/.openviking-venv

# 激活
source ~/.openviking-venv/bin/activate

# 安装
pip install openviking
```

## 步骤 2: 配置 OpenViking

创建 `~/.openviking/ov.conf`:

```json
{
  "embedding": {
    "dense": {
      "api_base": "https://ark.cn-beijing.volces.com/api/v3",
      "api_key": "你的豆包API Key",
      "provider": "volcengine",
      "dimension": 1024,
      "model": "doubao-embedding-vision-250615"
    }
  }
}
```

创建 CLI 配置 `~/.openviking/ovcli.conf`:

```json
{
  "url": "http://127.0.0.1:18790"
}
```

## 步骤 3: 启动服务

```bash
# 启动服务
~/.openviking-venv/bin/openviking-server --host 127.0.0.1 --port 18790

# 测试
curl http://127.0.0.1:18790/health
```

配置开机自启 (macOS):

```xml
<!-- ~/Library/LaunchAgents/com.openviking.server.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openviking.server</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/scott/.openviking-venv/bin/openviking-server</string>
        <string>--host</string>
        <string>127.0.0.1</string>
        <string>--port</string>
        <string>18790</string>
        <string>--config</string>
        <string>/Users/scott/.openviking/ov.conf</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>
</dict>
</plist>
```

加载:
```bash
launchctl load ~/Library/LaunchAgents/com.openviking.server.plist
```

## 步骤 4: 创建 viking-memory Skill

在 `~/.openclaw/workspace/skills/viking-memory/` 创建:

### SKILL.md

```markdown
---
name: viking-memory
description: OpenViking 长期记忆系统。用于语义检索用户偏好、历史对话、重要信息等。
---

# Viking Memory

基于 OpenViking 的向量化记忆系统，提供语义搜索能力。

## 功能

- 语义搜索 - 用自然语言搜索相关记忆
- 添加记忆 - 将重要信息存入长期记忆
- 读取内容 - 获取记忆详细内容
```

### index.js

```javascript
"use strict";

const VIKING_API_URL = 'http://127.0.0.1:18790';

async function vikingRequest(endpoint, body) {
    const response = await fetch(`${VIKING_API_URL}${endpoint}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(body),
    });
    return response.json();
}

async function searchHandler(args) {
    const { query, limit = 5, threshold = 0.3 } = args;
    const result = await vikingRequest('/api/v1/search/find', { query, limit, threshold });

    if (!result.result || result.result.total === 0) {
        return { content: [{ type: 'text', text: '没有找到相关记忆。' }] };
    }

    const memories = result.result.resources || [];
    const text = memories.map((m, i) =>
        `${i + 1}. [${m.uri}]\n   ${m.abstract || '(无内容)'}\n   相关度: ${(m.score * 100).toFixed(0)}%`
    ).join('\n\n');

    return { content: [{ type: 'text', text: `找到 ${memories.length} 条相关记忆:\n\n${text}` }] };
}

async function addMemoryHandler(args) {
    const { uri, content } = args;
    const result = await vikingRequest('/api/v1/resources', { uri, content });
    return { content: [{ type: 'text', text: `已保存记忆: ${uri}` }] };
}

const skill = {
    name: 'viking-memory',
    version: '1.0.0',
    description: 'OpenViking 长期记忆系统',
    actions: {
        search: {
            description: '语义搜索记忆',
            parameters: {
                type: 'object',
                properties: {
                    query: { type: 'string', description: '搜索查询' },
                    limit: { type: 'number', default: 5 },
                },
                required: ['query'],
            },
            handler: searchHandler,
        },
        add_memory: {
            description: '添加记忆',
            parameters: {
                type: 'object',
                properties: {
                    uri: { type: 'string', description: '记忆 URI' },
                    content: { type: 'string', description: '记忆内容' },
                },
                required: ['uri', 'content'],
            },
            handler: addMemoryHandler,
        },
    },
};

module.exports = skill;
```

### _meta.json

```json
{
  "slug": "viking-memory",
  "version": "1.0.0"
}
```

## 步骤 5: 重启 OpenClaw

```bash
openclaw gateway --force
```

## 步骤 6: 发布到 ClawHub (可选)

```bash
clawhub publish ~/.openclaw/workspace/skills/viking-memory \
  --slug viking-memory --name "Viking Memory" --version 1.0.0
```

## API 参考

### 语义搜索

```bash
curl -s -X POST http://127.0.0.1:18790/api/v1/search/find \
  -H "Content-Type: application/json" \
  -d '{"query": "用户偏好", "limit": 5}'
```

### 添加资源

```bash
curl -s -X POST http://127.0.0.1:18790/api/v1/resources \
  -H "Content-Type: application/json" \
  -d '{"uri": "viking://user/preferences/咖啡", "content": "用户喜欢喝拿铁"}'
```

### 读取内容

```bash
curl -s -X POST http://127.0.0.1:18790/api/v1/content/read \
  -H "Content-Type: application/json" \
  -d '{"uri": "viking://user/preferences/咖啡"}'
```

## 使用场景

1. **自动召回** - Agent 在对话开始时搜索相关记忆
2. **手动搜索** - 用户询问"我之前提到的..."
3. **自动保存** - 可以集成 hook 自动保存重要信息

## 优势

- 支持豆包 embedding (免费/低成本)
- 向量语义搜索
- 层级记忆结构
- Agent 原生集成

## 进阶：自动记忆捕获

可以通过 OpenClaw Hook 实现自动记忆捕获。

### 创建 session-end Hook

在 `~/.openclaw/workspace/hooks/viking-capture/` 创建 HOOK.md:

```markdown
---
name: viking-capture
description: 自动将会话摘要保存到 OpenViking
events:
  - session:end
---

# Viking Capture Hook

在会话结束时自动提取关键信息并保存到 OpenViking。

## 工作流程

1. 提取对话摘要
2. 识别重要信息（偏好、决策、事实）
3. 保存到 OpenViking
```

### 脚本示例

```javascript
// hooks/viking-capture/index.js
const VIKING_API = 'http://127.0.0.1:18790';

async function onSessionEnd(context) {
    const { session, messages } = context;

    // 提取摘要
    const summary = messages.slice(-10).map(m =>
        `${m.role}: ${m.content?.slice(0, 100)}`
    ).join('\n');

    // 保存到 OpenViking
    await fetch(`${VIKING_API}/api/v1/resources`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
            uri: `viking://agent/sessions/${session.id}`,
            content: summary
        })
    });
}

module.exports = { onSessionEnd };
```

## 成本对比

| 方案 | Embedding 成本 |
|------|---------------|
| 豆包 (100万 tokens/月免费) | ¥0 |
| OpenAI text-embedding-3-small | ~$0.02/1M |

## 常见问题

### Q: OpenViking 服务启动失败？

检查：
```bash
# 查看日志
tail -f ~/.openviking/server.error.log

# 检查端口占用
lsof -i :18790
```

### Q: 搜索不到结果？

确认已添加记忆：
```bash
curl -s http://127.0.0.1:18790/api/v1/system/status
```

### Q: API 返回 404？

确认服务正常运行：
```bash
curl http://127.0.0.1:18790/health
```

## 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    OpenClaw Agent                       │
│  ┌─────────┐  ┌──────────┐  ┌──────────────────────┐   │
│  │ 短期记忆 │→│ 中期摘要 │→│   长期记忆 (Viking)  │   │
│  │ Sliding │  │Summarizer│  │   向量语义搜索       │   │
│  │ Window  │  │          │  │   豆包 Embedding    │   │
│  └─────────┘  └──────────┘  └──────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

## 相关资源

- [OpenViking 文档](https://openviking.io)
- [ClawHub Skill](https://clawhub.ai/skill/viking-memory)
- [豆包 Embedding 模型](https://www.volcengine.com/docs/67079)
