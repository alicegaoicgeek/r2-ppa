# 跑 CPU 看性能 + OpenLane — 当前状态与下一步

更新：本机已装一批工具（见下）。目标分两条线，别混。

---

## 1. 两条线

| 线 | 问题 | 主要工具 | 状态 |
|---|---|---|---|
| **A 软件/仿真性能** | CoreMark、周期数、功能 | verilator + riscv-gcc +（可选）spike | **gcc 已可编 RV32**；差 spike；差真实核工程 |
| **B 实现 PPA** | 频率/面积/GDS | OpenLane Docker | **未拉镜像**；有 Docker |

---

## 2. 本机工具状态（2026-07-10）

### 已可用

| 工具 | 版本/路径 | 用途 |
|---|---|---|
| yosys | 0.66（随 sby 升级） | 综合 |
| verilator | 5.044 | **CPU 仿真主路径** |
| iverilog | 13 | 备选仿真 |
| z3 | 已有 | formal 后端 |
| **sv2v** | 0.0.13 | SV→Verilog |
| **sby** | 0.67 | formal 前端 |
| **riscv64-elf-gcc** | 16.1.0（支持 `-march=rv32imac`） | **编基准/测试程序** |
| fusesoc | 已有 | 包管理 |
| omp | 已有 | AI 执行 |
| docker | 已有 | OpenLane 容器 |
| dtc | 1.8.1 | 编 spike 依赖 |
| **eqy** | `~/.local/bin/eqy` → `~/tmp/eqy` + venv | 形式等价 |
| **cocotb** | `~/tools/eda-venv` 内 2.0.1 | Python TB（可选） |

### 未装 / 缓装

| 工具 | 说明 |
|---|---|
| **spike** | brew 的 spike 是乐高，不是 RISC-V；需源码编译 |
| gtkwave | brew cask 已 deprecated |
| OpenLane 镜像 | 未 pull，体积大 |

### 环境注意

```bash
# 每次开终端建议
export PATH="/opt/homebrew/bin:$HOME/.local/bin:$HOME/.bun/bin:$PATH"
# cocotb / eqy 用 venv 时：
source "$HOME/tools/eda-venv/bin/activate"
```

**编 RV32 程序：**

```bash
riscv64-elf-gcc -march=rv32imac -mabi=ilp32 ...
```

（公式名是 riscv64，但 **multi-lib 含 rv32**，已冒烟通过。）

---

## 3. 线 A：跑 CPU 看性能（你优先）

### 3.1 还差什么

1. **一个带 sim 的核工程**（推荐从小到大）  
   - PicoRV32 + picosoc  
   - 或 tinyriscv  
   - 或你现有仓库  
2. **基准程序**（CoreMark / Dhrystone / 仓库自带）  
3. （可选）**spike** 做软件金模对照  

### 3.2 最小操作环

```text
改 RTL
  → riscv64-elf-gcc -march=rv32… 编基准
  → verilator 仿真
  → 读分数 / 周期
  → 不够再改
```

闸门：功能测试通过 + 记录 CoreMark/MHz（或周期）。

### 3.3 下一步（你点头再做）

任选：

- **A1** clone PicoRV32/picosoc，跑通默认 sim  
- **A2** 源码安装 spike  
- **A3** 指定你本地已有核仓库路径，对着改脚本  

---

## 4. 线 B：OpenLane（实现侧）

### 4.1 何时上

- 要 **sky130 上时序/面积/GDS**  
- 不替代 CoreMark  

### 4.2 安装（Docker，约数 GB～几十 GB）

```bash
# 确认
docker info >/dev/null

# 镜像名以官方当前文档为准，常见：
docker pull efabless/openlane:latest
# 或 OpenLane2 文档推荐镜像

# 冒烟
docker run --rm efabless/openlane:latest yosys -V
```

首次跑完整 flow 前再按文档挂 PDK。

### 4.3 与线 A 组合

```text
功能/基准 OK（线 A）
  → 小模块/小核丢进 OpenLane
  → 看 WNS / 面积
  → 再改 RTL
```

---

## 5. 建议顺序（不绕）

1. **本周：** 线 A — 固定一个小核 + verilator + 出第一个性能数字  
2. **有数字后：** 是否要 OpenLane 做 PPA 再决定 pull  
3. **spike：** 需要「无 RTL 先验软件」再编  

---

## 6. 本次已执行安装命令摘要

```bash
brew install sv2v sby riscv64-elf-gcc dtc
# yosys 顺带升到 0.66
# python@3.12 venv + cocotb + click
# ~/.local/bin/eqy 包装
```
