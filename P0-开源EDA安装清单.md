# P0 开源 EDA 安装清单（macOS arm64）

> 目标：补齐前端闭环闸门，给 AI 作业用  
> 机器：已有 yosys / verilator / iverilog / z3 / fusesoc / eqy / omp / docker  

---

## 0. 装前自检

```bash
export PATH="$HOME/.bun/bin:/opt/homebrew/bin:$PATH"

echo "=== 已有 ==="
yosys -V | head -1
verilator --version | head -1
iverilog -V 2>&1 | head -1
z3 --version
which fusesoc omp docker
test -f "$HOME/tmp/eqy/src/eqy.py" && echo "eqy: $HOME/tmp/eqy/src/eqy.py"
```

---

## 1. Homebrew（波形 / SV 前端 / formal 入口）

```bash
brew update
brew install gtkwave

# sv2v：SystemVerilog → Verilog（喂 yosys）
brew install sv2v || brew install --HEAD sv2v

# SymbiYosys（sby）：开源 formal 入口（公式名因 brew 版本可能不同）
brew search sby
brew search symbiyosys
# 任选能装上的一个，例如：
# brew install sby
# 或： brew install symbiyosys
```

**自检：**

```bash
which gtkwave && gtkwave --version 2>&1 | head -1
which sv2v && sv2v --version 2>&1 | head -1
which sby && sby --help 2>&1 | head -3
```

若 `sby` 无 formula：用 YosysHQ 源码装，或 Docker 内用 sby（见 §3）。

---

## 2. Python：cocotb

```bash
python3 -m pip install -U pip
python3 -m pip install cocotb
```

**自检：**

```bash
python3 -c "import cocotb; print('cocotb', cocotb.__version__)"
```

---

## 3. RISC-V 工具链 + Spike

### 3.1 交叉编译器（优先预编译，避免从源码编数小时）

```bash
# 方案 A：brew（有则优先）
brew search riscv
# 常见：
# brew install riscv-gnu-toolchain
# 或 riscv64-elf-gcc 一类 formula，以 brew search 结果为准

# 方案 B：官方/社区预编译包
# 下载后解压并加入 PATH，例如：
# export PATH="$HOME/tools/riscv/bin:$PATH"
```

**自检：**

```bash
riscv64-unknown-elf-gcc --version 2>/dev/null \
  || riscv32-unknown-elf-gcc --version 2>/dev/null \
  || riscv64-elf-gcc --version 2>/dev/null \
  || echo "NEED: install riscv gcc"
```

### 3.2 Spike（ISA 金模）

```bash
# 若 brew 有：
# brew install spike

# 否则源码（依赖 dtcs 等，按官方 README）：
# git clone https://github.com/riscv-software-src/riscv-isa-sim.git
# cd riscv-isa-sim && mkdir build && cd build
# ../configure --prefix=$HOME/tools/spike
# make -j$(sysctl -n hw.ncpu) && make install
# export PATH="$HOME/tools/spike/bin:$PATH"
```

**自检：**

```bash
spike --help 2>&1 | head -3 || echo "NEED: spike"
```

---

## 4. EQY 进 PATH（你已有源码）

```bash
# 若已 make 过：
export EQY_SCRIPT="$HOME/tmp/eqy/src/eqy.py"
# 可选包装：
mkdir -p "$HOME/.local/bin"
cat > "$HOME/.local/bin/eqy" <<'EOF'
#!/usr/bin/env bash
exec python3 "${EQY_SCRIPT:-$HOME/tmp/eqy/src/eqy.py}" "$@"
EOF
chmod +x "$HOME/.local/bin/eqy"
export PATH="$HOME/.local/bin:$PATH"
```

**自检：**

```bash
python3 "$HOME/tmp/eqy/src/eqy.py" -h 2>&1 | head -5
```

---

## 5. Docker：OpenLane 演示（后端，不装原生）

```bash
docker --version

# 拉取（镜像名以官方当前文档为准，以下为常见写法）
docker pull efabless/openlane:latest
# 或 OpenLane2 文档推荐镜像

# 冒烟：进容器看工具是否在
docker run --rm efabless/openlane:latest yosys -V
```

**说明：** PDK（如 sky130）按 OpenLane 文档挂载；第一次体积大，预留几十 GB。

---

## 6. 装完总自检脚本

```bash
export PATH="$HOME/.bun/bin:$HOME/.local/bin:/opt/homebrew/bin:$PATH"

check() {
  if command -v "$1" >/dev/null 2>&1; then echo "OK  $1"; else echo "NO  $1"; fi
}

check yosys
check verilator
check iverilog
check z3
check gtkwave
check sv2v
check sby
check fusesoc
check omp
check docker
python3 -c "import cocotb; print('OK  cocotb', cocotb.__version__)" 2>/dev/null || echo "NO  cocotb"
test -f "$HOME/tmp/eqy/src/eqy.py" && echo "OK  eqy.py" || echo "NO  eqy.py"
riscv64-unknown-elf-gcc --version >/dev/null 2>&1 && echo "OK  riscv-gcc" || echo "NO  riscv-gcc"
spike --help >/dev/null 2>&1 && echo "OK  spike" || echo "NO  spike"
```

期望：**P0 标 OK 的越多越好**；`sby` / `riscv-gcc` / `spike` 若 NO，按上面对应节补。

---

## 7. 本阶段明确不做

- 本机从源码编译完整 OpenROAD/OpenLane  
- 一次装齐所有 RISC-V 开源核  
- 把 55nm 流片班车当日常依赖  

---

## 8. 与作业闸门对照

| 装好后 | 可支撑 |
|---|---|
| yosys + sv2v | 综合网表 |
| verilator/iverilog + gtkwave | 仿真 + 看波 |
| eqy / sby + z3 | 等价 / formal |
| cocotb | Python 向 TB 与 AI 闭环 |
| riscv-gcc + spike | 核软件与 ISA 金模 |
| docker openlane | RTL→GDS 演示闸门 |
| omp | AI 执行节点 |
