> 📄 **本目录仅含文档，不含仿真器源码**——尊重原作者的小范围传播授权，暂不公开代码，仅用于交流思路；项目完善后再整体收录。
> 源码获取与署名见 [仿真器须知.md](./仿真器须知.md)（原作者：河北科技大学 刘续 Jack；二次开发：成都大学 谢金恩 pawpaw）。
> 以下为原项目 README，其描述的 `simulator/`、`guided_dart/`、`example/` 源码目录**未随本仓库发布**。

# RoboMaster Simulator System

本项目将宿主机模拟器和接收端示例分成两个独立目录：

```text
simulator_system/
├── simulator/     # Rust + Bevy 视觉模拟器
├── guided_dart/   # 制导飞镖仿真模块，可由 daedalus 模式选择启动
├── example/       # sim_receiver 接收库与调用示例
├── API.md         # 通信协议与接口说明
├── USAGE.md       # 使用说明
└── TUTORIAL.md    # 第一次环境部署与配置教程
```

## 快速开始

```bash
# 1. 启动模拟器，默认监听 0.0.0.0:7666
cd simulator
./run_host.sh

# 2. 编译并运行接收示例
cd ../example
cmake -B build
make -C build -j4
./build/sim_receiver_example ../config/simulator.yaml
```

Windows 宿主机使用：

```bat
cd simulator
run_host.bat
```

当模拟器和运算端分别运行在两台主机上时，使用远程主机预设：

```bash
cd simulator
./run_remote.sh
```

该预设继续保持视觉端标定所需的 `1440x1080`，但将 JPEG 质量降到 `80`、采集
帧率降到 `20 FPS`，适合百兆链路或普通无线网络；同时默认设置
`DAEDALUS_PREVIEW=0`，关闭重复的窗口预览和主相机，只保留 TCP 离屏采集相机。
需要本地观察窗口时可显式设置 `DAEDALUS_PREVIEW=1`。需要更高画质时可覆盖
`DAEDALUS_JPEG_QUALITY` 和 `DAEDALUS_CAPTURE_FPS`。

模拟器跑在**没接显示器的远程主机（NUC / 服务器）**上时用 `run_nuc.sh`：它会
自动挑好那台机器自己的 Wayland/X 会话、检查 GPU 与锁屏（这台机器要有 GPU，
否则会退到 CPU 软件渲染），再按 `run_remote.sh` 的参数启动。

```bash
cd simulator
./run_nuc.sh          # 前台跑
./run_nuc.sh -d       # 后台跑，日志写 nuc_sim.log
```

Windows、macOS 和 Linux 的宿主机启动脚本均默认使用 Release 构建，并将
`CARGO_BUILD_JOBS` 设为 `2`。脚本会先完成构建，再直接运行生成的程序；
已安装 `sccache` 时会自动使用缓存。内存较小的环境可将该变量设为 `1`，内存
充足时可提高到 `3` 或更高。

## 制导飞镖场景

`guided_dart/` 是视觉模拟器中的可选仿真模式，包含固体舵面挂载、四片舵机模型、气动力/力矩、发射物理、第一人称相机和打击基地。**它只做物理，不含任何制导律**：控制算法在视觉端，通过 `DART <pitch> <yaw> <roll>`（弧度）报文下发舵角，仿真器执行后再按 `[vision] state_telemetry_hz` 回推 `STATE` 状态。将 `simulator/config/camera.yaml` 中的 `scenario` 改为 `guided_dart`，再正常启动 `simulator` 即可进入该模式；启用 `tcp-only` 时，飞镖第一人称相机也会作为 TCP 图像源。

```bash
cd simulator
./run_host.sh
```

飞镖模式操作：按 `F` 发射，按 `R` 复位，按 `V` 在主/副视角之间切换。默认**主视角是第三人称跟随镜头**（跟在飞镖后方，能看清镖体姿态与舵面偏转），**副视角是机头第一人称**、缩成右下角一块 4:3 画中画；关掉跟随镜头后机头相机恢复铺满窗口。TCP 图像流抓的始终是机头相机（它带 `CameraRig`），窗中布局不影响回传给视觉端的画面。模型、相机、PID 舵机执行和发射参数仍由 `guided_dart/config/` 管理，也可以单独进入 `guided_dart/` 调试该模块。飞镖模式在 `0.0.0.0:7667` 接收 `DART`/`SERVO` 舵角报文、`GIMBAL` 调试报文、`FIRE` 和遗留的 `szu_uart` 二进制帧，并回推 `STATE` 状态行。

飞镖场景默认隐藏官方模型中的完整场地地面，并沿发射仓到基地重新生成局部视觉
地面；原模型仍保留给物理碰撞使用。需要恢复完整官方地面时设置
`DAEDALUS_FIELD_CROP=0`。

`armor_chassis` 场景下按 `R` 键可开启/关闭整车小陀螺，默认转速为 `5 rad/s`
（可用 `DAEDALUS_SMALL_GYRO_SPEED` 覆盖，实车常态约 1.5~3 rad/s）。

`outpost` 场景用官方整场模型里的前哨站（`guided_dart/assets/models/outpost.glb`，
是用 `simulator/tools/extract_glb_subtree.py` 从 `rmuc2026.glb` 抽出来的
`001_ASM-2` 子树）：**塔体**用模型，**装甲板**换成代码生成的那套（官方图标
贴纸 + 两侧发光红条，尺寸按 `DAEDALUS_OUTPOST_*`）。只有**上面那一圈**在转：
三块高低错落的大装甲板（位置取模型实测的 1.116 / 1.216 / 1.316 m）+ 挂板支架 +
顶盖，塔座、立柱和底盘固定，默认 `0.8π rad/s`。模型文件缺失时塔体自动回退到
程序化几何。该场景导出 `dataset.csv`（每帧 3 行）与 `light_bars.csv`（每帧 6 行）。

`base` / `rmuc2026` 两个场景用于**直接查看整场模型**（`base.glb` 是 25 赛季的
飞镖赛场，`rmuc2026.glb` 是官方整场模型）。它们自带地面、不锁相机，鼠标即可
绕着看。

详细步骤见 `TUTORIAL.md`。

## 当前版本

当前版本为 `Bata-3.2.1`，详细版本记录由 `version.txt` 维护。

已实现功能：

- 能量机关仿真
- 原始整车模型仿真
- 前哨站仿真（官方塔体 + 程序化装甲板：三块高低错落的板随支架绕轴旋转，塔身固定）
- 场地查看场景 `base` / `rmuc2026`（整场模型直接加载，自由视角）
- 整车小陀螺，按 `R` 键开启/关闭，默认 `5 rad/s`
- 按 `E` 键复位相机和机器人状态
- 整车相对相机移动，使用 `W/A/S/D`
- TCP 图像流与推流画质优化
- 宿主机 `dataset.csv` 装甲板数据集导出
- 宿主机 `light_bars.csv` 八灯条位置与时间戳导出
- `plot_results.py` 数学曲线描绘与单装甲板选择
- TCP 回发射击、云台直控与纯视觉接管 `GIMBAL_VISION`
- 相机与机器人复位功能
- 飞镖视觉仿真及四种打击模式

接口与协议说明见 `API.md`，操作说明见 `USAGE.md`，部署教程见 `TUTORIAL.md`。
