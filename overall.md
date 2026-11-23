# CalendarPulse

## 引言 · Introduction

### 日历如何塑造经济活动 · How calendars shape economic activity

许多人类活动遵循共同的日历安排，由此在观测序列中形成多个时间尺度上的周期规律。工作周塑造日常运营的节奏，节假日影响需求的时间分布，年度申报安排则使退税支付集中于一年中的特定阶段。这些规律会在同一日期交叠。例如，退税季中一个周一的支付规模，既受每周付款节奏影响，也与当年的退税进程有关。即使星期和时节相同，今年的支付高峰仍可能提前、推迟，或达到不同的规模。因此，预测一个未来日期，需要先依据其日历位置确定哪些历史记录具有参照价值，再结合近期走势，判断哪些规律仍在延续，以及当前业务在时间和规模上出现了哪些变化。

Many human activities follow shared calendars, giving rise to patterns that recur over several timescales. The working week shapes day-to-day operations, holidays shift the timing of demand, and annual filing schedules concentrate tax refunds in particular parts of the year. These patterns overlap on any given date. The amount paid on a Monday during refund season, for example, reflects both the weekly payment cycle and how far the current season has progressed. Even for the same weekday at the same time of year, the peak may arrive earlier, come later, or reach a different level. Forecasting a particular date therefore requires identifying past observations that are relevant to its place in the calendar, then using the recent trajectory to assess which patterns persist and how their timing or magnitude has changed.

### 近期走势与历史参照 · Recent trajectories and historical references

仅使用近期窗口的模型能够捕捉当前走势，却可能遗漏往年同期的关键记录。按日历对齐后取历史均值，可以引入这些长期参照，但无论当前业务正在加速还是放缓，各条记录都按相同的规则汇总。这两种做法都难以根据当前业务进程调整历史参照对预测的影响。

A model that reads only a recent window can capture the current trajectory, yet observations from comparable periods in earlier years may lie beyond its input. Averaging calendar-aligned observations brings those records into the forecast, but applies the same aggregation rule whether activity is accelerating or slowing. Both approaches leave a gap in how the current trajectory informs the use of historical references.

### 双路信息聚合 · Dual-route information aggregation

大语言模型为聚合不同来源的信息提供了计算基础。数值观测与描述其日历位置和业务类别的文本标签可以映射到共同的表示空间，由同一套注意力机制处理。基于这一能力，我们提出 **CalendarPulse**，并构建两条信息通道。近期通道编码连续观测，刻画当前业务走势。周期通道则为每个目标日组织与其对齐的周度和年度历史记录。我们直接利用语言模型原有的查询、键、值计算和键值缓存机制，将两条通道的交互嵌入共享主干。周期通道首先运行，在各全局注意力层缓存历史参照的键和值。近期通道随后以自身状态生成查询，在对应层与缓存的键计算注意力权重，并据此聚合相应的值。由此，目标日期决定哪些历史记录进入参照范围，当前走势决定这些记录如何共同形成目标日的预测表示。

Large language models provide a computational basis for aggregating information from different sources. Numerical observations and text labels describing their calendar positions and business categories can be mapped into a common representation space and processed by the same attention mechanism. We build on this capability in **CalendarPulse**, which organizes its inputs into two routes. The recent route encodes successive observations to represent the current trajectory. For each target date, the periodic route assembles weekly and annual records aligned with that date. The routes interact inside a shared backbone through the language model’s native query, key, and value computations and its key–value cache. The periodic route runs first, caching keys and values for the historical references at each global attention layer. At the corresponding layer, the recent route forms queries from its own state, computes attention weights against the cached keys, and uses those weights to aggregate the values. The target date thus determines which records enter the reference set, while the current trajectory guides how they contribute to the representation used for that date’s forecast.

### 从退税预测到资源配置 · From refund forecasting to resource allocation

我们以美国财政部的所得税退税为切入点，预测未来五个报告日内，个人与企业通过支票和电子转账获得的逐日退税金额。对公共资金管理而言，即使五天的累计支出估计准确，错判付款高峰的日期，仍可能低估当日的现金需求。在 246 个留出预测窗口上，CalendarPulse 较同星期几均值基线将日总额平均绝对误差降低了 45.6%，平均每个目标日的绝对误差减少约 6.4 亿美元。

We first apply CalendarPulse to U.S. Treasury income tax refunds, forecasting daily payments to individuals and businesses by check and electronic funds transfer over the next five reporting days. For public cash managers, even an accurate estimate of cumulative five-day outflows can understate cash needs on the busiest day if the forecast places the peak on the wrong date. Across 246 held-out forecast windows, CalendarPulse reduces mean absolute error in daily total payments by 45.6% relative to the mean of recent observations from the same weekday. This amounts to approximately $640 million less absolute error per target day on average.

退税所体现的时间规律广泛存在于人类经济活动中。零售备货围绕节假日展开，企业结算遵循账期，公共福利按发放日历支付。这些活动共享一种时间结构：日历规定了反复出现的业务节点，而每一轮活动何时加速、达到多大规模，则取决于当期进程。正是这种共性，使 CalendarPulse 的预测能力有望从退税延伸到商品需求、企业现金流和公共支出预测。对于需要提前配置资金、库存或运力的组织，更早识别下一次高峰的时间和规模，意味着可以在需求兑现之前作出资源配置决策。

The temporal patterns seen in tax refunds recur across economic activity. Retailers stock ahead of holidays, businesses settle accounts on billing cycles, and public benefits follow disbursement calendars. These activities share a timing structure. The calendar identifies recurring events, while the course of the current cycle shapes when activity accelerates and how large it becomes. This common structure gives CalendarPulse a path from refund forecasting to retail demand, corporate cash flow, and public spending. For organizations that allocate cash, inventory, or transport capacity in advance, earlier estimates of a peak’s timing and size can inform those commitments while there is still time to act.
