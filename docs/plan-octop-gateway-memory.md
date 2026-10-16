# BT 移植计划：octop-gateway 多平台通道桥 + octop-memory 分层记忆

> 日期：2026-10-16　分支：dev
> 目标：把 Octop 官方两个可移植包接入 BT，**不破坏现有 /api/* 与 9876 生产**。

---

## 一、背景与现状

BT 已有手写的 `platform_ext/channels.py`（玩具版通道抽象，仅 http/websocket 日志回声）和
自研记忆（`chat_client/memory.py` 会话滑动窗口 + mem0 工具）。Octop 官方有成熟实现：

| 包 | 版本 | Python | 用途 |
|---|---|---|---|
| `octop-gateway` | 1.0.3 | ≥3.12 ✓ | **多平台 IM 通道桥**：微信/飞书/钉钉/Telegram/QQ/企业微信/元宝/小一/叮叮/QQ + manager/channel/group_context/push_routing |
| `octop-memory` | 1.0.2 | ≥3.12 ✓ | **分层记忆系统**：core/service/types + migration/portable + pipeline/episode |

BT 的 `.venv` 是 **Python 3.12**，满足两包要求。

---

## 二、依赖核对（.venv 现状）

### octop-gateway 依赖
| 依赖 | 已装 | 说明 |
|---|---|---|
| aiohttp / httpx / pydantic / python-dotenv / websocket-client / websockets / cryptography | ✅ | 已就绪 |
| dingtalk-stream | ❌ | 钉钉 |
| discord-py | ❌ | Discord |
| lark-oapi | ❌ | 飞书 |
| paho-mqtt | ❌ | MQTT |
| python-socketio[asyncio-client] | ❌ | SocketIO |
| python-telegram-bot | ❌ | Telegram |
| wecom-aibot-sdk | ❌ | 企业微信 |

**策略**：核心 `channel.py / manager.py / group_context.py / models.py / utils.py / push_routing.py`
依赖 `aiohttp + httpx + pydantic`（已具备），可先接入。**平台 SDK 按需安装**——只装用户要激活的
平台（如微信/飞书/Telegram），不一次全装，避免臃肿。

### octop-memory 依赖
| 依赖 | 已装 | 说明 |
|---|---|---|
| pydantic | ✅ | 核心 |
| fastapi / uvicorn | ✅ | dashboard 可选 |
| chromadb / sentence-transformers / qdrant | ❌ 可选 | 向量库（BT 已有 mem0 + 自研，可跳过） |

**策略**：`core.py / service.py / types.py` 核心轻依赖，可直接接入。向量/embedding 可选 extra 不装。

---

## 三、实施范围（不破坏现有 API）

### 阶段 1：octop-gateway 接入

1. **vendor 包**：把 `octop_gateway/` 放入 `vendor/octop_gateway/`（或 site-packages 安装）。
   - 更稳妥：`pip install --target vendor/octop-gateway`，随仓库携带，避免全局污染。
2. **新命名空间 `platform_ext/gateway.py`**：
   - 封装 `octop_gateway.manager.GatewayManager` 的启动/停止/注册。
   - 复用 BT 现有 `agent.py` 处理入站消息（把消息交给集团/单 agent）。
   - 复用 BT 现有 `wechat_media.py` / 869 发送能力作为「微信」通道落地（BT 已有 869 微信）。
3. **API**（前缀 `/api/v2/gateway/*`）：
   - `GET /api/v2/gateway/channels` — 列出可用通道与激活状态
   - `POST /api/v2/gateway/channels/{id}/start` / `stop` — 启停某平台通道
   - `GET /api/v2/gateway/status` — 全局状态
4. **配置**：`config.json` 新增 `platform_ext.gateway` 段，存各平台 token/webhook。
5. **不破坏**：BT 现有微信小程序/clawbot/869 逻辑**原样保留**；gateway 只作为**新增通道**叠加。

### 阶段 2：octop-memory 接入

1. **vendor 包**：`pip install --target vendor/octop-memory` 随仓库携带。
2. **新命名空间 `platform_ext/memory_octop.py`**：
   - 封装 `octop_memory.core` / `service` 的分层记忆（episode/digest/recall/consolidate）。
   - 把 BT 现有会话记忆（`chat_client/memory.py`）作为底层存储接入，或让 octop-memory 作为**增强记忆层**叠加。
   - 提供 `/api/v2/memory/*` API：recall、list、consolidate、export/import（可迁移）。
3. **与 mem0 共存**：BT 已有 mem0 工具（search/add/list/delete/import）。octop-memory 作为**可选增强**，
   不替代 mem0；提供桥接让 octop-memory 的 recall 结果可注入 agent 上下文。

### 不做（避免破坏）
- 不替换 BT 现有 `chat_client/memory.py`（会话滑动窗口）——它承担核心会话上下文。
- 不替换现有 mem0 工具——两者并存，octop-memory 增强长期记忆。
- 不一次安装全部 IM SDK——只按需装用户激活的平台。
- 不引入 octop-harness（LangChain agent 框架）——太重。

---

## 四、验证与回归

1. **导入检查**：`import platform_ext.gateway`、`import platform_ext.memory_octop`、`import octop_gateway`、`import octop_memory` 均成功。
2. **独立端口冒烟**：在 19879 启动，验证 `/api/v2/gateway/status`、`/api/v2/memory/*` 响应。
3. **现有测试全量**：`pytest tests/ -q` 全通过（当前 287 项，本次新增测试预计 +15~25）。
4. **生产 9876 不动**：新代码只挂 `/api/v2/*` 新前缀，不触碰现有 `/api/*`。

---

## 五、依赖安装顺序

```bash
cd /home/sxkiss/bt
# 阶段 1（gateway 核心）
.venv/bin/pip install --target vendor/octop-gateway octop-gateway==1.0.3
# 阶段 2（memory 核心）
.venv/bin/pip install --target vendor/octop-memory octop-memory==1.0.2
# （按需，仅激活平台时）如微信/Telegram SDK
.venv/bin/pip install --target vendor/octop-gateway python-telegram-bot lark-oapi dingtalk-stream wecom-aibot-sdk python-socketio paho-mqtt discord-py
```

> 使用 `--target vendor/` 而非全局，符合项目「依赖必须进虚拟环境/随项目携带，禁止全局安装」的铁律。

---

## 六、风险与缓解

| 风险 | 缓解 |
|---|---|
| octop-gateway 依赖 `octop-harness`（dev extra） | 生产不装 dev extra；核心不 import harness |
| IM SDK 版本冲突 | 只按需装用户激活平台；vendor 隔离，不影响 BT 现有依赖 |
| octop-memory 需要向量库 | 用 BT 现有 mem0/自研存储，跳过 chroma/qdrant |
| 两包 Python 3.12 要求 | BT .venv 已是 3.12 ✓ |
| 破坏现有 /api/* | 全部走 `/api/v2/*` 新前缀；现有通道逻辑原样保留 |

---

## 七、交付物

- `vendor/octop-gateway/`、`vendor/octop-memory/`（随仓库）
- `platform_ext/gateway.py`（多平台通道桥封装 + `/api/v2/gateway/*`）
- `platform_ext/memory_octop.py`（增强记忆封装 + `/api/v2/memory/*`）
- `config.json` 新增 `platform_ext.gateway` / `platform_ext.memory_octop` 段
- `tests/test_gateway.py`、`tests/test_memory_octop.py`
- 更新 `.gitignore`（vendor 运行时数据如凭证缓存）