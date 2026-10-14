# Trend Resonance · 趋势共振

**A long-only multi-timeframe trend-following breakout strategy for cryptocurrency
perpetual futures.** This repository hosts the public writeup: architecture,
backtest results and risk disclosures.

📄 **Live page:** https://dio1009.github.io/resonance-breakout/ · [中文版](https://dio1009.github.io/resonance-breakout/zh/)

---

## What this is

A systematic, **long-only** trading system for crypto perpetual futures. The core
idea is to let a **higher timeframe decide direction** and a **lower timeframe pull
the trigger**: a higher-timeframe channel first answers "is the market trending or
ranging?", and only when direction is confirmed may a lower-timeframe breakout
signal fire. Once a breakout is confirmed, the system still waits for price to pull
back into a tolerance band before entering, so breakouts are not chased at the top.

In ranging markets this means long stretches with no position at all. That is the
design working, not a fault.

## Backtest results

380 days of replayed historical data across 15 instruments. Trading fees, funding
and stop slippage are all charged; stops trigger on bar highs and lows as they do
in live execution.

| Metric | Value | Metric | Value |
|---|---|---|---|
| Sample length | 380 days | Trades | 396 |
| Total return | **+250.89%** | Win rate | 44.44% |
| Annualized | **233.9%** | Payoff ratio | 1.92 |
| Max drawdown | **36.44%** | Profit factor | 1.54 |
| Closed by stop | 23 / 5.8% | Other exits | 373 |

**These are backtest figures, not realized account performance.** The system does
run on a live account, but the accumulated live trade count is too small to support
statistical conclusions, so live performance numbers are not published here.

### Profit is highly concentrated — this is the thing to know

- The **best two months** together account for **70.3%** of all profit, and **4 of
  the remaining 12 months were negative**
- The pattern repeats across instruments: the single largest contributor produced
  **34.6%** of total profit, while the bottom two were negative

Returns come from a handful of strong trending regimes, not from steady daily
accumulation. **Any arithmetic that extrapolates the annualized figure down to a
month or a week is wrong.**

## Method: six gates, every one of them rejects orders

A signal must clear six layers in order. Any gate that fails leaves a decision
record with its reason and no position.

| | Gate | What it does |
|---|---|---|
| 01 | **Market regime** | A breakout channel and a volatility channel on the higher timeframe jointly classify trending vs ranging. When ranging, every lower-timeframe breakout is rejected outright |
| 02 | **Breakout trigger** | Each session derives a pair of buy/sell rails from historical range, fixed for the day. Only a **close** beyond a rail counts; an intrabar wick does not |
| 03 | **Entry timing** | A confirmed breakout is not chased. Price must pull back into a tolerance band around a short-period moving average. Breakouts that stay outside are abandoned rather than filled worse |
| 04 | **Signal quality** | Trend conviction, breakout depth, pullback precision and volume expansion combine into a 0–1 score. Below threshold, no position |
| 05 | **Event avoidance** | Inside a fixed window before high-impact events such as Federal Reserve rate decisions, no new positions and no adds |
| 06 | **Position sizing** | Stop location is fixed first, then contract count is derived from the risk the trade may carry. Per-trade risk, per-position risk, per-instrument margin share and total exposure each clip the size in turn |

## Engineering: the strategy is half of it

Live trading and backtesting **run the same engine code**, and one configuration
both replays history and drives the live account — so backtest conclusions can be
verified bar by bar against live behaviour.

- **Full decision audit** — not just fills. Every bar and every instrument writes a
  decision record with each quality-score component and the reason for rejection.
  "Why was there no entry?" can be answered down to the specific gate
- **Market data resilience** — a dropped feed reconnects and backfills missing bars;
  an outage past a threshold trips a circuit breaker that halts new entries; brief
  flaps are debounced so they do not generate alert noise
- **Ledger reconciliation** — while flat, book equity is compared against the
  exchange balance every 15 seconds and any mismatch alerts immediately
- **Stops live at the exchange** — the initial stop rests as an exchange order, so a
  crash, network loss or restart does not affect it; after a restart the system
  adopts existing positions without duplicating orders
- **Hard capital limits** — four layers clip contract count before the order is sent;
  when a limit binds, the reason is recorded rather than the size silently shrunk
- **Parameter changes leave a trail** — every configuration change ships with its
  evidence: before-and-after backtests, the sample window and the basis for the
  decision, committed to version control

## Risk disclosure

**This repository is a technical writeup. It is not investment advice, an offer, or
a promise of returns.**

Crypto perpetual futures are leveraged derivatives with violent price swings. You
can lose your entire capital in a short time. This system trades with leverage and
its backtest maximum drawdown is **36.44%**, meaning an account could shrink by more
than a third under comparable conditions.

Backtest results are produced from historical data and cannot fully reproduce real
fill liquidity, latency or slippage in extreme conditions. **Past performance does
not indicate future results.** Parameters were selected over multiple passes on
historical data, so **overfitting is possible**.

**Keywords:** cryptocurrency trading, crypto perpetual futures, algorithmic trading,
quantitative trading, systematic trading, trading strategy, trading bot, trend
following, breakout strategy, multi-timeframe analysis, backtesting, risk
management, position sizing, technical analysis, quantitative finance.

---

## 中文说明

**一套只做多头的加密货币永续合约多周期趋势突破量化交易系统。** 本仓库是对外技术说明:架构、回测业绩与风险声明。

📄 **说明页:** https://dio1009.github.io/resonance-breakout/zh/ · [English](https://dio1009.github.io/resonance-breakout/)

### 核心思路

用**高周期的方向判定**去约束**低周期的入场扳机**。高周期通道先回答"现在是趋势还是震荡",只有方向明确时,低周期的突破信号才被允许生效;信号成立后还要等价格回踩确认,避免在冲高末端追入。横盘期因此会长时间空仓——这是设计结果,不是故障。

### 回测业绩

380 天历史数据回放,15 个交易标的,手续费、资金费与止损滑点全部计入,止损按 K 线最高最低价触发。

| 指标 | 数值 | 指标 | 数值 |
|---|---|---|---|
| 样本长度 | 380 天 | 交易笔数 | 396 |
| 总收益 | **+250.89%** | 胜率 | 44.44% |
| 年化 | **233.9%** | 盈亏比 | 1.92 |
| 最大回撤 | **36.44%** | 利润因子 | 1.54 |
| 止损了结 | 23 笔 / 5.8% | 其余出场 | 373 笔 |

**以上均为回测结果,不是已实现的账户业绩。** 该系统已在真实账户运行,但实盘累计成交笔数尚不足以支撑统计结论,故不发布实盘业绩数字。

### 利润高度集中,这是最该知道的一件事

- 最好的**两个月**合计占全部利润的 **70.3%**,其余 12 个月里有 **4 个月为负**
- 按标的看是同一形态:贡献最大的单一标的占总利润的 **34.6%**,排名末位的两个为负

收益来自少数强趋势行情,而不是稳定的每日积累。**任何把年化收益线性外推到月度或周度的算法都是错的。**

### 方法:六道门,每道都会拒单

| | 层 | 作用 |
|---|---|---|
| 01 | **市场状态** | 高周期的突破通道与波动率通道共同判定趋势/震荡。判为震荡时,低周期任何突破都被否决 |
| 02 | **突破扳机** | 每个交易时段由历史波幅推出当日固定的买卖轨道,价格**收于**轨道之外才算有效突破 |
| 03 | **入场时机** | 突破成立后不立即追入,要求价格回落到短周期均线附近的容许带内。带外的突破放弃,而不是以更差价格成交 |
| 04 | **信号质量** | 趋势确信度、突破深度、回踩精度、量能扩张合成 0–1 的质量分,低于门槛不开仓 |
| 05 | **事件回避** | 美联储议息等高影响事件公布前的固定窗口内,不开新仓也不加仓 |
| 06 | **仓位** | 先定止损位置,再由本笔愿意承担的风险反推张数。单笔风险、单仓风险、单标的保证金占比、总敞口四重上限依次裁剪 |

### 工程:策略只是一半

实盘与回测**共用同一套引擎代码**,同一份配置既能回放历史也能驱动实盘,因此回测结论可以逐根 K 线对齐验证。

- **全量决策审计** —— 不只记录成交,每根 K 线每个标的都写一条带原因的决策记录
- **行情容错** —— 断线自动重连补齐 K 线;持续中断超阈值触发熔断;短暂抖动去抖不报噪声
- **账本对账** —— 空仓时每 15 秒比对账面权益与交易所余额,不一致立即告警
- **止损在交易所侧** —— 以委托形式挂在交易所,程序崩溃断网重启都不影响生效
- **资金上限硬约束** —— 四层上限在下单前裁剪张数,触顶时记录原因而非静默缩单
- **参数变更留痕** —— 每次调整都带改前改后的回测对比与判定依据,一并进版本库

### 风险声明

**本仓库为技术说明,不构成投资建议、要约或收益承诺。** 加密货币永续合约为高杠杆衍生品,价格波动剧烈,可能在短时间内损失全部本金。回测最大回撤 36.44%,意味着在同等行情下账户可能出现超过三分之一的缩水。回测无法完全复现真实成交的流动性、延迟与极端行情下的滑点,**历史表现不代表未来收益**,参数存在**过度拟合**的可能。
