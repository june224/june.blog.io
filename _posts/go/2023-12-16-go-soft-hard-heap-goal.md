---
title: 提议：分离出柔性堆和硬性堆目标大小【译】
author: june
date: 2023-12-16
category: go
layout: post
---

原文：《[Proposal: Separate soft and hard heap size goal](https://go.googlesource.com/proposal/+/master/design/14951-soft-heap-limit.md)》

### Background

The GC pacer is responsible for determining when to start a GC cycle and how much back-pressure to put on allocation to prevent exceeding the goal heap size. It aims to balance two goals:
* Complete marking before the allocated heap exceeds the GOGC-based goal heap size.
* Minimize GC CPU consumed beyond the 25% reservation.

GC pacer负责确定何时开始GC周期以及对分配施加多少背压以防止超过目标堆大小。它旨在平衡两个目标：
* 在分配的堆超过基于GOGC的目标堆大小之前完成标记。
* 最大限度地减少GC CPU消耗超出25%预留。

> 注：back-pressure，或者backpressure，是指对通过管道的所需流体流动的阻力的术语。这里的背压是指施加mutator assists的压力，增加扫描标记的工作量。

In order to satisfy the first goal, the pacer forces the mutator to assist with marking if it is allocating too quickly. These mark assists are what cause GC CPU to exceed the 25%, since the scheduler dedicates 25% to background marking without assists. Hence, to satisfy the second goal, the pacer's trigger controller sets the GC trigger heap size with the goal of starting GC early enough that no assists are necessary. In addition to reducing GC CPU overhead, minimizing assists also reduces the per-goroutine latency variance caused by assists.

为了满足第一个目标，如果分配速度太快，pacer会强制mutator协助标记。这些标记辅助是导致GC CPU超过25% 的原因，因为调度程序将25%CPU的时间专门用于没有辅助的后台标记。因此，为了满足第二个目标，pacer的触发控制器设置GC 触发堆大小，目标是足够早地启动GC，以便不需要辅助。除了减少GC CPU开销之外，最小化辅助还可以减少由辅助引起的每个协程的延迟差异。

In practice, however, the trigger controller does not achieve the goal of minimizing mark assists because it stabilizes on the wrong steady state. This document explains what happens and why and then proposes a solution.

然而，在实践中，触发控制器并没有实现最小化标记辅助的目标，因为它稳定在错误的稳定状态。本文档解释了发生的情况和原因，然后提出了解决方案。

For a detailed description of the pacer, see [ the pacer design document ](2023-12-03-go-current-gc-pacing.html). This document follows the nomenclature set out in the original design, so it may be useful to review the original design document first.

有关pacer的详细说明，请参阅[《 the pacer design document》](2023-12-03-go-current-gc-pacing.html)。本文档遵循原始设计中规定的术语，因此首先查看原始设计文档可能会很有用。

### Problem
The trigger controller is a simple proportional feedback system based on two measurements that directly parallel the pacer's two goals:
1. The actual heap growth ha at which marking terminates, as a fraction of the heap goal size. Specifically, it uses the overshoot ratio $$ h = (h_a − h_T) / (h_g−h_T) $$, which is how far between the trigger $$ h_T $$ and the goal $$ h_g $$ the heap was at completion. Ideally, the pacer would achieve $$ h = 1 $$.
2. The actual GC CPU consumed ua as a fraction of the total CPU available. Here, the goal is fixed at $$ u_g = 0.25 $$.

触发控制器是一个基于与pacer的两个目标直接平行的两个测量值的简单的比例反馈系统：
1. 标记终止时的实际堆增长大小记为ha，作为堆目标大小的一部分。具体来说，它使用超调比率$$ h = (h_a − h_T) / (h_g−h_T) $$，即堆完成时触发$$ h_T $$与目标$$ h_g $$之间的距离。理想情况下，pacer应达到$$ h = 1 $$。
2. 实际GC CPU消耗时间ua占可用CPU总量的一小部分。这里，目标固定为$$ u_g = 0.25 $$。

Using these, the trigger controller computes the error in the trigger and adjusts the trigger based on this error for the next GC cycle. Specifically, the error term is

使用这些，触发控制器计算触发器中的误差项，并根据该误差项调整下一个GC周期的触发器。具体而言，误差项为

![softhardheap_f1](/assets/post/go/softhardheap_f1.png "softhardheap_f1")

However,$$ e(n) = 0 $$not only in the desired case of$$ h = 1 $$,$$ u_a = u_g $$, but in any state where $$h = u_g / u_a $$. As a result, the trigger controller can stabilize in a state that undershoots the heap goal and overshoots the CPU goal. We can see this in the following plot of $$ e(n) $$, which shows positive error in blue, negative error in red, and zero error in white:

然而，不仅在$$ h = 1 $$,$$ u_a = u_g $$的理想情况下，而且在$$h = u_g / u_a $$的任何状态下，都有$$ e(n) = 0 $$。因此，触发控制器可以稳定在低于堆目标并超过CPU目标的状态。我们可以在下面的$$ e(n) $$图中看到这一点，其中蓝色显示正误差，红色显示负误差，白色显示零误差：

![softhardheap_f2](/assets/post/go/softhardheap_f2.png "softhardheap_f2")

Coupled with how GC paces assists, this is exactly what happens when the heap size is stable. To satisfy the heap growth constraint, assist pacing conservatively assumes that the entire heap is live. However, with a GOGC of 100, only half of the heap is live in steady state. As a result, marking terminates when the allocated heap is only half way between the trigger and the goal, i.e., at $$ h = 0.5 $$ (more generally, at $$ h = 100/(100+GOGC)) $$. This causes the trigger controller to stabilize at $$ u_a = 0.5 $$, or 50% GC CPU usage, rather than $$ u_a = 0.25 $$. This chronic heap undershoot leads to chronic CPU overshoot.

再加上GC paces assists，这正是堆大小稳定时发生的情况。为了满足堆增长约束，assist paces保守地假设整个堆都处于活动状态。然而，当GOGC为100时，只有一半的堆处于稳定状态。因此，当分配的堆仅位于触发器和目标之间的一半时，即$$ h = 0.5 $$（更一般地，$$ h = 100/(100+GOGC)）$$ 时，标记终止。这会导致触发控制器稳定在$$ u_a = 0.25 $$，即50%的GC CPU使用率，而不是$$ ua = 0.25 $$。这种慢性堆下冲会导致慢性CPU超调。

### Example

The garbage benchmark demonstrates this problem nicely when run as garbage` -benchmem 512 -benchtime 30s `. Even once the benchmark has entered steady state, we can see a significant amount of time spent in mark assists (the narrow cyan regions on every other row):

当垃圾基准测试以垃圾` -benchmem 512 -benchtime 30s `运行时，很好地演示了这个问题。即使基准进入稳定状态，我们也可以看到在标记辅助上花费了大量时间（每隔一行的狭窄青色区域）：

![softhardheap_f3](/assets/post/go/softhardheap_f3.png "softhardheap_f3")

Using GODEBUG=gcpacertrace=1 , we can plot the exact evolution of the pacing parameters:

使用 GODEBUG=gcpacertrace=1 ，我们可以绘制pacer参数的精确演变：

![softhardheap_f4](/assets/post/go/softhardheap_f4.png "softhardheap_f4")

The thick black line shows the balance of heap growth and GC CPU at which the trigger error is 0. The crosses show the actual values of these two at the end of each GC cycle as the benchmark runs. During warmup, the pacer is still adjusting to the rapidly changing heap. However, once the heap enters steady state, GC reliably finishes at 50% of the target heap growth, which causes the pacer to dutifully stabilize on 50% GC CPU usage, rather than the desired 25%, just as predicted above.

粗黑线显示了触发误差为0时堆增长和GC CPU的平衡。交叉显示了基准运行时每个GC周期结束时这两者的实际值。在预热期间，pacer仍在适应快速变化的堆。然而，一旦堆进入稳定状态，GC就会可靠地以目标堆增长的50%完成，这会导致Pacer 尽职尽责地稳定在50%的GC CPU使用率，而不是如上面预测的所需的25%。

### Proposed solution

I propose separating the heap goal into a soft goal, $$ h_g $$, and a hard goal, $$ h_g' $$, and setting the assist pacing such the allocated heap size reaches the soft goal in expected steady-state (no live heap growth), but does not exceed the hard goal even in the worst case (the entire heap is reachable). The trigger controller would use the soft goal to compute the trigger error, so it would be stable in the steady state.

我建议将堆目标分为柔性目标$$ h_g $$和硬性目标$$ h_g' $$，并设置辅助步调，使分配的堆大小在预期稳定状态下达到的软目标（无实时堆增长），但不会即使在最坏的情况下（整个堆均可到达）也超过了硬性目标。触发控制器将使用柔性目标来计算触发误差，因此在稳定状态下会保持稳定。

Currently the work estimate used to compute the assist ratio is simply $$ W_e = s $$, where $$ s $$ is the bytes of scannable heap (that is, the total allocated heap size excluding no-scan tails of objects). This worst-case estimate is what leads to over-assisting and undershooting the heap goal in steady state.

目前，用于计算辅助率的工作估算只是$$ W_e = s $$，其中$$ s $$是可扫描堆的字节数（即，分配的总堆大小，不包括对象的非扫描尾部）。这种最坏情况的估算会导致稳定状态下过度协助和堆目标下冲。

Instead, between the trigger and the soft goal, I propose using an adjusted work estimate $$ W_e = s / (1+h_g) $$. In the steady state, this would cause GC to complete when the allocated heap was roughly the soft heap goal, which should cause the trigger controller to stabilize on 25% CPU usage.

相反，在触发点和柔性目标之间，我建议使用调整后的工作量估算公式$$ W_e = s / (1+h_g) $$。在稳定状态下，这将导致GC在分配的堆大致达到柔性堆目标时完成，这将导致触发器控制器稳定在25%的CPU使用率。

If allocation exceeds the soft goal, the pacer would switch to the worst-case work estimate $$ W_e = s $$ and aim for the hard goal with the new work estimate.

如果分配超过软目标，pacer将切换到最坏情况的工作估计$$ W_e = s $$，并以新的工作估算来实现硬性目标。

This leaves the question of how to set the soft and hard goals. I propose setting the soft goal the way we currently set the overall heap goal: $$ h_g = GOGC / 100 $$, and setting the hard goal to allow at most 5% extra heap growth: $$ h_g' = 1.05h_g $$. The consequence of this is that we would reach the GOGC-based goal in the steady state. In a heap growth state, this would allow heap allocation to overshoot the GOGC-based goal slightly, but this is acceptable (maybe even desirable) during heap growth. This also has the advantage of allowing GC to run less frequently by targeting the heap goal better, thus consuming less total CPU for GC. It will, however, generally increase heap sizes by more accurately targeting the intended meaning of GOGC.

这就留下了如何设定柔性目标和硬性目标的问题。我建议按照我们当前设置总体堆目标的方式设置柔性目标：$$ h_g = GOGC / 100 $$，并以允许最多5%的额外堆增长设置硬性目标：$$ h_g' = 1.05h_g $$。这样做的结果是我们将在稳定状态下达到基于GOGC的目标。在堆增长状态下，这将允许堆分配稍微超出基于GOGC的目标，但这在堆增长期间是可以接受的（甚至可能是理想的）。这样做的优点还在于，可以通过更好地瞄准堆目标来降低GC的运行频率，从而减少GC消耗的CPU总量。然而，它通常会通过更准确地瞄准 GOGC的预期含义来增加堆大小。

With this change, the pacer does a significantly better job of achieving its goal on the garbage benchmark:

通过这一更改，pacer在垃圾基准测试中明显更好地实现了其目标：

![softhardheap_f5](/assets/post/go/softhardheap_f5.png "softhardheap_f5")

### Evaluation

To evaluate this change, we use the go1 and x/benchmarks suites. All results are based on CL 59970 (PS2) and CL 59971 (PS3). Raw results from the go1 benchmarks can be viewed here and the x/benchmarks can be viewed here.

为了评估此次更改，我们使用go1和x/benchmarks套件。所有结果均基于CL 59970 (PS2)和CL 59971 (PS3)。可以在此处查看go1 benchmarks的原始结果，也可以在此处查看x/benchmarks。

### Throughput

The go1 benchmarks show little effect in throughput, with a geomean slowdown of 0.16% and little variance. The` x/benchmarks `likewise show relatively little slowdown, except for the garbage benchmark with a 64MB live heap, which slowed down by 4.27%. This slowdown is almost entirely explained by additional time spent in the write barrier, since the mark phase is now enabled longer. It's likely this can be mitigated by optimizing the write barrier.

go1 benchmarks对吞吐量影响不大，几何均值下降0.16%，方差很小。` x/benchmarks `同样显示出相对较小的放缓，但具有64MB活动堆的垃圾基准测试除外，该基准测试放缓了4.27%。这种减慢几乎完全可以用写屏障中花费的额外时间来解释，因为标记阶段现在启用的时间更长。这很可能可以通过优化写屏障来缓解。

### Alternatives and additional solutions

** Adjust error curve **. Rather than adjusting the heap goal and work estimate, an alternate approach would be to adjust the zero error curve to account for the expected steady-state heap growth. For example, the modified error term

** 调整误差曲线 **。另一种方法是调整零误差曲线以考虑预期的稳态堆增长，而不是调整堆目标和工作量估算。例如，修改后的误差项

![softhardheap_f6](/assets/post/go/softhardheap_f6.png "softhardheap_f6")

results in zero error when $$ h = u_g / (u_a(1+h_g)) $$, which crosses $$ u_a = u_g $$ at $$ h = 1 / (1+h_g) $$ , which is exactly the expected heap growth in steady state.

当$$ h = u_g / (u_a(1+h_g)) $$，且当$$ u_a = u_g $$，$$ h = 1 / (1+h_g) $$时，结果为零误差，这正是稳定状态下的预期堆增长。

This mirrors the adjusted heap goal approach, but does so by starting GC earlier rather than allowing it to finish later. This is a simpler change, but has some disadvantages. It will cause GC to run more frequently rather than less, so it will consume more total CPU. It also interacts poorly with large GOGC by causing GC to finish so early in the steady-state that it may largely defeat the purpose of large GOGC. Unlike with the proposed heap goal approach, there's no clear parallel to the hard heap goal to address the problem with large GOGC in the adjusted error curve approach.

这反映了调整后的堆目标方法，但通过更早开始GC而不是让它稍后完成来实现。这是一个更简单的更改，但有一些缺点。它会导致GC运行得更频繁而不是更少GC，因此会消耗更多的CPU总量。它还与大型GOGC相互作用很差，导致GC 在稳态下过早完成，从而可能在很大程度上违背大型GOGC的目的。与建议的堆目标方法不同，在调整误差曲线方法中，没有与硬性堆目标明确的并行来解决大GOGC的问题。

**Bound assist ratio**. Significant latency issues from assists may happen primarily when the assist ratio is high. High assist ratios create a large gap between the performance of allocation when assisting versus when not assisting. However, the assist ratio can be estimated as soon as the trigger and goal are set for the next GC cycle. We could set the trigger earlier if this results in an assist ratio high enough to have a significant impact on allocation performance.

**约束辅助率**。当辅助率较高时，辅助可能会出现严重的延迟问题。高辅助率会在辅助和不辅助时的分配有着巨大的性能差异。然而，一旦为下一个GC周期设置了触发器和目标，就可以估计辅助率。如果这导致辅助率足够高以对分配性能产生重大影响，我们可以更早地设置触发器。

Accounting for floating garbage. GOGC's effect is defined in terms of the "live heap size," but concurrent garbage collectors never truly know the live heap size because of floating garbage. A major source of floating garbage in Go is allocations that happen while GC is active, since all such allocations are retained by that cycle. These pre-marked allocations increase the runtime's estimate of the live heap size (in a way that's dependent on the trigger, no less), which in turn increases the GOGC-based goal, which leads to larger heaps.

漂浮垃圾的核算。GOGC的效果是根据“活动堆大小”来定义的，但由于浮动垃圾，并发垃圾收集器永远无法真正了解活动堆大小。Go 中浮动垃圾的一个主要来源是GC活动时发生的分配，因为所有此类分配都被该周期保留。这些预先标记的分配增加了运行时对活动堆大小的估算（在某种程度上取决于触发器），这反过来又增加了基于GOGC的目标，从而导致更大的堆。

We could account for this effect by using the fraction of the heap that is live as an estimate of how much of the pre-marked memory is actually live. This leads to the following estimate of the live heap:![softhardheap_f7](/assets/post/go/softhardheap_f7.png "softhardheap_f7"), where $$ m $$ is the bytes of marked heap and $$ H_T $$ and $$ H_a $$ are the absolute trigger and actual heap size at completion, respectively.

我们可以通过使用活动堆的比例来估计预标记内存的实际活动量，从而解释这种影响。这导致对活动堆的估计如下：![softhardheap_f7](/assets/post/go/softhardheap_f7.png "softhardheap_f7")，其中$$ m $$是标记堆的字节数，$$ H_T $$和$$ H_a $$分别是完成时的绝对触发器和实际堆大小。

This estimate is based on the known post-marked live heap (marked heap that was allocated before GC started), $$ m-(H_a-H_T) $$. From this we can estimate that the overall fraction of the heap that is live is $$ (m-(H_a-H_T))/H_T $$. This yields an estimate of how much of the pre-marked heap is live: $$ (H_a-H_T)*(m-(H_a-H_T))/H_T $$. The live heap estimate is then simply the sum of the post-marked live heap and the pre-marked live heap estimate.

此估计基于已知的标记后的活动堆（在GC开始之前分配的标记堆），$$ m-(H_a-H_T) $$。由此我们可以估计活跃堆的总体比例是$$ (m-(H_a-H_T))/H_T $$。这会产生对预先标记的堆中有多少是活动的估计：$$ (H_a-H_T)*(m-(H_a-H_T))/H_T $$。那么，活动堆估计只是后标记活动堆和预标记活动堆估计的总和。

**Use work credit as a signal**. Above, we suggested decreasing the background mark worker CPU to 20% in order to avoid saturating the trigger controller in the regime where there are no assists. Alternatively, we could use work credit as a signal in this regime. If GC terminates with a significant amount of remaining work credit, that means marking significantly outpaced allocation, and the next GC cycle can trigger later.

**使用工作信用作为信号**。上面，我们建议将后台标记工作CPU降低到20%，以避免在没有辅助的情况下触发控制器饱和。或者，我们可以使用工作信用作为该制度中的信号。如果GC终止时剩余大量工作信用，则意味着标记明显超过分配速度，并且下一个GC周期可以稍后触发。

TODO: Think more about this. How do we balance withdrawals versus the final balance? How does this relate to the heap completion size? What would the exact error formula be?

TODO：更多地考虑这一点。我们如何平衡提款与最终余额？这与堆完成大小有何关系？确切的误差公式是什么？

**Accounting for idle**. Currently, the trigger controller simply ignores idle worker CPU usage when computing the trigger error because changing the trigger won't directly affect idle CPU. However, idle time marking does affect the heap completion ratio, and because it contributes to the work credit, it also reduces assists. As a result, the trigger becomes dependent on idle marking anyway, which can lead to unstable trigger behavior: if the application has a period of high idle time, GC will repeatedly finish early and the trigger will be set very close to the goal. If the application then switches to having low idle time, GC will trigger too late and assists will be forced to make up for the work that idle marking was previously performing. Since idle time can be highly variable and unpredictable in real applications, this leads to bad GC behavior.

**核算闲置**。目前，触发器控制器在计算触发器错误时只是忽略空闲工作线程CPU使用情况，因为更改触发器不会直接影响空闲CPU。然而，空闲时间标记确实会影响堆完成率，并且由于它有助于工作信用，因此也会减少协助。因此，无论如何，触发器都会依赖于空闲标记，这可能会导致不稳定的触发器行为：如果应用程序有一段时间的空闲时间较长，GC将反复提前完成，并且触发器将设置为非常接近目标。如果应用程序随后切换到具有较低的空闲时间，GC将触发得太晚，并且辅助将被迫弥补空闲标记之前执行的工作。由于在实际应用中空闲时间可能变化很大且不可预测，因此这会导致不良的GC行为。

To address this, the trigger controller could account for idle utilization by scaling the heap completion ratio to estimate what it would have been without help from idle marking. This would be like assuming the next cycle won't have any idle time.

为了解决这个问题，触发控制器可以通过缩放堆完成率来估计空闲利用率，以估计在没有空闲标记帮助的情况下的情况。这就像假设下一个周期不会有任何空闲时间。