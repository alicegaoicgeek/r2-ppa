# AI 做 FPGA 时序优化的 Aha 时刻

原创 JieL AI驱动FPGA

 _2026年5月18日 15:26_

![](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

324 → 360 MHz

通用时序优化工具的一次验证，11% Fmax 提升，零额外流水线代价

用 AI 做成一次 FPGA 时序优化之后，我们想把这种成功复制下来。最直接的办法是把那次的经验做成工具：让 AI 自己去查 Vivado 里所有能调时序的手段，自己把它们编排到一条优化流程里，再自己去读时序报告判断当前应该用哪一招。我们想要的是一个通用工具，不挑设计，也不挑目标频率。

这一篇讲的是这个工具的第一次实战，以及它意外给出的一个 Aha 时刻。

## 1工具是怎么搭起来的

我们让 AI 把 Vivado 的官方文档和过往项目经验都翻一遍，把所有能影响时序的选项收集起来：布局指令、物理优化指令、扇出与 DSP 映射属性、SRL 推断、报告读取流程、读-改-写流水线切分模式，等等。然后让 AI 把这些选项编排到一条优化流程里：

- ▶分析瓶颈
    
     把当前的关键路径按已知模式分类
    
- ▶选定修复策略
    
     从工具选项目录里挑出一组候选
    
- ▶执行并对比
    
     跑一次综合加布线，再读时序报告
    
- ▶决定下一步
    
     选下一类选项，或者回滚到上一个最优点
    

![工具的内部结构，流程加目录加边界](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

工具的内部结构，流程加目录加边界

整个流程考虑的所有候选方案都是基于 Vivado 工具层的招数。代码层的优化，比如重写 RTL 让综合工具更容易识别等价关系，并没有事先考虑进去。

这个工具层边界后面被 AI 自己捅破了。

## 2拿一个真实的工程问题验证

第一个测试场景，是高频交易系统里的订单簿存储管理模块。把这个模块的标的数量调大之后，布线后的最高频率是 324.4 MHz，我们想要的目标是 360 MHz。差 36 MHz，11% 的频率提升，这种幅度不是改一两个参数就能拿到的。

工具的输入很明确：一份跑在 324.4 MHz 的设计，加一个 360 MHz 的目标。输出应该是一份能闭环的修改方案。剩下的让工具自己走。

## 3第一刀 · 工具自己就能走到 333 MHz

工具上手第一件事是把关键路径分类。订单簿模块的关键路径，是一个非常典型的"读-改-写"环路：从 BRAM 里读出当前的累计数量，加一个有符号的变化量，做饱和处理，再写回同一块 BRAM。BRAM 的读出延迟差不多 1 纳秒，写入建立时间又要 0.3 纳秒，两个加起来已经占了 333 MHz 一个周期的一半。中间挤下一个 24 比特的加法和 3 路饱和选择，时钟周期就爆了。

![一个时钟周期的命运](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一个时钟周期的命运

这是 FPGA 时序里经典的一个瓶颈，工具的目录里早就有对应的修复：在读-改-写环路里插一级寄存器，把这条 BRAM 加法器 BRAM 的组合链路切成两段。代价是这个模块上多一个时钟周期的延迟。考虑到整条流水线的端到端预算还没用满，多 1 个周期是赚的，值得换更高的工作频率。

![改前与改后，切成两段](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

改前与改后，切成两段

工具按这个套路把代码改完，时序模型同步加一拍，下一次综合下来 333 MHz 干干净净地收敛了，留了几皮秒余量。这一步基本是机械动作，工具自己就能完成。到这里，工具完全在我们预想的剧本里。

但我们要的是 360 MHz，不是 333 MHz。

## 4撞墙 · 工具层都走完了，卡在 348 MHz

把目标提到 360 MHz 之后，工具进入它最熟悉的模式：在工具选项目录里轮转，每改一个跑一次综合，几个小时下来 AI 把每一类选项都试了一遍。

跑下来有个出乎意料的观察。原本默认的那个布局策略，在更紧的时钟约束下反而排到了中段；而原本评分最低的一个策略，在 360 MHz 目标下成了最好的。同一个设计在不同的时钟周期下，布局策略的排名会大幅变化，因为时钟越紧，工具越倾向于走最短连线的版本。这个观察被工具记进了知识库：碰到新目标频率，要重新排查布局策略，不能照搬上一次的结论。

这一轮里 AI 也撞了两次墙。

![两次失败的尝试，记下来不丢掉](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

两次失败的尝试，记下来不丢掉

把扇出高的信号加手动复制属性，让综合工具多生成几个副本服务不同的下游模块。局部看更快了一点，全局重跑反而慢了 166 皮秒，多出来的副本挤散了相邻模块原本紧凑的布局。把一段加法显式映射到 DSP 专用单元，指望专用硬件更快，结果 DSP 离上游数据源太远，连线绕一大圈，反而慢了 675 皮秒。

这两次失败被以注释的形式写回了源代码，注明"什么条件下试过、为什么没用"。下次再有人或 AI 走到这条路径，不会再重复犯错。

把工具选项目录里所有的招数都打完，我们停在了 348 MHz，离目标还差 12 MHz。按工具最初的设计，到这里应该写一份"工具层已经穷尽"的报告，把决定权交回给人。

> 348 MHz 看起来就是不增加流水线延迟的天花板了。

![工具的内置手段都在左边，AI 跳到了右边那一支](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

工具的内置手段都在左边，AI 跳到了右边那一支

## 5Aha 时刻 · AI 自己做的两个决定

工具的 Vivado 选项目录走完了，预期下一步是写报告交差。但 AI 在写报告之前做了两件我们没安排的事：它回头重读了 SystemVerilog 源代码，又重新翻开了布线工具的迭代日志。

这两个动作的共同点是，**我们没把它们放进工具的目录里**。第一个动作跨过了工具层和代码层的边界。第二个动作把我们硬编码进工具的一个参数上限自己拉高了。这一节讲这两个决定怎么发生的，以及为什么有意思。

**▍ 决定一·跳进 SystemVerilog 源代码**

AI 翻出了订单簿包装层里这样三行代码：

  

> assign rmw_dec_w         = (align_dec_qty_w > oh_qty_stored_w) ? oh_qty_stored_w : align_dec_qty_w;  
> assign rmw_remaining_w   = oh_qty_stored_w - rmw_dec_w;  
> assign rmw_full_remove_w = ... & (rmw_remaining_w == '0);

  

三行依次做三件事：第一行裁剪扣减量（如果想扣的比存留的多，就只扣存留那么多），第二行算扣完之后还剩多少，第三行判断这一笔订单是不是被全部消除了。三步串在一拍里，每一步在硬件上对应一条独立的进位链。三条进位链堆在一起，正好是这一拍关键路径上最长的那一段。

AI 盯着第三行看了一会儿，提了个问题。第三行在判断"剩余等于零"，可剩余的定义来自第二行，第二行的输入又来自第一行的裁剪结果。如果把这条依赖链展开看，第三行会不会本来就等于第一行那个比较器的输出？

这个问题可以用两种方式验证，直接枚举两种情况，或者沿着代数恒等式一步一步推。

第一种验证最直观，把"扣减"和"存留"的两种可能组合摆出来。

|情况|扣减 vs 存留|裁剪扣减 = min(扣减，存留)|剩余 = 存留 − 裁剪扣减|剩余 == 0|扣减 ≥ 存留|
|---|---|---|---|---|---|
|A|扣减 ≥ 存留|存留（裁剪触发）|存留 − 存留 = 0|true|true|
|B|扣减 < 存留|扣减（裁剪未触发）|存留 − 扣减 > 0|false|false|

两种情况，最后两列完全一致。也就是说 `(剩余 == 0) ≡ (扣减 ≥ 存留)`。第三行那个比较，第一行已经算过一次了。

第二种验证是代数推导。从 `剩余 == 0` 出发，把每个临时变量用它的定义替换掉，五步落到一个已知的比较器上。

![从](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从 "剩余 == 0" 一路推到 "扣减 ≥ 存留"，五步代数恒等式

两种验证给出同一个结论：第三行的输出等于第一行的比较器输出，直接接线即可，不用再走"减法加判零"那两条进位链。剩下要做的只是把代码改一下，让综合工具看见这条复用关系。

  

> assign dec_ge_stored_w   = (align_dec_qty_w >= oh_qty_stored_w);  
> assign rmw_dec_w         = dec_ge_stored_w ? oh_qty_stored_w : align_dec_qty_w;  
> assign rmw_full_remove_w = ... & dec_ge_stored_w;

  

改完之后，三条进位链塌缩成一条，关键路径短了 96 皮秒，资源用量还少了 38 个查找表。

![共享比较器之后，三条进位链塌缩成一条](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

共享比较器之后，三条进位链塌缩成一条

**▍ 综合工具为什么没自己做这件事**

Vivado 综合器里有公共子表达式消除（CSE）这套机制，但它的工作方式是匹配抽象语法树里句法相同的表达式。`(剩余  0)` 和 `(扣减 ≥ 存留)` 长得完全不一样，一个是 `(X − Y)  0` 的形态，另一个是 `X ≥ Y` 的形态，中间还隔着一个 MUX（裁剪的条件选择）。工具没办法跨过这个 MUX 做语义级的等价证明。

![同一段代码，工具和 AI 看见的不是同一件事](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

同一段代码，工具和 AI 看见的不是同一件事

这件事在学术界有专门的研究方向。RTLRewriter（Yao 等，ICCAD 2024）明确指出商用综合工具"在提取语义信息和优化方面能力有限"。ROVER（Coward 等，IEEE TCAD 2024）提出用 e-graph 等价类重写来自动发现这类化简。它依托的引擎 egg（Willsey 等，POPL 2021）是这一类工作的事实标准。更早的 Hosangadi-Fallah-Kastner（ICCAD 2004）是这条思路的源头，他们就指出，标准 CSE 只能处理已暴露的公共子表达式，隐藏在临时变量后面的需要先做代数变形才能暴露。

换句话说，AI 这次看见的等价关系，整个学术界都在花力气教自动化工具去看。AI 用的不是新工具，是源代码加上代数直觉。

这种"裁剪后再判零"的模式在很多地方都会出现：cuckoo 哈希的踢出 FSM、位置表的更新、网络流表、几乎任何带饱和减法加条件移除的读改写。这个等价改写的套路可以横扫整个代码库找类似的模式，每命中一次都是几百皮秒外加一把查找表。

**▍ 决定二·把布线工具的迭代上限自己拉上去**

代码改完之后，时序报告里的规范配置给出 WNS −0.004 ns，差 4 皮秒就闭环了。这个余量小到默认设置就有概率被波动吃掉。

工具最初的设置里，物理优化收敛的迭代上限是 5 轮。这是 Vivado 文档里给的默认值，我们也照搬了。这一轮里 AI 读完迭代日志之后做了一个我们没安排的决定：把上限调到 10。

它的依据是迭代日志本身。第 1 到第 4 轮的 WNS 在 −0.022、−0.030、−0.022、−0.063 之间来回晃，第 5 轮一下子跌到 −0.161 ns，是这一段里最差的一个状态。如果就按默认设置在第 5 轮停下，工具会触发回滚，前面几轮的进展全部作废。但 AI 看完日志的判断是，这条曲线在游荡，不是在收敛，再多跑几轮不一定更差。

![默认 5 轮停在最糟糕的中间态，AI 把上限调到 10，第 8 轮闭环](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

默认 5 轮停在最糟糕的中间态，AI 把上限调到 10，第 8 轮闭环

把上限改到 10 之后，第 6、第 7 轮还在乱晃，第 8 轮一下子跳到 WNS +0.001 ns，0 个失败路径，360 MHz 干净地闭环。整条流水线的延迟和 333 MHz 那一版完全一样，没有再多花一个时钟周期。

360 MHz 是这样收敛的。

这件事和决定一是同一类。我们把 5 轮硬编码进工具，是因为 Vivado 的默认值是 5，我们没多想。AI 在没人告诉它"这个数字可以改"的情况下，自己把它改了。这不是跨抽象层的越界，而是跨参数上限的越界，但本质上做的是同一件事：在工具最初的边界之外再往前走一步。

  

## 6工具到位之后，AI 会自己越界

  

![订单簿模块时序收敛之路 · 工具走到 348 MHz，AI 自己又往前走了一段](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

订单簿模块时序收敛之路 · 工具走到 348 MHz，AI 自己又往前走了一段

这一次实战里，AI 做了两件我们没明确放进工具目录的事。

- ▶跨过抽象层
    
     目录里只有 Vivado 选项，AI 自己加上了"读 RTL 找等价改写"这一步
    
- ▶跨过参数上限
    
     目录里写死了"最多 5 轮"，AI 自己改成了 10
    

这两件事我们都没明确告诉它要做。但单独看每一件，它做的事都是合理且可解释的：第一件靠代数恒等式可以验证，第二件靠迭代日志的走势可以判断。AI 不是凭空蹦出新招数，而是在它已经具备的能力之内，把搜索空间向外延伸了一点点。

两次越界把"工具的边界 = AI 能力的边界"这个假设打掉了。我们做工具的时候默认 AI 会守在我们给的目录里，结果证明：当目录里的招数走完、AI 又判断目标还没达成时，它会自己再往前找一找。跨层、跨参数上限，都可能发生。

这两个发现已经被回写到工具里。共享比较器复用这一类等价改写的模式作为新的代码层优化机会，被加入了瓶颈分类目录；物理优化迭代上限的判定，作为新的一条诊断规则进了报告分析流程，当规范配置接近闭环、迭代曲线非单调时，工具会自动把上限拉到 10。下一个项目遇到同类情况，不需要让 AI 重新发现。

> 工具在给 AI 搭脚手架，AI 在工具上发现新的招数，新的招数回到工具里成为下次脚手架的一部分。发现一次，下次免费。

AI 不替工程师做判断，但脚手架到位之后，它会去工程师都没仔细看过的地方自己试一试。我们要做的，是把这种发现能力变成可复制的。

---

**延伸阅读**

- ▶RTLRewriter
    
    （Yao 等，ICCAD 2024，arXiv:2409.11414）LLM 辅助 RTL 重写，与 Yosys 和 e-graph 基线做了显式对比，确认商用综合工具在语义级化简上的能力上限。
    
- ▶ROVER
    
    （Coward、Drane、Constantinides，IEEE TCAD 2024，arXiv:2406.12421）用 e-graph 等价饱和做 RTL 数据通路重写，附带商用等价性检查器的形式化证明。
    
- ▶egg
    
    （Willsey 等，POPL 2021，arXiv:2004.03082）现代 e-graph 工程的事实标准引擎，最近一批 RTL 等价饱和工作多数构建在它之上。
    
- ▶Hosangadi-Fallah-Kastner
    
    （ICCAD 2004，"Reducing hardware complexity of linear DSP systems by iteratively eliminating common subexpressions"）算法因子化与代数 CSE 的源头工作，最早指出隐藏在临时变量后面的公共子表达式必须先做代数变形才能被消除。
    

阅读原文

阅读