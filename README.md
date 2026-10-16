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

Two configurations over the same window: the system as it ran until October 2026,
and the current one with a **second quality gate** added on top (explained below).

| Metric | Until Oct 2026 | **Current** |
|---|---|---|
| Sample length | 380 days | 380 days |
| Trades | 396 | 372 |
| Total return | +221.44% | **+293.24%** |
| Annualized | 207.0% | **272.6%** |
| Max drawdown | 36.45% | **29.47%** |
| Win rate | 43.43% | 44.62% |
| Payoff ratio | 1.95 | 1.95 |
| Profit factor | 1.5 | 1.57 |
| Closed by stop | 22 / 5.6% | 22 / 5.9% |

**These are backtest figures, not realized account performance.** The system does
run on a live account, but the accumulated live trade count is too small to support
statistical conclusions, so live performance numbers are not published here.

Market entries are market orders, so a spread-crossing cost is charged on every
entry in addition to fees, funding and stop slippage. Earlier versions of this page
quoted figures without that entry cost; over 380 days it is worth roughly
30 percentage points of total return, and the numbers above include it.

### Profit is highly concentrated — this is the thing to know

- The **best two months** together account for **65.7%** of all profit, and
  **4 of the remaining 12 months were negative**
- The pattern repeats across instruments: the single largest contributor produced
  **35.6%** of total profit, while 3 of them were negative

Returns come from a handful of strong trending regimes, not from steady daily
accumulation. **Any arithmetic that extrapolates the annualized figure down to a
month or a week is wrong.**

## Method: seven gates, every one of them rejects orders

A signal must clear seven layers in order. Any gate that fails leaves a decision
record with its reason and no position.

| | Gate | What it does |
|---|---|---|
| 01 | **Market regime** | A breakout channel and a volatility channel on the higher timeframe jointly classify trending vs ranging. When ranging, every lower-timeframe breakout is rejected outright |
| 02 | **Breakout trigger** | Each session derives a pair of buy/sell rails from historical range, fixed for the day. Only a **close** beyond a rail counts; an intrabar wick does not |
| 03 | **Entry timing** | A confirmed breakout is not chased. Price must pull back into a tolerance band around a short-period moving average. Breakouts that stay outside are abandoned rather than filled worse |
| 04 | **Signal quality** | Trend conviction, breakout depth, pullback precision and volume expansion combine into a 0–1 score. Below threshold, no position |
| 05 | **Second quality gate** | A separate score, built from a broad factor library reduced to independent clusters, reviews what gate 04 passed. It may only **reject**, never admit. Signals in its weakest band are dropped and the instrument is left free for the next qualifying bar |
| 06 | **Event avoidance** | Inside a fixed window before high-impact events such as Federal Reserve rate decisions, no new positions and no adds |
| 07 | **Position sizing** | Stop location is fixed first, then contract count is derived from the risk the trade may carry. Per-trade risk, per-position risk, per-instrument margin share and total exposure each clip the size in turn |

## The second quality gate, and why it may only reject

Gate 04 scores a signal from its own entry geometry. Gate 05 was added after
measuring a much larger factor library against realized outcomes, and two findings
shaped it.

**Most of a large factor library is the same factor wearing different clothes.** A
large share of the candidates collapse into a single correlated cluster: at this
system's holding horizon, "trend strength" and "price extension" turn out to be the
same quantity measured two ways. A naive equal-weight vote across the library would
therefore hand most of the weight to that one crowded cluster. So the library is
first reduced to independent clusters, and weight is assigned per cluster rather
than per factor.

**The new dimensions can tell you what to skip, but not what to take.** Designs
that let the new score *replace* the original one — so it both rejected and admitted
— all came out worse. Trade-level attribution showed why: the trades those designs
dropped were genuinely below average, but the trades they let in instead were worth
close to nothing. Rejection carried skill; admission carried none. The shipped
design therefore has one power only: it can veto what gate 04 passed, and it can
never promote anything.

**It re-times entries rather than cutting exposure.** Each instrument holds at most
one position, so a vetoed signal does not reduce participation — it frees the
instrument, and the next bar on that instrument which clears every gate takes the
slot, usually within the hour. This is also why it behaves differently from simply
delaying entries by a fixed clock, which was tested separately and was clearly
harmful.

**It is a drawdown filter more than a return amplifier.** The gain is largest in the
weakest market regimes and roughly neutral in the strongest one. Over the window
above it lowered maximum drawdown by about seven percentage points while taking
slightly fewer trades.

### How it was accepted

The acceptance bar was fixed before the runs, not chosen afterwards:

- **Compared against a random-rejection control at matched trade count**, not only
  against the previous configuration. Being pickier lowers drawdown on its own, so
  without that control a design that merely trades less looks like a design that
  trades better.
- **Scored on zero-overlap windows**, with one window disqualified from ranking
  because a single trade dominated its result.
- **Checked for concentration** — a design whose advantage came mostly from one
  trade or one instrument was treated as an artifact, not a finding.
- Alternatives were discarded rather than re-tuned. Several scores that looked
  promising against the previous configuration failed the random-rejection control
  and were dropped, including the one expected beforehand to work best.

Live since 2026-10-16.

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

同一窗口下的两个配置:2026 年 10 月之前运行的版本,以及在其之上加了**第二道质量门**的当前版本(下文说明)。

| 指标 | 2026-10 之前 | **当前** |
|---|---|---|
| 样本长度 | 380 天 | 380 天 |
| 交易笔数 | 396 | 372 |
| 总收益 | +221.44% | **+293.24%** |
| 年化 | 207.0% | **272.6%** |
| 最大回撤 | 36.45% | **29.47%** |
| 胜率 | 43.43% | 44.62% |
| 盈亏比 | 1.95 | 1.95 |
| 利润因子 | 1.5 | 1.57 |
| 止损了结 | 22 笔 / 5.6% | 22 笔 / 5.9% |

**以上均为回测结果,不是已实现的账户业绩。** 该系统已在真实账户运行,但实盘累计成交笔数尚不足以支撑统计结论,故不发布实盘业绩数字。

开仓走市价单,因此除手续费、资金费与止损滑点之外,每笔入场还计入一次跨价差成本。本页早前版本引用的数字未含这项入场成本——在 380 天样本上它约值 30 个百分点的总收益,上表已计入。

### 利润高度集中,这是最该知道的一件事

- 最好的**两个月**合计占全部利润的 **65.7%**,其余 12 个月里有 **4 个月为负**
- 按标的看是同一形态:贡献最大的单一标的占总利润的 **35.6%**,其中 3 个标的为负

收益来自少数强趋势行情,而不是稳定的每日积累。**任何把年化收益线性外推到月度或周度的算法都是错的。**

### 方法:七道门,每道都会拒单

| | 层 | 作用 |
|---|---|---|
| 01 | **市场状态** | 高周期的突破通道与波动率通道共同判定趋势/震荡。判为震荡时,低周期任何突破都被否决 |
| 02 | **突破扳机** | 每个交易时段由历史波幅推出当日固定的买卖轨道,价格**收于**轨道之外才算有效突破 |
| 03 | **入场时机** | 突破成立后不立即追入,要求价格回落到短周期均线附近的容许带内。带外的突破放弃,而不是以更差价格成交 |
| 04 | **信号质量** | 趋势确信度、突破深度、回踩精度、量能扩张合成 0–1 的质量分,低于门槛不开仓 |
| 05 | **第二道质量门** | 另一套分数复核第 04 道放行的信号,它由一个较大的因子库先归约成相互独立的簇后构成。该门**只能否决、不能放行**:落在它最弱区间的信号被剔除,该标的随即空出,留给下一根通过全部门的 K 线 |
| 06 | **事件回避** | 美联储议息等高影响事件公布前的固定窗口内,不开新仓也不加仓 |
| 07 | **仓位** | 先定止损位置,再由本笔愿意承担的风险反推张数。单笔风险、单仓风险、单标的保证金占比、总敞口四重上限依次裁剪 |

### 第二道质量门,以及它为何只能否决

第 04 道门用信号自身的入场几何形态打分。第 05 道门是在把一个大得多的因子库对照真实盈亏做过测量之后加上的,两个发现决定了它的形态。

**一个大因子库里的大部分因子,是同一个量换了件衣服。** 候选因子中有很大一部分坍缩进同一个高相关簇:在本系统的持仓尺度上,"趋势强度"与"价格伸展"其实是同一个量的两种量法。因此对整个因子库做朴素等权投票,等于把大部分权重交给那一个拥挤的簇。所以因子库先被归约成相互独立的簇,权重按簇分配而不是按因子分配。

**新维度能告诉你该跳过什么,但不能告诉你该拿什么。** 让新分数**替换**原分数的那些设计——即新分数既能否决也能放行——全部更差。逐笔归因给出了原因:这些设计剔掉的交易确实低于平均,但它们换进来的交易价值接近于零。否决有技能,放行没有。因此最终上线的设计只有一种权力:否决第 04 道门放行的信号,永远不能提拔任何信号。

**它改的是入场时点,不是敞口。** 每个标的同时只持一仓,所以被否决并不减少参与度——它让该标的空出来,该标的上下一根通过全部门的 K 线接走这个位置,通常在一小时之内。这也是它与"按固定时钟延迟入场"本质不同的地方,后者单独测过,明显有害。

**它更像回撤过滤器而不是收益放大器。** 收益在最弱的市场状态下提升最大,在最强的状态下基本中性。在上文窗口里,它把最大回撤压低约七个百分点,同时成交笔数略少。

#### 它是怎么被接受的

准入标准在跑之前就定死,不是事后挑的:

- **与同等成交笔数的随机否决对照比**,而不只是与旧配置比。更挑剔本身就能降低回撤,没有这个对照,一个只是少交易的设计会被误认为是交易得更好的设计。
- **在零重叠窗口上评分**,其中一个窗口因为结果被单笔交易主导而被取消排序资格。
- **检查集中度** —— 优势主要来自某一笔或某一个标的的设计,被当作假象而不是发现。
- 不合格的方案被丢弃,而不是回去调参。有几套分数对旧配置看起来不错,但没过随机否决对照而被放弃,其中包括事前最被看好的那一套。

2026-10-16 上线。

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
