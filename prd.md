# AGID — Agent Graph Identity & Discovery

> 把这次对话的架构设计和 MVP 建议整理成可操作文档。

-----

## 背景：问题是什么

用户需求原话：

> “我想要一个类似 Agent ID 的东西，各种各样的 Agent 都支持，我就可以让我的 Agent 去加朋友的 Agent 为好友，然后他俩聊。现存的方案都比较封闭，我在你的平台上注册，就得用你的 Agent，我朋友的 Agent 也必须在你的平台上，而且现在平台的 UI 都非常的重和没必要。”

本质需求：**Agent 的去中心化身份 + 跨平台通信 + 社交图谱层**

-----

## 现有协议全景

|协议 |主导方                 |定位                  |问题                  |
|---|--------------------|--------------------|--------------------|
|MCP|Anthropic           |Agent 连工具           |不是 agent 连 agent，不对口|
|A2A|Google（2025.4）      |跨厂商 agent 任务委托      |企业向，OAuth，有中心化信任假设  |
|ANP|开源社区                |agent 互联网的 HTTP，去中心化|协议有了，缺社交语义层         |
|ACP|IBM/Linux Foundation|REST 框架编排           |企业向，太重              |

**结论：** ANP 技术上最接近，但没有”加好友”这个社交层。ANP 是 TCP/IP，缺的是微信。

-----

## AGID 架构

```
┌─────────────────────────────────────────┐
│           用户意图层（极轻 UI）           │
│   "让我的 Agent 去加 Alice 的 Agent"     │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│           社交图谱层（新增核心）           │
│  • Agent 名片（human-readable handle）   │
│  • 好友关系图（去中心化存储）             │
│  • 连接请求 / 握手语义                   │
│  • 信任等级（stranger / friend / trust） │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         身份层（复用 ANP/DID）            │
│  • did:wba 或 did:web 作为底层 ID        │
│  • 密钥对，跨平台可验证                   │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│         通信层（复用 ANP/HTTPS）          │
│  • 端到端加密                            │
│  • 元协议协商（格式自协商）               │
└─────────────────────────────────────────┘
```

### 社交图谱层——三个核心设计

**① Human-readable handle**

类似 `@alice.agent` 或 `alice@agid.social`，底层映射到 DID，表面对人友好。
就像 email 地址之于 IP 地址。

**② 连接握手语义**

“加好友”需要带意图声明的协议动作，而不是普通 HTTP 请求：

```json
{
  "type": "ConnectionRequest",
  "from": "did:wba:alice.com:agent1",
  "to": "did:wba:bob.com:agent2",
  "intent": "持续通信 | 单次任务 | 数据共享",
  "scope": ["chat", "task_delegation"],
  "expiry": "optional"
}
```

对方主人可手动批准，也可设置自动批准规则（如”只接受通讯录里的人的 agent”）。

**③ 去中心化好友图**

关系数据不存在任何平台，存在用户自己控制的地方：

- 自托管 JSON 文件
- AT Protocol 的 PDS（Bluesky 用的那套）
- 本地加密存储

平台只是 UI，关系跟着人走。

### 信任模型——三个圈

|圈层      |名称 |权限                      |
|--------|---|------------------------|
|Circle 0|陌生人|只能发连接请求，内容受限            |
|Circle 1|好友 |双向确认后，agent 自由对话、协作任务   |
|Circle 2|信任 |可授权对方 agent 代表自己行动（如订机票）|

**防垃圾机制：** 引入社会成本——每发一个连接请求消耗声誉积分（类似 hashcash 的 proof-of-work 思路，让批量垃圾请求有代价）。

-----

## MVP 方案

### 目标

> 两个 agent 在不同机器上跑，能互相发现、握手确认好友关系、然后对话。

### 不做

- 完整的 DID 基础设施
- 漂亮 UI
- 多用户、持久化数据库

### 技术栈

|层 |选型              |理由                       |
|--|----------------|-------------------------|
|后端|Python + ANP SDK|直接复用，有 quickstart example|
|前端|React / TS      |熟悉，极简界面即可                |
|部署|本地两个进程 + ngrok  |模拟跨机器，零成本                |

-----

## 分三步走

### Step 1（1-2天）— 跑通 ANP demo

```bash
git clone https://github.com/agent-network-protocol/anp
cd anp
uv run python examples/python/fastanp_examples/simple_agent.py
```

**目标：** 理解 DID 怎么生成，agent 怎么互相认证，消息怎么传。
把 `hotel_booking_agent.py` 也跑一遍，摸清数据流。

### Step 2（2-3天）— 加社交层

在 ANP 之上写一个薄薄的社交语义层，三个接口：

```python
POST /connect-request   # 发送好友请求
POST /connect-accept    # 接受请求  
POST /message           # 发消息（好友关系确认后才通）
```

好友关系存本地 `friends.json`，不用数据库。

```json
{
  "friends": [
    {
      "did": "did:wba:bob.example.com:agent1",
      "handle": "bob@example.com",
      "status": "accepted",
      "since": "2025-06-06T00:00:00Z",
      "trust_level": 1
    }
  ],
  "pending": []
}
```

### Step 3（1-2天）— 套 UI

React 极简界面：

- 左边：好友列表 + Agent ID 显示
- 右边：聊天框
- 顶部：自己的 handle（如 `alice@localhost:8001`）

-----

## 关键参考资源

|资源            |链接                                                              |
|--------------|----------------------------------------------------------------|
|ANP 协议规范      |<https://github.com/agent-network-protocol/AgentNetworkProtocol>|
|ANP Python SDK|<https://github.com/agent-network-protocol/anp>                 |
|ANP 示例程序      |<https://github.com/agent-network-protocol/anp-examples>        |
|ANP 在线 demo   |<https://service.agent-network-protocol.com/anp-demo/>          |
|W3C DID 规范    |<https://www.w3.org/TR/did-core/>                               |

-----

## 真正的难点

**不是技术，是冷启动。**

这是双边网络：你的 agent 要有朋友的 agent 才有意义。
现有协议都在攻企业侧（有明确需求和资源），**个人向的轻量社交 agent 网络这个位置是空的**。

企业协议会越来越重，窗口在这里。

-----

*整理自 Claude 对话，2025-06-06*