# Claude Fable5 设计AHB总线矩阵从 RTL 到签核

原创 芯球上面 芯球上面

 _2026年6月14日 16:13_

想必大家都知道了，Claude Fable5 只上线了4天，就被「拔了网线」，Anthropic 公告说法是国家安全方面的考虑，AI 早就不是简单的技术力量。

我是在它上线当晚设计了这个电路，设计实现花了30分钟左右，然后用/goal 指令让它继续跑验证，大概花了40分钟。接着让它从 rtl 反吐出spec，最终 usage干到17%，输出 5个RTL 模块，1032行， testbench 24个文件，3933行，21页spec，spec里面的图我是用$image 画的。

也就刚到周末，Fable5就停服了，不得已只能退回Opus4.8 用DC跑综合，pdk使用开源的 sky130 三角 MCMM 综合签核——从零到网表，我基本没碰过 RTL。设计是 Claude Fable 5 写的，验证环境是 Fable 5 搭的，Spec是Fable 5 生成的，综合签核是 Claude Opus 4.8 跑通的。 最终结果：167MHz 三角全 MET，8753 µm²，0.741 mW，7 组回归测试 UVM_ERROR 全零。

---

## 1我只输入了这一句话

我给 Fable 5 的提示词是这样的：

![图片](https://mmbiz.qpic.cn/mmbiz_png/tibCF9pXWVtAnZ1d0VcZRLUxRz5aic2sIkPGvdeL6IPwSf5q8vra1BldI1hA3qwgjjwIDgKoa5cvLPicPUMUHvg0VDqQFDq6HRJicY3VsXHXj6g/640?wx_fmt=png&from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

就这么一段话，没有 spec 文档，没有微架构图，没有寄存器列表。然后 Fable 5 给我吐出来了五个文件：`ahb_matrix.sv`、`ahb_mtx_ip.sv`、`ahb_mtx_dec.sv`、`ahb_mtx_op.sv`、`ahb_mtx_arb.sv`。我怀疑是它是学习或训练了CMSDK的总线矩阵结构，不然不会给出来这么规整的文件规划以及功能划分，总线矩阵的顶层互连、输入级、地址译码、输出级、仲裁器，一个不少。语料主要是学习ARM提供的外部spec，因为在我让它生成spec的时候，它很自然的说采用ARM的蓝色主题大厂风格。

![图片](https://mmbiz.qpic.cn/mmbiz_png/tibCF9pXWVtArl2ibHYlkUdSibSibBQrUzu6hP0LOJR0yCNv9cJydTgibP3gqNrMqOViaic9q9iaEGE2rRx41mJNSCjDzxPPYk3NgH694dhC4qFjg7g/640?wx_fmt=png&from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=1)

![图片](https://mmbiz.qpic.cn/mmbiz_png/tibCF9pXWVtDT5ontSDiarUe2Lt7wkwkXvU1juKZZQibAP11TNRMQSmWQ4Hzwmaq6vvknzgiaJazjmrb8wmibibcHbWaDzYbXsyloNNb6fuZau1hY/640?wx_fmt=png&from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=2)

我当时的第一反应是：先别高兴，看看它有没有偷懒——把所有逻辑塞进一个 `always` 块，或者时序逻辑里混了 latch。

结果看下来，没有。代码是干净的同步逻辑，异步复位，模块划分也和 CMSDK 风格对得上。更让我在意的是：它不只是写出来了，它能解释为什么这样写——这个后面再说。

先讲这颗总线矩阵是个什么东西。

---

## 2这颗总线矩阵解决什么问题

做过 SoC 集成的工程师应该都遇到过这个场景：片上有多个主设备（CPU、DMA、DSP），要访问多个从设备（SRAM、外设、ROM），主从之间不能一对一连，那样布线会炸，而且效率极低。

总线矩阵（bus matrix）就是解决这个问题的基础设施。它让任意主设备都能访问地址映射允许的任意从设备，而且**不同主访问不同从的时候可以真正并行跑**，不用排队。

这个工程实例化的是 **2 主 × 2 从**，但它是通用总线结构，也就是通过参数例化最多可生成16*16的多mater多slave的结构，32-bit 地址和数据，AHB-Lite 协议。

AHB vs AHB-Lite 核心差异： AHB-Lite = AHB 去掉多主支持，无仲裁器，无 HBUSREQ/HGRANT 信号 HRESP 从 2bit 缩减为 1bit（去掉 SPLIT/RETRY，只保留 OKAY/ERROR） 需要多主时改用 multi-layer AHB-Lite + interconnect 实现 地址相位流水、burst 类型、HTRANS/HSIZE 等信号语义两者完全一致。  

  

从验证角度看,AHB-Lite 的 protocol checker/assertion 明显更小——不用写 SPLIT 重发顺序、grant 切换时的 IDLE 插入这类断言。实际上现在的新设计(Cortex-M 系列外设总线等)几乎都用 AHB-Lite 或 AHB5, 经典多主 AHB 已经很少见了。  

  

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

微架构沿用的是 CMSDK（ARM Cortex-M System Design Kit）的三段式风格：

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)`   `

每个主端口进来先过**输入级**，负责把地址相位收下来并做译码；然后发 request 给对应的**输出级**，输出级负责多主竞争仲裁和信号 mux；从设备侧的 `HREADYOUT` 和 `HRDATA` 再往回走，经输出级回到对应的输入级，最终返回主设备。说起来简单，但要做对，细节还是有考究的——光是 holding register 怎么设计这一个点，展开就够讲半篇文章。  

这套分法是 ARM 在设计 CMSDK 里的 cmsdk_ahb_busmatrix 时用的结构，后来在业界被广泛参考，很多自研总线矩阵都沿用了这个模块划分方式。

---

## 3设计里三个值得讲的判断

### 3.1 输入级：先收下，自己保管

AHB 的流水特性决定了一件事：主设备在当前数据相位还没结束的时候，就已经把下一笔事务的地址打出来了。总线矩阵必须在那一拍把地址收下来——不然主端就乱了。

但收下来之后，目标从机不一定立刻空闲。这时候怎么办？

Fable 5 实现的方案是一个标准的 decoupling buffer：输入级 FSM 走 `IDLE → DATA/HELD/ERR1 → ...`，地址相位 `HREADYOUTS=1` 的那一拍**一定把地址锁进 holding register**。如果目标输出级同拍就给了授权而且从机就绪，那事务**组合直通**，零延迟，FSM 进 `DATA`；否则 FSM 进 `HELD`，对源端拉低 `HREADYOUTS`，把主设备的下一拍地址相位 stall 住，同时把 holding register 里的地址作为 NONSEQ 反复 replay 给输出级，直到被接受为止。

这个设计的代价很直接：**holding register 最后在综合报告里占了 62% 的面积**。这不是浪费，这是面积和吞吐之间真实的trade-off ——因为你必须在 `IDLE` 状态就对主端拉高 `HREADYOUTS` 去收地址，让源端流水继续，所以这一拍的地址必须有地方存。

做中后端看一个设计是不是"干净"，有个很快的判断方法：看 FF 都长在哪。这颗矩阵 116 个 FF，86 个在输入级的 holding register，剩下 30 个散在输出级——写数据通路零 FF。这个分布一看就想清楚了Fable 5的设计思路，它还是相当克制：该存的存，不该存的一个不多。时序也好收，综合出来 167MHz 慢角 WNS 还剩 5ps，说明没有冗余路径在拖后腿。

### 3.2 写数据为什么不进寄存器

写数据通路的设计是Fable 5 做对的另一个判断。

AHB 有个特性：主设备在写事务的数据相位里，会在**整个数据相位**（包括被 wait state 延长的每一拍）持续保持 `HWDATA` 有效。所以输出级的 `HWDATAM` 可以直接 mux 数据相位 owner 的 live `HWDATA`，不需要把写数据锁进寄存器。

我自己 review 这段代码的时候，带着黑人❓️专门盯着写数据通路看了一遍，可以这么说对于总线协议理解不到位的工程师，倾向于把所有信号都锁一拍，感觉更安全。做过后端的人都知道，前端多一个不必要的寄存器，后端不只是多一个 FF 的事——关键路径上多一级寄存器，CTS 的压力、hold 修复的难度都会变化。

Fable 5 实现的时候确实没给写数据加寄存器——输出级只寄存了 `downer_q`（数据相位 owner 编号）和 `dvld_q` 这两个 bit。省下来的 FF 直接反映在综合报告里：输出级面积只占了约 37%，而输入级（有 holding register）占了 62%。

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

如果写数据也进寄存器，面积比例会完全不同，而且在某些时序下可能引入不必要的延迟。这是一个需要理解 AHB 协议时序才能做出的判断，说明Fable 5 对时序理解得非常深刻。

### 3.3 顶层的一次转置

`ahb_matrix.sv` 里有一处比较隐蔽的设计，我觉得值得单独提一下。

输入级（每个主端口一个）发出的 request 信号，索引方式是 `req[master * NS + slave]`——按主设备编号排列。但每个输出级（每个从端口一个）需要的是"哪些主设备在请求访问我"，也就是按从设备编号排列的视图。

顶层用一个 `genvar` 循环把这个索引做了转置：`req_s[slave * NM + master] = req_m[master * NS + slave]`。一次转置，把 M×N 的互连网络理顺成每个从端口独立的仲裁问题，每个 `ahb_mtx_arb` 只需要关心自己这一列的请求。

代码量不多，但逻辑很清晰。这种"把问题分解成正交子问题"的思路，我觉得是好设计的标志。

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

---

## 4验证：四路 scoreboard

设计写完，验证环境 Fable 5 也一起搭了。

环境基于 `ahb_vip` UVM VIP，这里需要说明一下这个ahb_vip，也是一个开源项目，我之前已经调通master与slave的loopback。这里让Fable 5基于这个VIP 来做，它使用了参数化的方式搭建了 2 个 active master agent + 2 个 active slave agent（从机带随机 wait state）。这里使用了/goal 不达目的不眠不休来做，让它基于我一开始提的4个 ahb 特性进行验证，保证特性功能正确，后面它就一直跑，我早上起床后，看到已经全部验完了。跑了大概30分钟。

### 4.1 Scoreboard 的四路检查

验证环境里我最在意的是 scoreboard 的设计，因为除了AHB VIP，测试用例设计，它把所有的reference都在SB里实现。Fable 5 搭出来的 scoreboard 同时跑四类检查：

**第一路，写意图 vs 读回。** sequence 在写数据的时候就把预期值记录下来，读回来之后对照检查。这是最基础的数据完整性验证。

**第二路，每从在序路由检查。** 每一笔事务的 master→slave 路由、HMASTER 编号、地址、读写方向、传输宽度、HPROT、逐 beat 数据，用成对的 exp/got 队列按序排空。任何丢拍、重复、乱序，都会在这里报出来。这一路是检查总线矩阵路由逻辑的核心。

**第三路，从机侧字粒度参考存储。** 以字地址为 key 的 reference memory，支持 sub-word 写合并。byte write 和 half-word write 都能正确更新 reference，读回时做字粒度对比。

**第四路，unmapped → ERROR 检查。** 每一笔越界访问（地址落在 `0x2000_0000` 以上的空间里），scoreboard 必须收到 ERROR 响应，也就是做到类似于越界报错的效果。而且**零从机流量**——也就是说这笔事务不能到达任何从设备。

另外还有一个可选 knob：round-robin 公平性检查，验证每个主设备的仲裁授权份额 ≥30%，且不出现连续超过 2 拍的授权独占。

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

这个本质是一个对序列做单趟遍历的统计判定(streaming/one-pass statistical check),可以拆成两个经典子问题: 

它由两个标准算法构成：

1. 最长连续游程统计(longest run / max consecutive run) 

```
 if (rr_hist[s][i] == last) run++;  else begin last = ...; run = 1;  end if (run > max_run) max_run = run; 
```

这就是经典里的 "最长连续相同元素子段" 问题 —— 维护一个当前游程长度 run,元素相同就累加、不同就重置, 顺手记录历史最大值 max_run。在这里它回答的是:"有没有哪个 master 被连续授权太多次(饿死对方)"。 

2. 频次/份额统计(frequency counting → proportion test)

```
 if (... == 0) cnt0++; else cnt1++; ...  if (cnt0 < (cnt0+cnt1)*3/10 || ...) // 份额 < 30% 
```

就是计数 + 比例阈值判定,等价于一个非常轻量的"分布是否均衡"检验(可以看成卡方/比例检验的极简版,只用一个固定下界 30%)。回答的是:"长期看授权是不是大致均分"。

- 优点:黑盒、与 RTL 仲裁实现解耦;统计判据(连发上限 + 份额)对实现细节不敏感,只钉住"公平"这个规格意图。 

- 盲区:它是事后统计而非逐拍协议检查,不会精确验证"严格轮转顺序"(比如 RTL 用的是 LRU 还是真正 round-robin,只要两者都满足 ≤2 连发且 ≥30% 份额就都算过)。要更强的检查得加 SVA 逐拍比对授权指针。 

- 阈值是工程容差:≤2 连发和30%都是为吸收"对方偶发不发请求"的边界抖动而留的余量,不是 round-robin 的数学定义本身。

  

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

### 4.2 七个测试，随机四个种子，全绿

|测试名|覆盖特性|
|---|---|
|`ahb_mtx_test`|数据完整性，写/读回，随机 wait state|
|`ahb_mtx_b2b_test`|背靠背事务对，每笔任意 AHB 类型|
|`ahb_mtx_same_slave_test`|两主饱和争用同一从|
|`ahb_mtx_rr_test`|饱和下的 round-robin 算法|
|`ahb_mtx_parallel_test`|两主同时访问不同从（吞吐测试）|
|`ahb_mtx_err_test`|默认从机 ERROR + 恢复流量|
|`ahb_mtx_burst_test`|全 burst 类型 + BYTE/HALF/WORD + BUSY 插入|

  

---

## 5综合签核：167MHz 三角全 MET

### 5.1 工具和工艺

- **工具**：Synopsys Design Compiler `compile_ultra -no_autoungroup`
    
- **工艺库**：SkyWater `sky130_fd_sc_hd` 标准单元
    
- **签核角**：三角 MCMM——`tt_025C_1v80`（典型）、`ss_100C_1v60`（慢角，签 setup）、`ff_n40C_1v95`（快角，签 hold）
    
- `.lib` → `.db` 用 Library Compiler 转换，`set_min_library` 做多角配置
    

地址映射参数通过 `elaborate -parameters` 显式传入（`SLV_MASK=0xF000_0000`），保证默认从机和 ERROR 路径不被 DC 优化掉。

综合分三步跑：

```
dc_shell -f scripts/run_syn.tcl          # tt 基线 @100MHz（PPA 基准 + latch 检查）dc_shell -f scripts/run_syn_mcmm.tcl     # 多角：setup@ss / hold@ffdc_shell -f scripts/run_signoff_167.tcl  # ★ 167MHz 签核网表 + SDC
```

### 5.2 签核结果

HCLK = 6.0 ns，IO budget 20%（= 1.2 ns，已记入 SDC），setup 在慢角签，hold 在快角签：

|指标|值|
|---|---|
|**setup WNS @ ss_100C_1v60**|**+0.005 ns（MET）**|
|**hold WNS @ ff_n40C_1v95**|**+0.021 ns（MET，hold 已修）**|
|约束违例|**无**|
|面积|**8753 µm²**|
|单元数|**1273 leaf**<br><br>（FF 116 / 组合 1157，**无 latch**）|
|功耗 @167MHz|**0.741 mW**<br><br>（内部 0.673 + 翻转 0.062 + 漏电 ~5.9 µW）|

三角全 MET，约束零违例，无 latch。

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

### 5.3 面积都花在哪了

这里有一个数字我觉得很值得展开：输入级占了总面积的约 62%。

这 62% 主要是 holding register——约 86 个 FF，在总共 116 个 FF 里占了大半。**写数据通路一个 FF 都没有**，输出级只用了 2 个 bit 的状态寄存器（`downer_q` + `dvld_q`）。

这个数字正好印证了 3.1 节说的设计判断：holding register 是 spec 3.2.2 要求的解耦缓冲，它的面积代价是确定性的、不可避免的，而写数据不进寄存器是一个主动的优化决策，省出来的面积是真实的。

两个角度来看综合报告，得到的结论是一致的——这正是我觉得"AI 能解释为什么这样设计"这件事有价值的原因：**可解释的设计判断，可以在综合数据里得到验证**。就好比说Fable 5 的行为类似于一个资深设计师，它理解思绪，懂的怎么写好可综合 verilog，并在完成设计功能的前提下，在可靠的PPA边缘进行试探以找到合适的折中路径。

|模块|面积占比|说明|
|---|---|---|
|输入级 `g_ip[0]+[1]`|~62%|addr+ctrl holding register（≈86 FF，**不含 wdata**）|
|输出级 `g_op[0]+[1]`|~37%|地址/写数据组合 mux + 仲裁器（仅 2 bit FF）|
|译码器 `u_dec`|~8%|解 `HADDR[31:28]`（256MB 粒度）|

### 5.4 fmax 扫频

对签核网表逐角扫频，**签核 fmax ≈ 200 MHz（5 ns，ss 角限**），167 MHz 有舒适裕量：

|周期|频率|setup@ss|hold@ff|状态|
|---|---|---|---|---|
|6.0 ns|167 MHz|+0.005 ns|+0.021 ns|**MET（签核点）**|
|5.0 ns|200 MHz|−0.001 ns|−0.008 ns|临界墙|
|4.5 ns|222 MHz|−0.32 ns|−0.005 ns|VIOLATED|

后续交给后端：PnR + CTS 之后用真实时钟树重查 hold，按真实 IO 路径设 budget，有条件的话再跑一轮 PrimeTime 多角多模 STA。

最终Sign off 产物：

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

  

![图片](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

---

## 6一张全景总表

|阶段|工具 / 框架|完成者|关键结果|
|---|---|---|---|
|RTL 设计|SystemVerilog（AHB-Lite 2×2 matrix）|**Claude Fable 5**|5 模块，CMSDK 三段式，零 latch，可综合|
|UVM 验证|Synopsys VCS + UVM-1.2（ahb VIP）|**Claude Fable 5**|7 test × 4 seed 全 PASS，UVM_ERROR 0|
|DC 综合签核|Synopsys DC + sky130 三角 MCMM|**Claude Opus 4.8**|167 MHz 三角全 MET，8753 µm²，1273 cell，0.741 mW，fmax ≈ 200 MHz|

---

## 7说实话，这件事让我想了挺久

Fable 5设计的这个模块你感用在实际项目并 Sign off 吗？

答案是pass，这个模块的通路不管是仿真还是波形，综合报告都check过，没有问题，但不排除在某些功能corner场景没考虑到。在快速开发节点提拉下，我们可以把ip放在soc 验证系统场景功能是否正确，尤其是CPU 可能发出非标时序，多master并发的压力测试场景，但在主体功能我认为是没问题，如果非要走完备性验证，我觉得下一步更应该是Formal，这个地方打通了，我觉得验证的底座基本就成了。 

还有一个我觉得更值得记录的，不是"它做完了"这个结论，而是**它在哪些地方做出了真正的工程判断**：

holding register 占 62% 面积，这不是随机综合出来的数字，背后是对 AHB 解耦缓冲机制的理解；写数据不进寄存器，这是利用 AHB 协议时序特性的主动决策。

这些判断，如果是一个刚入行两年的工程师来做，我不确定他能做对。

说这些不是为了制造恐慌——我自己也是做这行的，我知道芯片工程里有大量 AI 现在还做不好的事：复杂 SoC 集成、跨团队接口谈判、从一份需求文档里挖出真正的设计约束。这些事不会在短期内消失。

但**从一句话提示词到可交付的综合签核网表**，这件事已经可以发生了。

喜欢作者

AI设计师 · 目录

上一篇AI设计SPI综合实战：当你的RTL遇上开源"前辈"

阅读 3693

​

[](javacript:;)

![](https://mmbiz.qpic.cn/mmbiz_png/Opay6Ye9p2OO1a9Focm9YWlAmZjZvOaibmAWcK4qvVuQHZhauwgGXMVhNezBSsuiaFZO7XIaaXBACTicqgZjPlpJQ/300?wx_fmt=png&wxfrom=18)

芯球上面

42

283

22

13

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

暂无评论