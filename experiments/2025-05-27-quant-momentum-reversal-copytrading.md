# 量化学习笔记：动量/反转效应 & 毫秒级跟单架构

> 日期：2025-05-27
> 标签：quant, momentum, mean-reversion, copy-trading, okx, low-latency

---

## 一、动量效应 (Momentum) vs 反转效应 (Mean Reversion)

### 核心概念

| 维度 | 动量 (Momentum) | 反转 (Mean Reversion) |
|------|----------------|----------------------|
| **假设** | 涨的继续涨，跌的继续跌 | 价格偏离均值后会回归 |
| **驱动逻辑** | 趋势跟踪、羊群效应、信息滞后 | 超跌反弹、估值回归、套利 |
| **时间周期** | 中期（周/月级）最显著 | 短期（秒/分钟级）更有效 |
| **典型策略** | CTA、趋势跟踪 | 做市、套利、网格 |

### 经典文献要点

- **Jegadeesh & Titman (1993)**：发现个股月度动量效应，过去6-12个月收益率能预测未来收益
- **Aharon & Yaari (2005)**：长期（3-5年）存在明显反转效应
- **Narwal & Jasmindeep (2021)**：加密市场动量效应存在但比传统市场更短暂
- **挂单/吃单不对称性**：在加密市场，Maker（挂单方）比 Taker 更有优势

### 量化实现框架

```python
# 简化版动量因子
def momentum_factor(prices: pd.Series, lookback: int = 20) -> pd.Series:
    """动量因子：过去N期收益率"""
    return prices.pct_change(periods=lookback)

# 简化版均值回归因子
def mean_reversion_factor(prices: pd.Series, window: int = 20) -> pd.Series:
    """均值回归因子：Z-score = (当前价 - 移动均值) / 标准差"""
    ma = prices.rolling(window).mean()
    std = prices.rolling(window).std()
    return (prices - ma) / std

# 经典RSI（相对强弱指数）= 变相的均值回归
def rsi(prices: pd.Series, period: int = 14) -> pd.Series:
    delta = prices.diff()
    gain = delta.where(delta > 0, 0).rolling(period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(period).mean()
    rs = gain / loss
    return 100 - (100 / (1 + rs))
```

---

## 二、OKX 一键跟单的问题

### 一键跟单的延迟来源

```
用户下单
  → OKX 服务器接收信号（~50-200ms）
  → 信号处理 + 风控检查（~100-500ms）
  → 信号转发到跟单者账户（~50-200ms）
  → 订单执行（~10-100ms）
  ─────────────────────────────────
  总延迟：通常 200ms ~ 1s+，极端情况 3s+
```

### 为什么延迟是致命的

- **滑点**：在波动剧烈的行情中，1秒价格可能偏移 0.1%~1%+
- **深度踩踏**：跟单者集中平仓导致流动性枯竭
- **信号失效**：趋势信号在高频下可能瞬间反转

---

## 三、毫秒级跟单：自己实现的核心思路

### 架构总览

```
┌──────────────┐    WebSocket     ┌─────────────────┐   HTTP/WS   ┌─────────────────┐
│  交易员账户  │ ───────────────→ │  信号接收服务   │ ──────────→ │  执行引擎       │
│ (Signal Src) │  (实时推送)      │  (Python/Go)    │ (预处理)    │ (低延迟下单)    │
└──────────────┘                  └────────┬────────┘             └────────┬────────┘
                                          │                                │
                                          │ REST/WebSocket                  │ API
                                          ↓                                ↓
                                   ┌──────────────┐              ┌─────────────────┐
                                   │  数据缓存    │              │  交易所 API     │
                                   │  (Redis)    │              │  (OKX/币安)    │
                                   └──────────────┘              └─────────────────┘
```

### 关键技术选型

| 层次 | 技术方案 | 说明 |
|------|---------|------|
| **信号获取** | 交易所 WebSocket API | 实时推送，无需轮询，延迟 <10ms |
| **信号处理** | Go / C++ / Rust | Python GIL 限制，高频场景不推荐 |
| **消息队列** | Redis Pub/Sub / Kafka | 进程间通信，<1ms |
| **执行** | 交易所低延迟 API | 异步下单，预估 <50ms 端到端 |
| **网络** | 同城低延迟托管 / 专线 | 网络延迟是最大瓶颈 |

### 简化版 Python 实现框架

```python
import asyncio
import websockets
import aiohttp
import time
import hmac
import hashlib
import json
from typing import Optional

class MillisecondCopyTrader:
    def __init__(self, api_key: str, secret: str, passphrase: str):
        self.api_key = api_key
        self.secret = secret
        self.passphrase = passphrase
        self.ws_url = "wss://ws.okx.com:8443/ws/v5/private"
        self.rest_url = "https://www.okx.com"
        self._running = False

    def _sign(self, timestamp: str, method: str, path: str, body: str = "") -> str:
        """OKX API 签名"""
        message = timestamp + method + path + body
        mac = hmac.new(
            self.secret.encode(), message.encode(), hashlib.sha256
        )
        return mac.hexdigest()

    def _get_headers(self, method: str, path: str, body: str = "") -> dict:
        timestamp = str(time.time())
        sign = self._sign(timestamp, method, path, body)
        return {
            "OK-ACCESS-KEY": self.api_key,
            "OK-ACCESS-SIGN": sign,
            "OK-ACCESS-TIMESTAMP": timestamp,
            "OK-ACCESS-PASSPHRASE": self.passphrase,
            "Content-Type": "application/json",
        }

    async def place_order(self, inst_id: str, side: str, pos_side: str,
                          ord_type: str, sz: str, px: str = ""):
        """异步下单 - 关键性能点"""
        path = "/api/v5/trade/order"
        body = {
            "instId": inst_id,
            "tdMode": "isolated",    # 或 "cross"
            "side": side,
            "posSide": pos_side,
            "ordType": ord_type,     # "market" / "limit"
            "sz": sz,
        }
        if px:
            body["px"] = px
        body_str = json.dumps(body)
        headers = self._get_headers("POST", path, body_str)

        async with aiohttp.ClientSession() as session:
            async with session.post(
                self.rest_url + path,
                headers=headers,
                data=body_str
            ) as resp:
                result = await resp.json()
                latency = (time.time() - start_ts) * 1000
                print(f"[EXEC] Order placed in {latency:.2f}ms: {result}")
                return result

    async def on_signal(self, signal: dict):
        """信号处理 + 立即执行"""
        start = time.time()
        # 提取关键字段
        inst_id = signal.get("instId")
        side = signal.get("side")        # "buy" / "sell"
        sz = signal.get("sz")            # 数量
        px = signal.get("px", "")        # 市价单可为空

        # 判断开多/开空/平多/平空
        if side == "buy":
            pos_side = "long"
        else:
            pos_side = "short"

        await self.place_order(inst_id, side, pos_side, "market", sz)
        print(f"[LATENCY] Signal→Execution: {(time.time()-start)*1000:.2f}ms")

    async def run(self):
        """主循环：监听交易员 WebSocket 信号"""
        self._running = True
        # TODO: 连接交易员的持仓/订单 WebSocket 流
        # 实际需要申请 OKX 的跟单 API 或爬取公开信号
        async for msg in websockets.connect(self.ws_url):
            data = json.loads(msg)
            if data.get("event") == "error":
                continue
            # 解析交易信号
            await self.on_signal(data)
            if not self._running:
                break
```

### 延迟优化技巧

1. **预签名订单**：提前计算好签名，下单时只替换数量/价格字段
2. **连接池复用**：HTTP Keep-Alive + Session 复用，避免 TLS 握手
3. **同区域托管**：将执行服务部署在交易所同区域（如 AWS Tokyo、新加坡）
4. **FPGA/硬件加速**：专业做市商使用 FPGA 在网络层处理订单
5. **订单簿预热**：提前订阅深度数据，减少行情获取延迟

### 关键风险

- **跟单 vs 跟人**：交易员可能同时开多单，你的保证金率不同
- **仓位比例**：1:1 跟单不等于 1:1 盈亏（要考虑合约乘数、保证金）
- **穿仓风险**：极端行情下跟单者先爆
- **API 限频**：OKX 有每秒最多 2 次下单的限制（普通 API Key）

---

## 四、学习资源推荐

### 量化基础
- **书籍**：《主动投资组合管理》（Grinold & Kahn）、《量化交易》
- **课程**：QuantConnect / Backtrader 回测平台入门

### 跟单系统
- **参考项目**：Hummingbot（开源做市 + 跟单框架）
- **OKX API 文档**：<https://www.okx.com/docs-v5/zh/>
- **低延迟开发**：Linux 内核调优、socket options (TCP_NODELAY)

### 实践路径建议
```
1. 用 Backtrader 回测动量/反转策略
2. 用 OKX Demo Trading 测试 WebSocket 接收延迟
3. 用 Python asyncio 实现最小可行跟单
4. 对比手动下单 vs 程序跟单的滑点差异
5. 迭代优化延迟
```

---

## 五、待探索问题

- [ ] OKX 是否提供官方的"信号跟单"API（非人工中转）？
- [ ] 不同时间周期下，动量/反转效应的衰减曲线？
- [ ] 高频跟单中，滑点损耗 vs 手动延迟损耗哪个更大？
- [ ] Rust/Go 实现 vs Python asyncio 在毫秒级的实际差距？
