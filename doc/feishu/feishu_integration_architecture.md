# 飞书（Feishu/Lark）集成架构设计方案

**版本**: v1.0  
**日期**: 2026-04-01  
**状态**: 设计阶段  

---

## 1. 项目背景与目标

### 1.1 当前系统现状

本项目是一个基于 Python 的 AI Agent 服务，当前架构如下：

```
用户消息 → HTTP Webhook POST → webhook_server.py → debounce.py → llm.chat() → message.send_text()
                                    ↓
                              本地JSON会话存储 + 可选的LanceDB记忆
```

**核心组件**:
- `main.py`: 入口，初始化所有组件，启动HTTP服务器
- `webhook_server.py`: HTTP请求处理器，接收消息回调
- `debounce.py`: 消息防抖合并，管理owner白名单
- `llm.py`: LLM调用、工具循环、会话管理
- `message.py`: 消息发送接口（目前只是打印日志）
- `tools.py`: 工具注册中心
- `scheduler.py`: 定时任务引擎

### 1.2 集成目标

将当前系统接入飞书（Feishu/Lark）企业自建应用，实现：

1. **接收消息**: 用户在飞书聊天界面发送消息时，Agent能够接收到飞书发来的消息
2. **发送消息**: Agent调用大模型处理完毕后，将回复发送到飞书用户聊天界面
3. **无公网IP要求**: 使用WebSocket长连接保持与飞书服务器的连接，无需公网IP

### 1.3 关键约束

- **无公网IP**: 不能使用传统的Webhook回调方式
- **WebSocket长连接**: 必须使用飞书提供的WebSocket事件订阅模式
- **最小代码量**: 以最小的改动实现需求
- **保留HTTP健康检查**: 需要保留HTTP端口用于健康检查和监控

---

## 2. 技术方案选型

### 2.1 WebSocket vs Webhook 对比

| 特性 | Webhook | WebSocket |
|------|---------|-----------|
| **连接方向** | 飞书 → 你的服务器（被动接收） | 你的服务器 → 飞书（主动连接） |
| **网络要求** | 需要公网 IP + 域名/端口 | 只需要能访问外网（出向） |
| **防火墙** | 需要开放入站端口 | 无需开放入站端口 |
| **实时性** | 实时（HTTP POST） | 实时（双向推送） |
| **复杂度** | 简单（HTTP 服务器） | 稍复杂（长连接维护） |
| **适用场景** | 有公网服务器 | 内网/无公网IP |

### 2.2 选择WebSocket的原因

**核心原因：无公网IP环境下Webhook无法工作**

```
Webhook 方案（不可行）              WebSocket 方案（可行）
─────────────────────              ─────────────────────

  飞书服务器                          飞书服务器
      │                                  ▲
      │  找不到你的服务器！                │
      ▼  (没有公网IP)           长连接建立 │
  ┌─────────┐                    ┌───────┴─────┐
  │ ❌ 无法  │                    │ ✓ 连接成功   │
  │   推送   │                    │  消息双向流动 │
  └─────────┘                    └─────────────┘
       你的内网服务器                    你的内网服务器
```

**WebSocket的优势**:
1. **主动outbound连接**: 从内网连到外网，不需要公网IP
2. **防火墙友好**: 只需要出向443端口（HTTPS），企业内网通常允许
3. **飞书官方支持**: 飞书开放平台原生支持WebSocket长连接模式

### 2.3 SDK选择

选择 **`lark-oapi`** 官方Python SDK，原因如下：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **lark-oapi官方SDK** | 封装完整，自动处理token刷新、重连、事件解析 | 引入外部依赖 |
| **手写WebSocket连接** | 轻量，完全可控 | 需要自己处理鉴权、心跳、重连 |

**决策**: 使用官方SDK，减少开发和维护成本。

---

## 3. 架构设计

### 3.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AI Agent 服务                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────────────────┐        ┌──────────────────────────────────────┐  │
│  │   HTTP Server        │        │   Feishu WebSocket Client            │  │
│  │   (健康检查)          │        │   (消息接收)                          │  │
│  │                      │        │                                      │  │
│  │   ├── GET /          │        │   1. 使用lark-oapi建立WebSocket连接   │  │
│  │   │   健康检查        │        │   2. 订阅im.message.receive_v1事件   │  │
│  │   └── 返回status     │        │   3. 解析并过滤消息                   │  │
│  │                      │        │   4. 调用debounce处理                 │  │
│  └──────────────────────┘        └──────────────────┬───────────────────┘  │
│                                                     │                       │
│                              ┌──────────────────────┘                       │
│                              ↓                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                        核心处理流程                                    │  │
│  │  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────┐ │  │
│  │  │  debounce   │ → │  llm.chat   │ → │  tools.exec │ → │  reply  │ │  │
│  │  │  防抖合并    │    │  LLM处理    │    │  工具执行    │    │  生成回复 │ │  │
│  │  └─────────────┘    └─────────────┘    └─────────────┘    └────┬────┘ │  │
│  │                                                                │      │  │
│  └────────────────────────────────────────────────────────────────┼──────┘  │
│                                                                   │         │
│                              ┌────────────────────────────────────┘         │
│                              ↓                                               │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │   Feishu Messenger                                                    │  │
│  │   (消息发送)                                                          │  │
│  │                                                                      │  │
│  │   使用飞书Bot API异步发送消息                                           │  │
│  │   POST https://open.feishu.cn/open-apis/im/v1/messages                │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
└────────────────────────────────────┼────────────────────────────────────────┘
                                     │
                                     ↓
                              ┌─────────────┐
                              │  飞书服务器  │
                              │  (云端)     │
                              └─────────────┘
                                     │
                                     ↓
                              ┌─────────────┐
                              │  用户聊天界面 │
                              └─────────────┘
```

### 3.2 模块职责划分

| 模块 | 职责 | 状态 |
|------|------|------|
| `feishu_client.py` | WebSocket客户端，接收飞书消息 | 新增 |
| `feishu_messenger.py` | 发送消息到飞书 | 新增 |
| `message.py` | 消息发送路由层 | 修改 |
| `webhook_server.py` | 不再用于接收消息，改为纯健康检查 | 修改/删除 |
| `main.py` | 初始化流程调整 | 修改 |

### 3.3 消息流转时序图

```
用户(飞书)          飞书服务器          feishu_client          debounce          llm          feishu_messenger
   │                    │                    │                   │              │                  │
   │ ──────────────────>│                    │                   │              │                  │
   │   发送"你好"        │                    │                   │              │                  │
   │                    │ ─────────────────> │                   │              │                  │
   │                    │   WebSocket推送     │                   │              │                  │
   │                    │   im.message.      │                   │              │                  │
   │                    │   receive_v1       │                   │              │                  │
   │                    │                    │ ─────────────────>│              │                  │
   │                    │                    │  debounce_message │              │                  │
   │                    │                    │  (sender_id, text)│              │                  │
   │                    │                    │                   │ ───────────> │                  │
   │                    │                    │                   │   llm.chat() │                  │
   │                    │                    │                   │              │                  │
   │                    │                    │                   │ <─────────── │                  │
   │                    │                    │                   │   返回回复    │                  │
   │                    │                    │                   │              │                  │
   │                    │                    │                   │ ──────────────────────────────> │
   │                    │                    │                   │              │   send_text()    │
   │                    │                    │                   │              │   (异步线程)      │
   │                    │ <──────────────────────────────────────────────────────────────────────── │
   │                    │   HTTP POST 发送消息 │                   │              │                  │
   │ <──────────────────│                    │                   │              │                  │
   │   收到回复          │                    │                   │              │                  │
```

---

## 4. 详细设计

### 4.1 配置文件结构

```yaml
# AI Agent 配置文件 - 飞书集成版

# 管理员ID列表（填写飞书open_id）
owner_ids:
  - "ou_xxxxxxxxxxxxxxxx"  # 你的飞书open_id
  - "ou_yyyyyyyyyyyyyyyy"  # 同事的飞书open_id

# 工作空间
workspace: "./workspace"

# 监听端口（HTTP健康检查）
port: 8080

# 消息防抖间隔（秒）
debounce_seconds: 3.0

# 消息平台配置
message:
  platform: "feishu"  # 标识使用飞书平台
  feishu:
    app_id: "cli_xxxxxxxxxxxx"           # 飞书应用ID
    app_secret: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"  # 飞书应用密钥
    domain: "feishu"                     # "feishu"(国内) 或 "lark"(国际版)
    # encrypt_key: ""                    # 可选：消息加密密钥

# 模型配置（保持不变）
models:
  default: "your_provider"
  providers:
    your_provider:
      api_base: "https://api.example.com/v1"
      api_key: "sk-xxx"
      model: "gpt-4"
      max_tokens: 8192
      timeout: 120
      extra_body: {}

# 记忆系统配置（保持不变）
memory:
  enabled: true
  embedding_api:
    api_base: "https://api.example.com/v1/embeddings"
    api_key: "sk-xxx"
    model: "text-embedding-3-small"
    dimension: 1536
  retrieve_top_k: 5
  similarity_threshold: 0.92
```

### 4.2 ID体系说明

#### 4.2.1 当前项目中的ID

| ID名称 | 来源 | 示例 | 用途 |
|--------|------|------|------|
| `sender_id` | 消息事件中的字段 | `"ou_xxxxxxxxxxxxxxxx"` | **通用概念**：表示"发送者ID"，是代码中使用的变量名 |
| `owner_ids` | `config.yaml`配置 | `["ou_xxxxxxxxxxxxxxxx"]` | **白名单**：允许触发AI回复的用户ID列表 |
| `session_key` | 代码生成 | `"dm_ou_xxxxxxxxxxxxxxxx"` | **会话标识**：用于存储对话历史文件名 |

#### 4.2.2 飞书中的ID类型

| ID类型 | 格式 | 用途 | 稳定性 |
|--------|------|------|--------|
| `open_id` | `ou_xxxxxxxxxxxxxxxx` | **用户唯一标识**（推荐用这个） | ✅ 永久不变 |
| `user_id` | `xxxxxxxx` | 企业内部员工号 | ❌ 可能变更 |
| `union_id` | `on_xxxxxxxxxxxxxxxx` | 跨应用用户标识 | ✅ 永久不变 |
| `chat_id` | `oc_xxxxxxxxxxxxxxxx` | 群聊唯一标识 | ✅ 永久不变 |

#### 4.2.3 ID映射关系

```
飞书消息事件                              当前代码
────────────────                         ────────
sender.sender_id.open_id  ───────────→  sender_id（变量）
                                        ↓
                                   与owner_ids匹配判断权限
                                        ↓
                                   session_key = f"dm_{open_id}"
```

### 4.3 核心模块设计

#### 4.3.1 feishu_client.py（WebSocket客户端）

**职责**:
1. 使用lark-oapi SDK建立WebSocket长连接
2. 接收`im.message.receive_v1`事件
3. 解析消息，提取sender_id和content
4. 过滤自己发送的消息（避免循环）
5. 群聊只处理@机器人的消息
6. 调用`debounce.debounce_message()`触发AI处理

**关键设计点**:
- 在后台线程运行，不阻塞主流程
- 自动重连机制由SDK处理
- 只处理文本消息类型

#### 4.3.2 feishu_messenger.py（消息发送器）

**职责**:
1. 使用飞书Bot API发送文本消息
2. 异步发送，避免阻塞主流程

**关键设计点**:
- 使用线程池或独立线程执行发送
- 支持私聊和群聊两种场景
- 错误处理和日志记录

#### 4.3.3 message.py（路由层）

**职责**:
1. 根据配置的`platform`路由到对应发送器
2. 保持接口统一，便于后续扩展其他平台

**关键设计点**:
- 对上层（debounce/llm）保持接口不变
- 根据`platform`配置动态选择发送器

#### 4.3.4 main.py（入口调整）

**职责**:
1. 初始化飞书客户端（如配置为飞书平台）
2. 在后台线程启动WebSocket连接
3. 保留HTTP健康检查服务器
4. 支持graceful shutdown

**启动流程**:
```
1. 加载配置
2. 初始化各子模块（llm、scheduler、tools、memory、debounce）
3. 初始化message模块（根据platform路由）
4. 如platform为feishu:
   a. 初始化feishu_client
   b. 在后台线程启动WebSocket连接
5. 启动HTTP健康检查服务器（主线程阻塞）
6. 捕获Ctrl+C，优雅关闭:
   a. 停止feishu_client
   b. 关闭HTTP服务器
```

### 4.4 消息处理逻辑

#### 4.4.1 接收消息处理流程

```
im.message.receive_v1事件
        ↓
提取sender.sender_id.open_id → sender_id
        ↓
检查sender.sender_type:
  - "app" → 过滤（自己发送的消息）
  - "user" → 继续处理
        ↓
检查message.chat_type:
  - "p2p"（私聊）→ 继续处理
  - "group"（群聊）→ 检查message.mentions是否包含机器人
        ↓
检查message.message_type:
  - "text" → 解析content.text
  - 其他类型 → 忽略
        ↓
调用debounce.debounce_message(sender_id, text)
```

#### 4.4.2 发送消息处理流程

```
llm.chat()返回reply
        ↓
调用message.send_text(to_id, reply)
        ↓
根据platform路由到feishu_messenger
        ↓
在后台线程执行:
  POST https://open.feishu.cn/open-apis/im/v1/messages
  参数:
    - receive_id_type: "open_id"或"chat_id"
    - receive_id: to_id
    - content: {"text": reply}
    - msg_type: "text"
```

---

## 5. 文件变更清单

| 文件 | 操作 | 说明 |
|------|------|------|
| `core/feishu_client.py` | 新增 | WebSocket客户端，接收飞书消息 |
| `core/feishu_messenger.py` | 新增 | 发送消息到飞书 |
| `core/message.py` | 修改 | 改为根据platform路由到对应发送器 |
| `core/webhook_server.py` | 删除/修改 | 不再用于接收消息，可删除或改为纯健康检查Handler |
| `main.py` | 修改 | 初始化feishu_client，调整启动流程 |
| `requirements.txt` | 修改 | 添加`lark-oapi`依赖 |
| `config-example.yaml` | 修改 | 添加飞书配置示例 |

---

## 6. 依赖管理

### 6.1 新增依赖

```
lark-oapi>=1.0.0
```

### 6.2 完整依赖列表

```
# 原有依赖
PyYAML>=6.0
loguru>=0.7.0
lancedb>=0.5.0
pyarrow>=14.0.0
numpy>=1.24.0

# 新增：飞书官方SDK
lark-oapi>=1.0.0
```

---

## 7. 飞书应用配置指南

### 7.1 创建飞书应用

1. 访问[飞书开放平台](https://open.feishu.cn/app)（国际版使用[Lark开发者控制台](https://open.larksuite.com/app)）
2. 点击"创建企业自建应用"
3. 填写应用名称和描述
4. 在"凭证与基础信息"页面复制App ID和App Secret

### 7.2 配置权限

在"权限管理"页面，开通以下权限：

```json
{
  "scopes": {
    "tenant": [
      "im:message",
      "im:message:readonly",
      "im:message:send_as_bot",
      "im:message.p2p_msg:readonly",
      "im:message.group_at_msg:readonly",
      "im:chat.members:bot_access",
      "contact:user.employee_id:readonly"
    ]
  }
}
```

### 7.3 启用机器人能力

在"应用能力" → "机器人"页面：
1. 启用机器人能力
2. 设置机器人名称

### 7.4 配置事件订阅

在"事件订阅"页面：
1. 选择"使用长连接接收事件（WebSocket）"
2. 添加事件：`im.message.receive_v1`

⚠️ **注意**: 配置事件订阅前，确保Agent服务已启动，否则长连接无法建立。

### 7.5 发布应用

在"版本管理与发布"页面：
1. 创建版本
2. 提交审核并发布
3. 等待管理员审批（企业自建应用通常自动通过）

---

## 8. 关键设计决策

| 决策点 | 选择 | 原因 |
|--------|------|------|
| SDK | `lark-oapi` | 官方维护，自动处理鉴权和重连 |
| 架构 | 单进程多线程 | 改动最小，与现有代码风格一致 |
| 消息接收 | WebSocket替换Webhook | 无需公网IP |
| 消息发送 | 异步线程 | 避免阻塞主流程 |
| 健康检查 | 保留HTTP端口 | 便于监控和容器化部署 |
| ID标识 | `open_id` | 全局唯一，永久不变 |
| 群聊处理 | 仅处理@机器人 | 避免噪音 |
| 自消息过滤 | `sender_type == "app"` | 防止循环 |

---

## 9. 风险与应对

| 风险 | 影响 | 应对措施 |
|------|------|----------|
| WebSocket连接断开 | 无法接收消息 | SDK自动重连，日志记录 |
| 飞书API限流 | 消息发送失败 | 实现重试机制，指数退避 |
| 消息解析异常 | 消息丢失 | 异常捕获和日志记录 |
| 配置错误 | 无法连接飞书 | 启动时校验配置，给出明确错误提示 |

---

## 10. 后续扩展建议

1. **支持更多消息类型**: 图片、文件、富文本等
2. **群聊增强**: 支持无需@的群聊模式配置
3. **多账号支持**: 同时接入多个飞书应用
4. **消息加密**: 支持飞书的消息加密功能
5. **交互卡片**: 支持飞书的卡片消息格式

---

## 11. 附录

### 11.1 术语表

| 术语 | 说明 |
|------|------|
| `open_id` | 飞书用户的唯一标识符，格式为`ou_`开头 |
| `app_id` | 飞书应用的唯一标识符，格式为`cli_`开头 |
| `app_secret` | 飞书应用的密钥，用于API鉴权 |
| `im.message.receive_v1` | 飞书消息接收事件类型 |
| WebSocket | 一种在单个TCP连接上进行全双工通信的协议 |
| Webhook | 一种HTTP回调机制，服务器主动推送数据到客户端 |

### 11.2 参考文档

- [飞书开放平台文档](https://open.feishu.cn/document/home/index)
- [lark-oapi Python SDK](https://github.com/larksuite/oapi-sdk-python)
- [OpenClaw飞书集成指南](https://openclawx.cloud/zh/channels/feishu)

---

**文档结束**
