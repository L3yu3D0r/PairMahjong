# PairMahjong — 架构说明

> 本文件为只读。任何修改需人工批准。
> 本文件与 `AGENTS.md` 冲突时，以 `AGENTS.md` 为准。

---

## 1. 系统概览

PairMahjong 是一个**无服务器**四人麻将游戏。

- 游戏状态完全由玩家客户端维护。
- 网络使用 WebRTC P2P 全互连。
- 同步模型为「权威排序 + 确定性重放」。
- 允许存在的中心服务：信令、STUN、TURN（均不参与游戏逻辑）。

---

## 2. 分层架构

```text
┌──────────────────────────────────────────┐
│  表示层 Presentation                      │
│  Godot 场景 / UI / 动画 / 音效            │
└───────────────────┬──────────────────────┘
                    │ 只调用公开 API，不直接改状态
┌───────────────────▼──────────────────────┐
│  游戏逻辑层 Game Logic                    │
│  麻将规则引擎 / AI 决策                   │
└───────────────────┬──────────────────────┘
                    │ 纯函数：reduce(state, event)
┌───────────────────▼──────────────────────┐
│  会话与状态层 Session & State             │
│  房间 / 座位 / 权威排序 / 重连 / 公平性   │
└───────────────────┬──────────────────────┘
                    │ 序列化消息
┌───────────────────▼──────────────────────┐
│  网络传输层 Networking                    │
│  ITransport 抽象 / 消息编解码             │
└───────────────────┬──────────────────────┘
                    │ WebRTC DataChannel
┌───────────────────▼──────────────────────┐
│  P2P 连接层 Connectivity                  │
│  WebRTCMultiplayerPeer / 信令 / NAT       │
└──────────────────────────────────────────┘
```

---

## 3. 工程结构

```text
PairMahjong/
├─ AGENTS.md                    # Agent 契约（只读）
├─ docs/
│  ├─ plan.md                   # 任务契约（只读）
│  ├─ architecture.md           # 本文件（只读）
│  ├─ protocol.md               # 消息协议（只读）
│  └─ conventions.md            # 编码规范（只读）
├─ .agent/
│  ├─ status.json               # 任务状态
│  ├─ context.md                # 上下文摘要
│  ├─ log.md                    # 执行日志
│  └─ blocked.md                # 阻塞记录（按需）
├─ src/
│  ├─ PairMahjong.Core/         # 纯 C# 类库
│  │  ├─ PairMahjong.Core.csproj
│  │  ├─ Tile.cs
│  │  ├─ GameState.cs
│  │  ├─ PlayerState.cs
│  │  ├─ PlayerView.cs
│  │  ├─ GameAction.cs
│  │  ├─ GameEvent.cs
│  │  ├─ Reduce.cs
│  │  ├─ LegalActions.cs
│  │  ├─ Hu.cs
│  │  ├─ Fan.cs
│  │  ├─ Wall.cs
│  │  ├─ Meld.cs
│  │  └─ Rng.cs
│  ├─ PairMahjong.Ai/           # 纯 C# 类库
│  │  ├─ PairMahjong.Ai.csproj
│  │  ├─ IBot.cs
│  │  ├─ RandomBot.cs
│  │  └─ EfficiencyBot.cs
│  └─ PairMahjong.Fairness/     # 纯 C# 类库
│     ├─ PairMahjong.Fairness.csproj
│     ├─ Commit.cs
│     ├─ Signature.cs
│     └─ StateHash.cs
├─ scripts/                     # Godot 相关 C#
│  ├─ Autoload/
│  │  ├─ NetworkManager.cs
│  │  └─ GameManager.cs
│  ├─ Net/
│  │  ├─ ITransport.cs
│  │  ├─ WebRtcTransport.cs
│  │  ├─ Codec.cs
│  │  ├─ Signaling/
│  │  └─ Session/
│  │     ├─ Room.cs
│  │     ├─ SessionManager.cs
│  │     ├─ Authority.cs
│  │     ├─ Sync.cs
│  │     ├─ Reconnect.cs
│  │     └─ AiTakeover.cs
│  └─ Ui/
├─ scenes/
│  ├─ Lobby/
│  ├─ Game/
│  └─ Ui/
├─ tests/
│  ├─ PairMahjong.Core.Tests/
│  ├─ PairMahjong.Ai.Tests/
│  └─ PairMahjong.Session.Tests/
├─ PairMahjong.sln
├─ PairMahjong.csproj
├─ Directory.Build.props
└─ project.godot
```

---

## 4. 依赖方向

```text
Ui / Autoload
    │
    ├──> Net / Session / Fairness
    │         │
    │         └──> Ai
    │                  │
    │                  └──> Core
    │
    └──> Core（只读）
```

**禁止反向依赖。**

`Core` / `Ai` / `Fairness` 禁止 `using Godot;`。

---

## 5. 模块职责

### 5.1 Core

- 纯逻辑，无副作用。
- 所有游戏规则、状态、动作、事件。
- 输入输出使用整数。
- 确定性：相同输入产生相同输出。

### 5.2 Ai

- 只读取 `PlayerView`。
- 输出 `GameAction`。
- 禁止读取 `GameState`。
- 禁止访问网络。

### 5.3 Fairness

- 种子承诺。
- 操作签名。
- 状态哈希。
- 不参与正常游戏逻辑。

### 5.4 Net

- WebRTC 连接管理。
- 消息序列化 / 反序列化。
- 信令抽象。
- 禁止实现游戏规则。

### 5.5 Session

- 房间生命周期。
- 座位分配。
- 权威排序：房主验证 Action -> 广播 Event。
- 重连与状态同步。
- AI 补位。

### 5.6 Ui

- Godot 场景与节点。
- 只调用 Session / GameManager 的公开 API。
- 禁止直接修改 `GameState`。

---

## 6. 核心数据流

### 6.1 本地对局

```text
UI -> GameAction -> GameManager
GameManager -> GameEvent -> Reduce -> GameState
GameState -> PlayerView -> UI
```

### 6.2 联机对局

```text
客户端 A UI -> GameAction
    │
    ▼
ActionMessage -> WebRTC -> 房主
    │
    ▼
房主 Authority.Validate
    │
    ▼
EventMessage (含 seq) -> 广播
    │
    ▼
所有客户端 Reduce(state, event) -> 新 GameState
    │
    ▼
StateHash 广播 -> 一致性校验
```

### 6.3 洗牌承诺

```text
每个玩家:
    seed, salt = random()
    broadcast SHA256(seed + salt)
开局后:
    broadcast seed, salt
    verify SHA256
    组合所有 seed -> 最终牌墙
```

---

## 7. 关键约束

| 编号 | 约束 |
|---|---|
| A1 | Core / Ai / Fairness 禁止引用 Godot |
| A2 | 所有规则计算禁止浮点 |
| A3 | 所有随机必须显式种子 |
| A4 | 事件顺序由 seq 决定 |
| A5 | 状态哈希使用 SHA-256 |
| A6 | 消息协议见 docs/protocol.md |
| A7 | 网络消息禁止使用 Godot.Collections.Dictionary |
| A8 | 每 TASK 一个 commit |

---

## 8. 非目标

- 完整日麻役种
- 观战系统
- 聊天系统
- 排行榜
- 账号系统
- 多阶段加密洗牌（MVP 只做种子承诺）
- 零知识证明

---

## 9. 参考

- `AGENTS.md` — Agent 契约
- `docs/plan.md` — 任务契约
- `docs/protocol.md` — 消息协议
- `docs/conventions.md` — 编码规范

---

## 10. 版本

```yaml
version: 0.1.0
last_updated: 2026-01-01
maintainer: human
agent_editable: false
```

**本文件为只读。任何修改需人工批准。**