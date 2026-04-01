# 飞书（Feishu/Lark）集成实现计划

**版本**: v1.0  
**日期**: 2026-04-01  
**状态**: 设计阶段  

---

## 阶段划分原则

1. **由底向上**：先基础设施，后业务逻辑
2. **独立可测**：每个阶段都有独立的测试验证点
3. **最小可用**：每个阶段结束都有可运行的状态
4. **风险前置**：技术风险高的模块先做验证

---

## 阶段一：飞书 Messenger 发送模块

### 目标
实现消息发送功能，能够主动向飞书用户发送文本消息。

### 交付物
- `core/feishu_messenger.py` - 飞书消息发送器
- `tests/test_feishu_messenger.py` - 单元测试

### 测试方案

| 测试类型 | 测试内容 | 验证方式 |
|---------|---------|---------|
| 单元测试 | 构造发送请求 | Mock飞书API，验证请求参数正确 |
| 集成测试 | 真实发送消息 | 配置测试应用，向指定用户发送测试消息 |
| 边界测试 | 超长消息、特殊字符 | 验证消息拆分和编码处理 |

### 测试命令
```bash
# 单元测试
python -m pytest tests/test_feishu_messenger.py -v

# 集成测试（需要配置测试凭证）
export FEISHU_TEST_APP_ID="cli_xxx"
export FEISHU_TEST_APP_SECRET="xxx"
export FEISHU_TEST_USER_ID="ou_xxx"
python -m pytest tests/test_feishu_messenger.py::test_send_real_message -v
```

---

## 阶段二：Message 路由层改造

### 目标
改造 `message.py`，支持根据配置路由到不同平台的发送器。

### 交付物
- 修改 `core/message.py` - 添加平台路由逻辑
- `tests/test_message_router.py` - 路由层测试

### 测试方案

| 测试类型 | 测试内容 | 验证方式 |
|---------|---------|---------|
| 单元测试 | 配置为feishu时路由正确 | Mock feishu_messenger，验证调用 |
| 单元测试 | 配置为未知平台时行为 | 验证日志警告和返回False |
| 兼容性测试 | 现有代码调用方式不变 | 验证debounce等模块无需修改 |

### 测试命令
```bash
python -m pytest tests/test_message_router.py -v
```

---

## 阶段三：飞书 Client 接收模块

### 目标
实现WebSocket客户端，能够接收飞书消息并解析。

### 交付物
- `core/feishu_client.py` - WebSocket客户端
- `tests/test_feishu_client.py` - 单元测试
- `tests/test_feishu_integration.py` - 集成测试

### 测试方案

| 测试类型 | 测试内容 | 验证方式 |
|---------|---------|---------|
| 单元测试 | 消息解析逻辑 | 构造飞书事件JSON，验证解析结果 |
| 单元测试 | 自消息过滤 | 构造sender_type=app的消息，验证被过滤 |
| 单元测试 | 群聊@检测 | 构造群聊消息，验证@检测逻辑 |
| 集成测试 | WebSocket连接建立 | 使用测试应用，验证能接收真实消息 |
| 集成测试 | 消息流转完整链路 | 发送消息→Agent处理→回复接收 |

### 测试命令
```bash
# 单元测试
python -m pytest tests/test_feishu_client.py -v

# 集成测试（需要配置测试凭证）
export FEISHU_TEST_APP_ID="cli_xxx"
export FEISHU_TEST_APP_SECRET="xxx"
python -m pytest tests/test_feishu_integration.py -v
```

---

## 阶段四：Main 入口整合

### 目标
整合所有模块，调整启动流程，支持graceful shutdown。

### 交付物
- 修改 `main.py` - 调整初始化和启动流程
- 修改 `requirements.txt` - 添加依赖
- 修改 `config-example.yaml` - 添加飞书配置示例
- `tests/test_main_integration.py` - 集成测试

### 测试方案

| 测试类型 | 测试内容 | 验证方式 |
|---------|---------|---------|
| 启动测试 | 配置为feishu时正常启动 | 验证无异常，WebSocket连接建立 |
| 启动测试 | HTTP健康检查端口正常 | curl http://localhost:8080/ 返回ok |
| 关闭测试 | Ctrl+C优雅关闭 | 验证WebSocket连接正常关闭 |
| 端到端测试 | 完整消息流转 | 飞书发送→Agent处理→飞书接收 |

### 测试命令
```bash
# 启动服务测试
python main.py &
sleep 3
curl http://localhost:8080/
kill %1

# 端到端测试（需要完整配置）
python -m pytest tests/test_main_integration.py -v
```

---

## 阶段五：端到端验收测试

### 目标
完整验证飞书集成功能，编写使用文档。

### 交付物
- `tests/test_e2e_feishu.py` - 端到端测试
- `doc/FEISHU_SETUP_GUIDE.md` - 飞书配置指南
- 更新 `README.md` - 添加飞书集成说明

### 测试方案

| 测试类型 | 测试内容 | 验证方式 |
|---------|---------|---------|
| 私聊测试 | 用户私聊Agent | 验证能接收消息并回复 |
| 群聊测试 | 群聊@Agent | 验证能接收@消息并回复 |
| 并发测试 | 多用户同时发送 | 验证消息不丢失、不混淆 |
| 长连接测试 | 运行24小时 | 验证连接稳定，自动重连正常 |

### 测试命令
```bash
# 端到端测试
python -m pytest tests/test_e2e_feishu.py -v

# 长连接稳定性测试（后台运行）
python tests/stress_test.py --duration=86400
```

---

## 阶段依赖关系

```
阶段一：Messenger发送模块
    ↓
阶段二：Message路由层改造
    ↓
阶段三：Client接收模块
    ↓
阶段四：Main入口整合
    ↓
阶段五：端到端验收测试
```

---

## 各阶段时间预估

| 阶段 | 预估工作量 | 关键风险 |
|------|-----------|---------|
| 阶段一 | 1-2小时 | 飞书API调用方式 |
| 阶段二 | 1小时 | 保持向后兼容 |
| 阶段三 | 2-3小时 | WebSocket连接稳定性 |
| 阶段四 | 1-2小时 | 启动流程协调 |
| 阶段五 | 2-3小时 | 端到端环境问题 |

---

## 测试环境需求

每个阶段都需要以下测试环境：

1. **飞书测试应用**
   - App ID 和 App Secret
   - 已开通必要权限
   - 已发布并通过审核

2. **测试用户**
   - 至少一个测试用户的 open_id
   - 测试用户已添加测试应用为好友

3. **网络环境**
   - 能访问飞书开放平台（open.feishu.cn）
   - 出向443端口开放

---

## 文件变更总览

| 阶段 | 新增文件 | 修改文件 | 删除文件 |
|------|---------|---------|---------|
| 阶段一 | `core/feishu_messenger.py`<br>`tests/test_feishu_messenger.py` | - | - |
| 阶段二 | `tests/test_message_router.py` | `core/message.py` | - |
| 阶段三 | `core/feishu_client.py`<br>`tests/test_feishu_client.py`<br>`tests/test_feishu_integration.py` | - | - |
| 阶段四 | `tests/test_main_integration.py` | `main.py`<br>`requirements.txt`<br>`config-example.yaml` | `core/webhook_server.py`（可选） |
| 阶段五 | `tests/test_e2e_feishu.py`<br>`doc/FEISHU_SETUP_GUIDE.md` | `README.md` | - |

---

## 关键配置示例

```yaml
# config.yaml 飞书配置部分
message:
  platform: "feishu"
  feishu:
    app_id: "cli_xxxxxxxxxxxx"
    app_secret: "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    domain: "feishu"  # 或 "lark"

owner_ids:
  - "ou_xxxxxxxxxxxxxxxx"  # 你的飞书open_id
```

---

**文档结束**
