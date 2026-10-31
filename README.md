# Edge Semantic Gateway — 边侧语义调度网关

边侧语义调度网关：收端侧 RGB 流 + 子图流双路上行，语义调度后回 `SemanticDirective` 下行。只产出 frontier 级任务建议，不碰运动控制。

> 仓库名 `edge-semantic-gateway`；本目录沿用 `RGB_ws`。

## 系统角色

| 项目 | 角色 | 与本网关的关系 |
| --- | --- | --- |
| [quadruped-crossfloor-nav-ws](https://github.com/zxinyu-robot/quadruped-crossfloor-nav-ws) | Go2 端侧导航执行栈 | 端边协同主链路（`EDGE_HOST` ↔ `GO2_HOST`） |
| [legged-navigation-runtime](https://github.com/zxinyu-robot/legged-navigation-runtime) | 定位失效恢复 | 安全兜底，优先于一切任务指令 |

详细耦合设计见 [docs/v1-与跨楼层导航栈及恢复插件的耦合.md](docs/v1-与跨楼层导航栈及恢复插件的耦合.md)。

## 数据流

```text
Go2 端侧                            边侧（本网关）
┌──────────────────┐                ┌─────────────────────┐
│ Super-LIO/PCT/SCAN│  子图 :9877   │  subgraph 接收        │
│ edge_uplink      │ ─────────────► │  (frontiers/nodes)    │
│ rgb_forward      │  RGB  :9878    │  RGB 缓存             │
│                  │ ─────────────► │  语义调度（scheduler） │
│ directive_receiver│ ◄───────────── │  SemanticDirective 下行│
│                  │  :9879          │  (frontier 级建议)    │
└──────────────────┘                └─────────────────────┘
```

## 调度器

| 调度器 | 说明 | 需要 GPU |
| --- | --- | --- |
| `qwen_vl`（默认） | Qwen2.5-VL-3B，视觉+子图对齐选 frontier | 真推理需，无 GPU 用 stub |
| `layer_fusion_stub` | L0结构+L1视觉+L2任务分层融合（CPU 规则） | 否 |
| `geom_baseline` | 纯几何基线（frontier 分数最高者） | 否 |
| `streamvln` | StreamVLN 7B 官方 HTTP 协议 | 是 |
| `mock` | 仅测连通（取第一个 frontier） | 否 |

## 快速开始（G0 协议级闭环，无 GPU）

```bash
cd RGB_ws
chmod +x scripts/*.sh

# 一键 G0 仿真评测（回环 127.0.0.1，不会发到真狗）
./scripts/run_sim_bench.sh                              # layer_fusion_stub，G0 全过
./scripts/run_sim_bench.sh --scheduler qwen_vl          # 千问 stub 路径
./scripts/run_sim_bench.sh --scheduler mock --no-strict # 仅测连通
```

报告：`sim/reports/<timestamp>/summary.md`（含 G0 Gate 明细）。

真 GPU 边侧：

```bash
# config/edge.env 中 QWEN_VL_STUB=false，默认模型 Qwen2.5-VL-3B
./scripts/setup_qwen_vl.sh
./scripts/run_edge_server.sh

# 另一终端：本机模拟上行（无需狗）
python3 scripts/mock_uplink_client.py
```

与 Go2 真机联调：狗端 `edge_uplink.yaml` 里 `edge_host` 改为本机 IP；本机 `config/edge.env` 里 `GO2_HOST` 改为狗 IP。

## 可选支路

### StreamVLN（真机 VLN 替代调度）

Go2 端侧推 RGB 流 → x86 跑 StreamVLN 7B → 返回导航动作序列。详见 `docs/v1-世界模型-VLN-VLA选型方案.md`；服务端安装见 `scripts/setup.sh` + `scripts/download_model.sh` + `scripts/run_server.sh`。

### Gazebo 物理仿真

```bash
./gazebo/scripts/install_gazebo_harmonic.sh   # 首次：Ubuntu 24.04
./gazebo/scripts/run_corridor.sh              # corridor 三岔 + Go2 spawn
./gazebo/scripts/run_gazebo_bench.sh          # GG5–GG8 闭环
```

### Go2 URDF 可视化

```bash
./scripts/run_visual_sim.sh --png-only        # 导出 corridor 场景 PNG
```

## 目录结构

```text
edge/         边侧网关（对齐器/调度器/语义指令/千问客户端）
server/       StreamVLN x86 服务端
go2/          Go2 狗端客户端（拷到狗上）
sim/          协议级仿真（G0 评测）
gazebo/       Gazebo 物理仿真
monitor/      Foxglove 网页监控
config/       edge/sim/server/monitor 四组 env
docs/         方案与选型文档
```

## 设计文档

| 文档 | 说明 |
| --- | --- |
| [v1-双路上行语义调度方案.md](docs/v1-双路上行语义调度方案.md) | v1 总方案 |
| [v1-与跨楼层导航栈及恢复插件的耦合.md](docs/v1-与跨楼层导航栈及恢复插件的耦合.md) | 与端侧两项目的耦合 |
| [v1-技术验证标准.md](docs/v1-技术验证标准.md) | G0–G4 验证标准 |
| [v1-世界模型-VLN-VLA选型方案.md](docs/v1-世界模型-VLN-VLA选型方案.md) | 模型选型矩阵 |
| [v1-Gazebo与Isaac仿真选型.md](docs/v1-Gazebo与Isaac仿真选型.md) | 仿真选型 |

## 参考

- StreamVLN 论文 [arXiv:2507.05240](https://arxiv.org/abs/2507.05240)
- 真机权重 [mengwei0427/StreamVLN_Video_qwen_1_5_r2r_rxr_envdrop_scalevln_real_world](https://huggingface.co/mengwei0427/StreamVLN_Video_qwen_1_5_r2r_rxr_envdrop_scalevln_real_world)
