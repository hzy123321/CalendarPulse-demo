# CalendarPulse：按目标日期读取历史的经济活动预测
# CalendarPulse: Forecasting Economic Activity with Target-Specific History

## 摘要 · Abstract

CalendarPulse 面向一类受日历安排影响的多步预测任务：未来几天共享截至今天的近期走势，每一天却有各自值得参照的历史。我们以美国财政部所得税退税为切入点，将最近 60 个报告日的付款轨迹与按目标日选取的周度、年度记录送入共享的 Gemma 主干。近期分支查询周期分支缓存的键和值，逐日预测个人和企业通过支票及电子转账获得的四类退款。在 246 个留出预测窗口上，CalendarPulse 将日总额平均绝对误差从同星期几基线的 14.03 亿美元降至 7.63 亿美元，降幅为 45.6%。交互系统让读者可以从一项预测回到发布时点，检查当时的近期走势、历史参照，以及随后实际支付的金额。

CalendarPulse addresses multi-step forecasting tasks shaped by the calendar. Future dates share the trajectory observed up to issuance, but each has its own historical reference points. We study U.S. Treasury income tax refunds and use a shared Gemma backbone to pair the most recent 60 reporting days with weekly and annual records selected for each target. Queries from the recent route read keys and values cached by the periodic route. The model predicts four daily amounts covering checks and electronic transfers to individuals and businesses. Across 246 held-out windows, CalendarPulse reduces daily total mean absolute error from $1.403 billion for the same-weekday baseline to $763 million, a reduction of 45.6%. An interactive replay lets readers inspect the recent trajectory and references available when a forecast was issued, then compare the forecast with the payments subsequently reported.

## 任务与动机 · Task and motivation

工作周塑造日常运营的节奏，年度申报安排使退税集中于一年中的特定阶段。两种周期会在同一天交叠。一个周一的退税规模既与每周付款节奏有关，也取决于当年的退税季走到了哪一步。最近几个周一提供了星期层面的参照，去年同期的周一则提供了季节层面的参照；今年最近的支付记录帮助判断，这些历史规律中哪些仍在延续，哪些已在时间或规模上发生变化。

The working week shapes day-to-day operations, while annual filing schedules concentrate refunds in particular parts of the year. Both cycles meet on the same date. Payments on a Monday depend on the weekly payment cycle and on how far the current refund season has progressed. Recent Mondays offer a weekday reference, and Mondays around the same time last year provide a seasonal reference. This year’s recent payments help establish which patterns persist and which have shifted in timing or magnitude.

仅使用近期窗口，可能看不到往年同期曾有多大的支付需求。按日历选取历史并取均值，能够纳入这些记录，却会用同一条平均规则处理今年正在加速和正在回落的付款。CalendarPulse 将这两个问题分开处理：目标日决定哪些历史记录可供参照，近期走势通过注意力计算决定如何使用它们。

A recent window may omit the payment volumes observed at the same point in earlier seasons. A calendar-aligned mean can incorporate historical records, but applies the same averaging rule whether payments are accelerating or slowing this year. CalendarPulse assigns these two decisions distinct roles. The target date determines which records are available for comparison, and attention conditioned on recent activity determines how the model uses them.

## 双路 Gemma 如何形成预测 · How dual-route Gemma produces a forecast

**为每个目标日组织两路输入。** 在同一个预测发布时点，未来五个报告日共享最近 60 个可用报告日的四渠道支付序列。周期分支则为五个目标日分别准备输入。以预测周一为例，先取截至发布时点最近四个有报告的周一，再逐年回看过去三年。每年找到最接近目标日期周年日的周一，取该周和前后各两周的周一。每类支付由此得到最多 19 个参照位置，即四个近期位置和三个年度各五个位置。缺报位置带有显式标记，与有报告但支付金额为零的位置区分开来。

**Constructing the two inputs for each target.** At a given forecast origin, all five target reporting days share the same four-channel history from the latest 60 available reports. The periodic route receives a separate input for each target. For a Monday, it takes the four most recent available Monday reports. In each of the preceding three years, it finds the Monday closest to the target date’s anniversary and takes that Monday plus the two Mondays on either side. Each payment category therefore has up to 19 reference slots, comprising four recent slots and five slots for each of three earlier years. An explicit availability mask distinguishes a missing report from a reported amount of zero.

**周期分支先建立逐层可读的历史表示。** 两路共用 OpenTSLM 的时序编码器、投影层和冻结的 Gemma 3 270M。周期分支把各组历史金额编码为数值片段，并与标明支付类别、参照周期和缺报位置的文本标签交错排列。该序列先通过 Gemma。在第 6、12、18 层，也就是主干的三个全局注意力层，模型保存该层为周期序列生成的键和值。每个目标日由此获得三组按层对应的缓存。缓存中的键和值表示编码后的序列位置；一个注意力权重并不直接等于某个历史日期对美元预测的贡献。

**Building layer-specific historical memory.** Both routes share an OpenTSLM time-series encoder and projector and a frozen Gemma 3 270M backbone. The periodic route encodes groups of historical amounts as numerical segments and interleaves them with text identifying the payment category, reference period, and slot availability. This sequence runs through Gemma first. At layers 6, 12, and 18, the backbone’s three global attention layers, CalendarPulse saves the keys and values that each layer produces for the periodic sequence. Each target date thus receives three layer-specific caches. Their entries correspond to encoded sequence positions, so an attention weight alone does not quantify the dollar contribution of an individual historical date.

**近期分支以自己的查询逐层读取历史。** 近期分支随后处理最近 60 个报告日。在局部层，每个位置通过滑动窗口内的因果自注意力整合此前的近期观测。进入第 6 层时，近期分支用自身的当前隐藏状态生成查询 Q；这一层用来响应查询的键 K 和值 V，取自周期分支在第 6 层保存的缓存。每个有效查询都与周期序列各有效位置的键计算匹配分数。模型归一化这些分数，得到注意力权重，按权重汇总相应的值，并通过输出投影和残差连接更新近期状态。第 12 层和第 18 层分别读取周期分支的同层缓存。由于新的查询来自已经更新的近期状态，后一次读取可以利用此前已经融入的历史信息。

**Reading historical memory with recent queries.** The recent route then processes the last 60 reporting days. In local layers, causal sliding-window self-attention lets each position integrate preceding observations within its window. Upon reaching layer 6, the recent route forms queries Q from its current hidden states. The keys K and values V used to answer those queries come from the periodic route’s layer-6 cache. Each valid query is scored against the keys at all valid memory positions. Attention normalizes the scores, weights the corresponding values, and updates the recent state through the output projection and residual connection. Layers 12 and 18 read their respective periodic caches in the same way. Because later queries arise from the updated recent states, each later read can build on historical information incorporated at earlier layers.

**从融合表示得到金额。** 模型取近期分支最后一个有效位置的隐藏状态，经投影后与目标日的日历特征拼接，由共享回归头直接输出个人支票、个人电子转账、企业支票和企业电子转账四类金额。训练同时考虑各渠道和日总额的误差。预测目标从周一换成周四时，两次预测从相同的近期输入出发，却在第 6 层开始读取不同的历史参照。对五个报告日重复这一过程，一个预测窗口就得到 20 个金额，可以逐渠道检查，也可以汇总成五天的资金需求。

**From fused states to payment amounts.** The model projects the hidden state at the last valid position of the recent route and concatenates it with calendar features for the target date. A shared regression head directly predicts individual checks, individual electronic funds transfers, business checks, and business electronic funds transfers. Training accounts for errors in both channel amounts and their daily sum. A Monday forecast and a Thursday forecast begin with the same recent input, then read different historical references starting at layer 6. Repeating the computation for five reporting days produces 20 amounts that can be inspected by channel or aggregated into a five-day cash requirement.

## 系统展示 · Interactive system

[首页 · CalendarPulse](index.html) 从 2026 年 5 月 6 日发布的一次预测切入。对于 5 月 13 日，最近四个有报告的周三给出的均值是 130.22 亿美元，CalendarPulse 预测约 27.77 亿美元。读者点击后可揭示实际支付的 27.81 亿美元，并查看五天走势。这个案例按明确规则选出：在同星期几基线绝对误差至少为 10 亿美元的测试预测中，取 CalendarPulse 绝对误差最小的一次。首页同时展示全测试集的比较结果。

[Home · CalendarPulse](index.html) opens with a forecast issued on May 6, 2026. For May 13, the mean of the four most recent reported Wednesdays was $13.022 billion, while CalendarPulse forecast approximately $2.777 billion. Readers can reveal the reported $2.781 billion and inspect the five-day trajectory. The case was selected by a stated rule: among test forecasts where the same-weekday baseline missed by at least $1 billion, it had the smallest CalendarPulse absolute error. The homepage also reports the full-test-set comparison.

[工作原理 · How it works](method.html) 使用另一组目标日来解释计算过程。切换 2026 年 2 月 19 日周四和 2 月 23 日周一，两者都使用 2 月 18 日发布预测时可见的同一段近期历史，而周度与年度参照、四渠道预测和日总额随目标日更新。读者可播放两路计算的顺序，检查某个历史位置的日期和支付金额，并打开财政部对应日期的数据行。

[How it works](method.html) uses a different pair of targets to explain the computation. Thursday, February 19, and Monday, February 23, 2026, share the recent history available when the forecast was issued on February 18. Switching targets changes the weekly and annual reference records, the four channel forecasts, and the daily total. Readers can follow the order of the two forward passes, inspect the date and observed amounts behind a historical slot, and open the Treasury records for that date.

[交互回放 · Refund Replay](replay.html) 支持在全部 246 个留出窗口间切换，按退税类别查看历史和未来五日预测。默认比较模式把 CalendarPulse 的青色实线、匹配单路模型的绿色虚线和实际支付绘在同一张图上。选择任一目标日即可对照两种预测与实绩；也可以隐藏实绩逐日回放。[结果与来源 · Evaluation](evaluation.html) 则把单个案例放回完整测试集，支持切换三种指标并导出 4,920 条分渠道记录。

[Refund Replay](replay.html) covers all 246 held-out windows. Readers can select a refund category and inspect its history and five-day outlook. By default, the chart overlays CalendarPulse as a solid teal line, the matched single-route model as a dashed green line, and reported payments. Selecting a target date brings the two forecasts and the actual amount into direct comparison. Readers can also hide the reported series and replay the forecasts one day at a time. [Evaluation](evaluation.html) places individual cases in the context of the full test set, with three selectable metrics and an export of all 4,920 channel-level records.

## 回测结果与资金决策 · Backtest results and cash-planning decisions

我们用美国财政部《每日财政报表》的固定快照构建了 3,466 个预测窗口，按时间顺序划分为 2,974 个训练窗口、246 个验证窗口和 246 个测试窗口。测试目标日期从 2025 年 11 月 18 日至 2026 年 11 月 17 日。匹配的单路和双路模型均以随机种子 42 训练 15 轮，按验证集日总额 MAE 选择检查点。每个测试窗口包含五个目标日和四类金额，共 1,230 个“发布时点—目标日期”组合。同一日期从不同发布时点作出的预测分别计分。

We constructed 3,466 forecast windows from a fixed snapshot of the U.S. Treasury’s Daily Treasury Statement and split them chronologically into 2,974 training, 246 validation, and 246 test windows. Test target dates span November 18, 2025 through November 17, 2026. The matched single- and dual-route models trained for 15 epochs with seed 42, with checkpoint selection based on validation daily total MAE. Each test window contains five target reporting days and four payment categories, giving 1,230 origin–target pairs. Forecasts for the same date from different origins are scored separately.

| 预测方法 / Forecast | 日总额 MAE / Daily total MAE (USD billions) | 日总额 WAPE / Daily total WAPE | 五日累计 MAE / Five-day total MAE (USD billions) |
| --- | ---: | ---: | ---: |
| 同星期几均值 / Same-weekday mean | 1.403 | 63.9% | 5.541 |
| 匹配单路模型 / Matched single route | 0.959 | 43.7% | 3.474 |
| CalendarPulse | **0.763** | **34.8%** | **2.502** |

每天要准备多少钱？先汇总当天四个渠道，再计算日总额绝对误差，并对全部 1,230 个组合取平均。CalendarPulse 的日总额 MAE 较同星期几基线下降 45.6%，较匹配的单路模型下降 20.4%。日总额 WAPE 以各次日总额绝对误差之和除以对应实绩绝对值之和。未来五天合计要准备多少钱？先把整个窗口的 20 个金额相加，再计算累计绝对误差，并对 246 个窗口取平均。这一口径下，CalendarPulse 将同星期几基线的 MAE 从 55.41 亿美元降至 25.02 亿美元。两种资金问题需要分别评估，因为高估一天、低估另一天可能在五日累计时抵消。

How much cash is needed on a given day? We first sum the four channels, calculate the absolute error in that day’s total, and average over all 1,230 pairs. CalendarPulse reduces daily total MAE by 45.6% against the same-weekday mean and by 20.4% against the matched single-route model. Daily total WAPE divides the sum of absolute daily errors by the sum of absolute reported daily totals. How much cash is needed across five days? We sum the 20 amounts in each window before calculating its absolute error, then average over 246 windows. On this measure, CalendarPulse reduces MAE from $5.541 billion for the same-weekday mean to $2.502 billion. The two decisions warrant separate evaluation because an overestimate on one day can offset an underestimate on another.

## 适用范围与前景 · Scope and outlook

峰值风险仍是资金调度中的关键问题。在实际五日支付总额最高的测试窗口中，CalendarPulse 预测 347.26 亿美元，实际支付 886.06 亿美元。总体误差的改善与极端窗口中的资金缺口需要同时考察。当前证据来自一个美国财政数据快照和一次匹配种子实验；回测使用实际观测到的报告日和假设的公布时滞。面向实际部署，下一步是用预测发布时可知的日历与按时点归档的数据做前瞻验证。零售备货、企业账款结算和公共福利支付也有明确的日历节点，并随当前周期出现提前、推迟或规模变化，可作为这套机制的后续检验场景。

Peak risk remains consequential for cash planning. In the test window with the largest observed five-day outflow, CalendarPulse projected $34.73 billion and the Treasury reported $88.61 billion. Average error reductions and shortfalls in extreme windows both matter. The current evidence comes from one U.S. Treasury snapshot and one matched training seed. The backtest uses observed reporting dates and an assumed publication lag. Prospective evaluation requires calendars known at issuance and archived as-of releases. Retail stocking, invoice settlement, and benefit disbursement offer further settings in which to test the approach: each has recurring calendar dates, yet activity can shift in timing and scale from one cycle to the next.
