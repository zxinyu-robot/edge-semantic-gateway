# Edge Semantic Gateway — Go2 端边语义导航网关

> 仓库名 `edge-semantic-gateway`；本目录沿用 `RGB_ws`。

## 文档

| 文档 | 说明 |
|------|------|
| [`docs/v1-双路上行语义调度方案.md`](docs/v1-双路上行语义调度方案.md) | v1 总方案：RGB 流 + 子图流双路上行，SemanticDirective 下行 |
| [`docs/v1-端侧剩余工作清单.md`](docs/v1-端侧剩余工作清单.md) | 端侧待办（Super-LIO / BT 已跑通前提下） |
| [`docs/v1-RGB硬件编码上行方案.md`](docs/v1-RGB硬件编码上行方案.md) | SHM→NVENC H.265 上行，5070 NVDEC 解码 |
| [`docs/v1-仿真环境技术选型方案.md`](docs/v1-仿真环境技术选型方案.md) | 本机协议级仿真 + 选型评测（无需 GPU/真狗） |
| [`docs/v1-世界模型-VLN-VLA选型方案.md`](docs/v1-世界模型-VLN-VLA选型方案.md) | 2026 WM/WAM/VLN/VLA 选型矩阵与分阶段推荐 |
| [`docs/v1-技术验证标准.md`](docs/v1-技术验证标准.md) | 专网 VLN、任务成功率、边端流畅性 G0–G4 验证标准 |
| [`docs/v1-Gazebo与Isaac仿真选型.md`](docs/v1-Gazebo与Isaac仿真选型.md) | Gazebo vs Isaac 选型 + **Gazebo 线**（`gazebo/`） |

## v1 边侧网关（本机）

收 Go2 上行子图 `:9877` + RGB `:9878`，语义调度后回 `SemanticDirective` 到狗 `:9879`。

**默认 scheduler**：`qwen_vl`（Qwen2.5-VL-7B INT4，见 `config/edge.env`）

```bash
cd RGB_ws
chmod +x scripts/*.sh

# 1. 安装 Qwen-VL 依赖（5070 + CUDA，首次）
./scripts/setup_qwen_vl.sh

# 2. 启动边侧（真 GPU：edge.env 中 QWEN_VL_STUB=false）
./scripts/run_edge_server.sh

# 无 GPU 仅测协议：QWEN_VL_STUB=true ./scripts/run_edge_server.sh

# 终端 2：本机自测（无需狗）
python3 scripts/mock_uplink_client.py

# 与 Go2 联调：狗 edge_uplink.yaml 里 edge_host 改为本机 IP
```

配置：`config/edge.env`（`SCHEDULER`、`QWEN_VL_*`、`GO2_HOST` 等）

## 本机仿真（G0 协议级闭环）

纯 Python、**同一 MSRGB/子图/Directive 协议**，L0/L1/L2 分层对齐 + scheduler 选型对比：

```bash
chmod +x scripts/run_sim_bench.sh
./scripts/run_sim_bench.sh                              # G0 全场景 strict 通过
./scripts/run_sim_bench.sh --scenario corridor_3frontiers
./scripts/run_sim_bench.sh --scheduler streamvln        # S3 官方 HTTP 协议 stub
./scripts/run_sim_bench.sh --scheduler mock --no-strict # 仅测连通
```

报告：`sim/reports/<timestamp>/summary.md`（含 G0 Gate 明细）  
配置：`config/sim.env`（回环 127.0.0.1，不会发到真狗）

## Go2 URDF 3D 可视化

基于你提供的 `Go2_URDF.zip`（已解压到 `assets/GO2_URDF/`），corridor 场景回放：

```bash
chmod +x scripts/run_visual_sim.sh
./scripts/run_visual_sim.sh              # 开 Open3D 窗口 + 导出 PNG
./scripts/run_visual_sim.sh --png-only   # 仅导出 PNG（无 GUI）
```

输出：`sim/reports/visual/<timestamp>/step_*.png`  
图例：绿=厨房 frontier · 红=走廊 · 紫=卧室

## Gazebo 物理仿真（corridor + Go2）

全物理仿真线（与 G0 协议级仿真互补）：

```bash
chmod +x gazebo/scripts/*.sh
./gazebo/scripts/install_gazebo_harmonic.sh   # 首次：Ubuntu 24.04 装 gz harmonic
./gazebo/scripts/run_corridor.sh              # corridor 三岔 + Go2 spawn
```

文档：[`docs/v1-Gazebo与Isaac仿真选型.md`](docs/v1-Gazebo与Isaac仿真选型.md) · [`gazebo/README.md`](gazebo/README.md)

**GG5–GG8 闭环**（本机、不动真狗）：

```bash
./gazebo/scripts/run_gazebo_bench.sh                              # live + Gazebo
./gazebo/scripts/run_gazebo_bench.sh --mode scenario --no-gz    # 仅协议桥
./gazebo/scripts/run_gazebo_bench.sh --scheduler streamvln        # StreamVLN 支路
./gazebo/scripts/run_gazebo_bench.sh --frontier-source scan       # LiDAR 推导 frontier
./gazebo/scripts/run_corridor.sh              # 开 GUI（前台，关窗口才退出）
./gazebo/scripts/run_nav_loop.sh              # 导航；默认不杀 Gazebo
./gazebo/scripts/run_gazebo_bench.sh --frontier-source scan --mode live
# 若 bench 后 Gazebo 没了：旧版脚本会 exit 时 kill；现已默认 --keep-gz
./gazebo/scripts/stop_gazebo.sh               # 手动停止
```

## StreamVLN 支路（可选）

Go2 端侧推送 RGB 流（~30fps）→ x86 跑 StreamVLN 7B → 返回导航动作 → 狗端解析执行。

基于 [StreamVLN 官方 Go2 真机部署](https://github.com/InternRobotics/StreamVLN/tree/main/realworld)。

## 架构

```
Go2 (ROS2)                         x86 GPU 服务器
┌─────────────────┐                ┌──────────────────────┐
│ RGB ~30fps      │  HTTP POST     │ StreamVLN 7B         │
│ /sportmodestate │ ─────────────► │ /eval_vln            │
│                 │  JPEG + JSON   │                      │
│ go2_vln_client  │ ◄───────────── │ 返回 action 序列      │
│ PID → /cmd_vel  │  [1,1,2,0]     └──────────────────────┘
└─────────────────┘
```

## 动作协议


| 值   | 含义         | 狗端执行     |
| --- | ---------- | -------- |
| `0` | STOP       | 停止，任务结束  |
| `1` | FORWARD    | 前进 25 cm |
| `2` | TURN LEFT  | 左转 15°   |
| `3` | TURN RIGHT | 右转 15°   |


HTTP 接口：

```bash
POST /eval_vln
Content-Type: multipart/form-data

  image: JPEG 文件
  json:  {"reset": true, "instruction": "走到门口停下"}
```

响应：

```json
{"action": [1, 1, 2, 0], "llm_output": "...", "done": false}
```

## x86 服务端安装

需要：NVIDIA GPU + CUDA 12.x + conda

```bash
cd RGB_ws
chmod +x scripts/*.sh

# 1. 安装依赖 + 克隆 StreamVLN
./scripts/setup.sh

# 2. 下载真机 checkpoint (~16GB)
./scripts/download_model.sh

# 3. 启动服务（默认 0.0.0.0:5801）
./scripts/run_server.sh
```

配置见 `config/server.env`（模型路径、端口、默认指令等）。

健康检查：`curl http://localhost:5801/health`

## Go2 狗端部署

把 `go2/` 目录拷到 Go2，修改 `go2_vln_client.py` 顶部配置：

```python
SERVER_URL = "http://<x86-IP>:5801/eval_vln"
INSTRUCTION = "你的导航指令"
RGB_TOPIC = "/你的/rgb/topic"
```

运行（Go2 上 ROS2 Foxy + unitree_ros2）：

```bash
python3 go2_vln_client.py
# 或
python3 go2_vln_client.py "http://192.0.2.100:5801/eval_vln" "走到红色门停下"
```

**逻辑**：狗端只在需要重规划时上行一帧 RGB（不是每帧都推理）。StreamVLN 每 4 步跑一次模型，PID 跟踪到位后触发下一次规划。30fps RGB 用于本地缓存最新帧。

## 本地测试

```bash
# 服务端启动后
python3 scripts/test_client.py --reset --instruction "Walk forward."
```

## 目录结构

```
RGB_ws/
├── config/server.env          # 服务端配置
├── scripts/
│   ├── setup.sh               # 安装 StreamVLN 环境
│   ├── download_model.sh      # 下载 HF checkpoint
│   ├── run_server.sh          # 启动推理服务
│   └── test_client.py         # 模拟上行测试
├── server/
│   └── http_realworld_server.py
├── go2/                       # 拷到 Go2 端
│   ├── go2_vln_client.py
│   ├── pid_controller.py
│   └── utils.py
├── third_party/StreamVLN/     # setup.sh 自动克隆
└── checkpoints/               # download_model.sh 下载
```

## 参考

- StreamVLN 论文: [arXiv:2507.05240](https://arxiv.org/abs/2507.05240)
- 真机权重: [mengwei0427/StreamVLN_..._real_world](https://huggingface.co/mengwei0427/StreamVLN_Video_qwen_1_5_r2r_rxr_envdrop_scalevln_real_world)

