# caseall 目录下"跑通 PPA 优化"案例梳理

> 搜索范围：`~/WorkBuddy/caseall`（含 symlink 到 Obsidian 笔记库的案例文件）
> 搜索日期：2026-07-14

---

## 一、已跑通 PPA 优化的案例

### 1. 695/894 — Claude Fable5 设计 AHB 总线矩阵从 RTL 到签核

**最直接的"跑通 PPA"案例。** AI（Claude Fable5 + Opus 4.8）完成了从 Spec → RTL → 验证环境 → DC 综合签核的全链路：

- **PPA 结果：** 167MHz 三角全 MET，8753 µm²，0.741 mW
- **关键路径：** `run_syn.tcl`（tt 基线 @100MHz，PPA 基准 + latch 检查）→ `run_syn_mcmm.tcl`（多角 setup@ss / hold@ff）→ `run_signoff_167.tcl`（167MHz 签核网表 + SDC）
- **原文明确说"跑通"：** "综合签核是 Claude Opus 4.8 跑通的"
- **PPA 折中：** "在可靠的 PPA 边缘进行试探以找到合适的折中路径"

### 2. 99 — AutoResearch × FPGA 时序优化

FPGA 时序优化（PPA 的 Performance 维度）：

- AutoResearch 工具 + PPA-Skill 做频率/WNS 优化
- RootCause-Agent 分析瓶颈 → FixSuggest-Agent 给方案 → PPA-Skill 画图 + 审计
- 把"试错过程"变成"知识资产"

### 3. 102 — FPGA 时序优化 AI 工具

五阶段 FPGA 时序优化闭环（PPA 的 Performance 维度）：

- 基线 PnR → 静态预筛（AI+LEC 闸门）→ 工具层搜索（参数扫）→ 越界探索（结构级）→ 知识沉淀
- **实测效果：** 333→348MHz，7-10% 频率提升

### 4. 801 — 对标 ARM Cortex-M7（深度面积篇）

面积分析（PPA 的 Area 维度）：

- 拆解 735 kGE 研究核的面积构成，发现 87% 的面积债锁在一个模块
- 功能对位 M7 后，用门数做硅片成本对比分析

### 5. 97 — AI 做 FPGA 时序优化的 Aha 时刻

FPGA 时序优化的"Aha 时刻"案例，属于 PPA Performance 维度的实践探索。

---

## 二、PPA 作为计划目标（尚未跑通）

### CPU性能与OpenLane-下一步.md（本地文件）

- **线 B：实现 PPA**（频率/面积/GDS），工具为 OpenLane Docker
- **状态：未拉镜像；有 Docker** — PPA 优化是明确的下一步计划，但尚未执行
- 建议路径："有数字后：是否要 OpenLane 做 PPA 再决定 pull"

---

## 三、PPA 作为方法论框架

### 工作流DAG梳理.md（本地文件）

PPA 在多个工作类型中作为签核闸门出现：

- **W1 FPGA 时序优化：** PnR 报告 Fmax/WNS 作为硬闸门
- **W3 RTL→签核：** "综合/STA/面积/fmax扫" → WNS/面积/fmax 可报告
- **公共骨架：** PnR 列为所有可交付案例的硬闸门之一

### P0-开源EDA安装清单.md（本地文件）

- OpenLane Docker 定位为"RTL→GDS 演示闸门"
- 安装清单支撑 PPA 闭环的前端工具链

---

## 小结

| 类别 | 案例 | PPA 维度 | 状态 |
|---|---|---|---|
| **已跑通** | 695/894 AHB总线矩阵 | P(频率)+P(功耗)+A(面积) | DC综合签核完成 |
| **已跑通** | 99 AutoResearch+FPGA | P(频率/Fmax) | 工作流闭环 |
| **已跑通** | 102 FPGA时序优化AI工具 | P(频率/Fmax) | 五阶段闭环，7-10%提升 |
| **已跑通** | 801 对标M7深度面积篇 | A(面积) | 面积拆解分析完成 |
| **已跑通** | 97 FPGA时序优化Aha | P(频率) | 实践探索 |
| **计划中** | CPU性能与OpenLane | P+A(OpenLane GDS) | 未拉镜像，待执行 |
| **框架** | 工作流DAG梳理 | P+A(签核闸门) | 方法论梳理 |
| **框架** | P0-开源EDA安装清单 | 工具链支撑 | 安装清单 |

真正意义上"跑通完整 PPA（频率+功耗+面积）优化"的案例是 **695/894 AHB 总线矩阵**——它跑通了 DC 综合签核并拿到了三维度数字。其余案例主要覆盖 PPA 的单一维度（频率或面积）。
