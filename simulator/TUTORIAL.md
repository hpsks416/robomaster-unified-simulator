# 第一次环境部署教程

## 最短路径：第一次跑通

只有下面四条命令，跑完就能看到模拟器画面、并且从图像流里收到图。原理和参数都在
后面的章节里，先按这个跑通再回头看。

**先决条件**（只需装一次）

- Rust 工具链：`cargo --version` 有输出就行，没有看下面的 §2「安装 Rust」。
- Linux（Ubuntu/Debian）还要装几个开发库，一条 apt 命令，见 §2「Linux 依赖」。
- CMake + OpenCV + yaml-cpp：只有编译 `example/` 接收示例才需要（装法见 §3），
  只想看模拟器画面可以跳过。

**1. 启动模拟器**

```bash
cd simulator
./run_host.sh          # Windows: run_host.bat
```

第一次要下载并编译 Rust 依赖，**会等几分钟**；内存 4 GiB 左右的环境先用
`CARGO_BUILD_JOBS=1 ./run_host.sh`。跑起来后会自动弹出一个预览窗口，同时开始往
`0.0.0.0:7666` 推图像流（窗口只是预览，真正发给算法的是 `1440x1080` 那一路）。

**2. 编译接收示例**（另开一个终端）

```bash
cd example
cmake -S . -B build && cmake --build build -j4
```

**3. 收一帧图**

```bash
cd example
./build/sim_receiver_example ../config/simulator.yaml
```

示例每 30 帧在**当前目录**存一张 `frame_XXXX.jpg`（收到 300 帧后自动退出），
能存出图就说明模拟器 ↔ 接收端通了。

**4. 换个场景看看**

改 `simulator/config/camera.yaml` 里的 `scenario:` 再重启模拟器：
`armor_chassis`（整车，默认）、`energy_large`（大符）、`outpost`（前哨站）、
`guided_dart`（飞镖）、`rmuc2026`（整场模型）。完整列表和各场景说明见 §4。

跑通之后：通信协议看 `API.md`，参数与操作键位看 `USAGE.md`，场景细节从下面
§4 开始。

## 1. 目录说明

```text
simulator_system/
├── simulator/     # 模拟器源码与资源
├── guided_dart/   # 制导飞镖仿真模块
├── example/       # 接收库与调用示例
├── API.md
├── USAGE.md
└── TUTORIAL.md
```

## 2. 宿主机部署模拟器

### Linux 依赖（Ubuntu / Debian）

Ubuntu 22.04 实测，编译模拟器需要下面这几个库：

```bash
sudo apt update
sudo apt install -y build-essential pkg-config \
  libwayland-dev libxkbcommon-dev libasound2-dev
```

`libwayland-dev` / `libxkbcommon-dev` 是 winit 的 Wayland 后端要的（缺了会在
`wayland-sys` 的 build script 里报 `wayland-client.pc not found`），
`libasound2-dev` 是 `bevy_audio` 走 ALSA 要的。运行期还需要 Vulkan 驱动：

```bash
sudo apt install -y libvulkan1 mesa-vulkan-drivers
```

**图像流编码**：启动脚本默认带上 `fast-jpeg`（libjpeg-turbo）编码器，需要
`cmake`；机器上已经有 libjpeg-turbo 的话走 pkg-config 也行，再装个 `nasm`
还能快一点。实测 1440x1080：编码 **22ms → 6.5ms**，`run_host.sh` 默认档
（JPEG 95）推流 **9~12 → 28 fps**。缺 cmake 时脚本会自动退回纯 Rust 编码器
（帧率低一些但照样能跑），也可以显式关掉：`DAEDALUS_FAST_JPEG=0 ./run_host.sh`。
`fast-jpeg` 已经是 Cargo 的默认 feature，直接 `cargo build` 也会带上；机器上
实在装不了 C 工具链时用 `--no-default-features` 去掉它。

**显卡**：模拟器靠 Vulkan 渲染，`ls /dev/dri` 里有设备才说明 GPU 可用。没有
GPU（或内核命令行带 `nomodeset` 导致 iGPU 不绑定）时会退到 CPU 的 llvmpipe
软件渲染，1440x1080 只剩个位数 fps——这种情况请把**仿真器**放到有 GPU 的机器上
跑（仿真器自己也会在日志里提示）。详见 §6 故障排查。

### 安装 Rust

Windows：

```powershell
winget install Rustlang.Rustup
```

macOS / Linux：

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

验证：

```bash
cargo --version
```

### 启动模拟器

Windows：

```bat
cd simulator
run_host.bat
```

macOS / Linux：

```bash
cd simulator
./run_host.sh
```

模拟器和运算端分别位于两台主机时，优先使用：

```bash
cd simulator
./run_remote.sh
```

该预设保持 `1440x1080` 的视觉标定尺寸，默认使用 JPEG 80、20 FPS，以减少
百兆链路或无线网络的拥塞。远程预设还会设置 `DAEDALUS_PREVIEW=0`，关闭窗口
预览和重复的主相机渲染，只保留 TCP 图像流；临时查看画面时可设置
`DAEDALUS_PREVIEW=1`。若 `iperf3` 测得链路吞吐充足，可通过
`DAEDALUS_JPEG_QUALITY` 和 `DAEDALUS_CAPTURE_FPS` 提高画质或帧率。

仿真器跑在远程主机（NUC）上时，把 `camera.yaml` 的 `protocol` 改成 `udp`
（`tcp` 是默认）即可再开一路 UDP 图像流（默认 `0.0.0.0:7669`，与 TCP 并存），
延迟/抖动比 TCP 好；接收端配置写 `protocol: udp` 即可。细节见 `API.md` §1.1 与
`USAGE.md` 的 UDP 低延迟模式。

首次运行会下载并编译 Rust 依赖，需要几分钟。宿主机脚本默认使用双任务
编译；4 GiB 左右或容易内存不足的环境可在启动前设置 `CARGO_BUILD_JOBS=1`。
脚本会保留 `target` 增量缓存，并直接运行已构建的程序。安装 `sccache` 后，
脚本会自动启用依赖缓存。默认监听 `0.0.0.0:7666`。

### 远程主机（NUC / 服务器）上跑

`run_remote.sh` 保持 1440x1080 的采集尺寸、20 FPS、JPEG 80，并关掉窗口预览
（`DAEDALUS_PREVIEW=0`），正好适合远程主机；接收端把
`example/config/simulator.yaml` 里的 `host` 改成这台机器的 IP 即可。

**直接用 `run_nuc.sh` 最省事**：它会把下面这些检查走一遍再启动（自动挑
Wayland/X 会话、检查 GPU 与锁屏、非交互式 SSH 下补上 `~/.cargo/bin`）：

```bash
cd simulator
./run_nuc.sh          # 前台跑，Ctrl-C 结束
./run_nuc.sh -d       # 后台跑，日志写到 ./nuc_sim.log，并打印 PID/停止命令
```

参数顺序和 `Ultra_Vision/tools/obs_gui.sh` 一致，也可以直接给场景参数（**小陀螺
转速 rad/s、相机到靶的水平距离 m**）：

```bash
./run_nuc.sh 5.0 6.0        # 5 rad/s 小陀螺 + 相机距靶 6.0 m
./run_nuc.sh -d 3.0 6.5     # 后台跑 + 3 rad/s + 6.5 m
```

第 2 个参数会在 `/tmp` 生成一份相机配置（`scenario` 沿用当前配置，只追加
`distance:`），不用手改 yaml；高度差保持不变，所以实际三维距离会略大于这个值。
不传就沿用 `run_nuc.local.sh` / 环境变量里的值。

它默认还会**关掉远端云台控制**（`DAEDALUS_GIMBAL_CONTROL=0`）：远程主机通常只当
稳定图像源用，视觉端发来的 `GIMBAL` / `GIMBAL_VISION` 会被忽略，相机固定在
`camera.yaml` 的开机位姿、图像包头里的姿态一直是 0（`FIRE` / `RESET` 照常生效）。
要恢复视觉端控云台：`DAEDALUS_GIMBAL_CONTROL=1 ./run_nuc.sh`。

下面是它替你做的事，想手动传环境变量也行。

**要跑在那台机器自己的图形会话里**（桌面登录着就行，窗口是隐藏的）：

```bash
# Wayland 会话（Ubuntu 22.04 默认）
XDG_RUNTIME_DIR=/run/user/$(id -u) WAYLAND_DISPLAY=wayland-0 ./run_remote.sh
# 或者 X11 / XWayland
DISPLAY=:0 ./run_remote.sh
```

⚠️ **别用 `xvfb-run` 跑有 GPU 的机器**：Xvfb 没有 DRI3，Intel 驱动在它上面无法
呈现，会直接 abort（`No supported formats for surface`）；就算退到软件渲染
（llvmpipe），1440x1080 也只有个位数 fps。

⚠️ **别让会话锁屏/黑屏**：GNOME 一进锁屏就不再合成画面，仿真器会卡在出帧上
（现象：进程还在，但接收端收不到帧）。长期跑之前先关掉：

```bash
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.desktop.screensaver lock-enabled false
```

⚠️ `DAEDALUS_CAPTURE_FPS` 按机器实际能力给：设到远超实际能跑的值时，采集循环会
被拖死、接收端直接收不到帧（实测把 20 改成 60 就会这样）。用 `run_remote.sh`
的默认值即可。

⚠️ **帧率 / 带宽预算**：`一帧字节数 × 帧率` 就是要占的带宽。1440x1080 下实测
JPEG 80 ≈ 77 KiB/帧（20 fps → 12 Mbps）、JPEG 70 ≈ 50 KiB/帧（20 fps → 8 Mbps）；
普通 Wi-Fi 实测只有 ~20 Mbps。所以想提帧率：先把质量降到 70 上下，再调
`DAEDALUS_CAPTURE_FPS`。**注意发送速率不能超过接收端的消费速率**——超了以后积压
会全堆在接收端的 socket 缓冲里（实测把采集目标从 20 改成 60，仿真器自己编到
29 fps，但算法端拿到的帧已经过期 6 秒，`source_delay=6090ms`）。接收端应该"丢掉
积压、只处理最新一帧"，这样丢几帧反而无所谓——这正是 UDP 那一路的设计前提。

每台机器固定的档位可以写进同目录的 `run_nuc.local.sh`（`run_nuc.sh` 会 source
它，命令行/环境变量优先级仍然更高）：

```bash
# simulator/run_nuc.local.sh
export DAEDALUS_JPEG_QUALITY=70
export DAEDALUS_CAPTURE_FPS=30
# 远程主机没有键盘、按不了 R：开机直接小陀螺（`RESET` 后也保持）
export DAEDALUS_SMALL_GYRO=1
export DAEDALUS_SMALL_GYRO_SPEED=5.0
```

**采集流水线是异步的**：GPU 渲染 → 拷贝到缓冲 → 回读 → JPEG 编码 → 发送。
拷贝用**双缓冲**轮转，和"回读 + 编码"并行（以前只有一个缓冲槽，GPU 拷贝必须等
上一帧编码完才能发生，采集被硬性量化成两帧一次）；编码直接吃映射内存里的 RGBA，
不再复制一份、也不做 RGBA→RGB 转换。1440x1080 实测：**采集 20 → 39 fps**（不再被
回读卡住，`map_avg` 从 ~17ms 降到 **0.2ms**），渲染始终 60 fps，GPU 并没到极限。
发送侧是"**发不出去就丢整帧**"（链路拥塞时掉帧，而不是把流水线拖慢），丢了多少
看统计行的 `udp_drop=`。

诊断用：

```bash
DAEDALUS_FPS_PROBE=1 ./run_nuc.sh -d   # 每秒打印一次「渲染帧率」
```

`[tcp]` 统计行里的 `cap=`（采集目标）、`map_avg=`（等回读）、`encode_avg=`（编码）、
`udp_drop=`（为保帧率丢掉的帧）可以一眼看出瓶颈在哪一段。

## 3. 部署接收示例

### 安装依赖

Ubuntu：

```bash
sudo apt update
sudo apt install -y cmake g++ libopencv-dev libyaml-cpp-dev
```

macOS：

```bash
brew install cmake opencv yaml-cpp
```

Windows：安装 CMake 和 OpenCV，并保证 CMake 能找到 `OpenCVConfig.cmake`，或使用 vcpkg 安装 `opencv4`、`yaml-cpp`。

### 编译

```bash
cd example
cmake -S . -B build
cmake --build build -j4
```

## 4. 配置修改

### 模拟器相机配置

文件：`simulator/config/camera.yaml` —— **只提供模式切换**，全文只有一项：

```yaml
scenario: armor_chassis
```

`scenario` 选择场景（`energy_rune` / `energy_small` / `energy_large` /
`armor_chassis` / `outpost` / `guided_dart` / `base` / `rmuc2026`，也可用
`DAEDALUS_SCENARIO` 覆盖），修改后重启模拟器。三个能量机关场景的区别只在两面
用什么符（模型是两面背靠背的，固定相机只看得见 `FACE_1` 那侧）：`energy_rune`
保持原样（可见侧是小符），`energy_small` / `energy_large` 让两面都用对应模式。

相机位姿与 FOV **硬编码在仿真器里**：`simulator/src/util/camera.rs` 的
`view_for()`（每个场景一组 `position` / `target`）和 `FOV_DEGREES`。默认流程里
没人会调这些参数，放在文件里只会出现"改了没生效"（比如 `guided_dart` 的相机
其实绑在飞镖头上）和"FOV 写成 45° 导致图像与视觉端内参对不上、距离/PnP
系统性偏掉"。要挪相机就改 `view_for()` 后重新编译。

需要**外部相机预置**时（`Ultra_Vision/tools/sim/*.yaml` 就是这么用的：远/中/近
不同距离、实车相机高度、45° FOV），自定义配置文件里的 `fov_degrees:` 与当前场景
块的 `position:` / `target:` 仍会覆盖硬编码默认值，用 `DAEDALUS_CAMERA_CONFIG`
指向该文件即可。

如果只想改**相机到部件中心的距离**，在 `camera.yaml` 里加一项就行，不用改代码：

```yaml
scenario: outpost
distance: 8.0
```

相机沿水平方向被推到该距离（相对部件中心的高度差保持默认视角的），所以远距离时
俯仰角会自然变小。不写这一项就用默认距离；开机自报里会打印实际生效的
`camera_pos` / `camera_target` / `camera_distance`。

`guided_dart` 的相机参数硬编码在 `guided_dart/src/config.rs`；配置文件
`guided_dart/config/simulator.toml` 只留会拨的开关（靶运动模式、回推频率、
连打发数、飞行日志路径），发射参数仍在 `guided_dart/config/launch.json`。

### 前哨站场景

把 `scenario` 改成 `outpost` 即可。前哨站固定在**世界原点**，用的是官方整场
模型里抽出来的前哨站（`guided_dart/assets/models/outpost.glb`）：**塔体**用模型，
**装甲板**换成代码生成的那套（官方图标贴纸 + 发光灯条），板的**位置**取模型实测值。
只有相机需要在这个 `outpost` 块里调。仿真器会把模型底面贴地、旋转轴对齐原点，
并只让上面那一圈（三块高低错落的装甲板 + 支架 + 顶盖）转，塔座和立柱不动；默认
转速 `0.8π rad/s`，`DAEDALUS_OUTPOST_SPEED=0` 让它静止、`-2.5133` 反向。

装甲板的参数（板尺寸 `0.235 × 0.127 m`、离地 `1.114 / 1.216 / 1.318 m`、半径
`0.275 m`、pitch `−15°`）可以用 `DAEDALUS_OUTPOST_RADIUS` / `_HEIGHT_STEP` /
`_CENTER_HEIGHT` / `_ARMOR_PITCH` / `_SPEED` 覆盖；模型文件不在时，塔体回退到
程序化几何。

该场景导出 `dataset.csv`（每帧 3 行，`plate_id` 为 `0~2`）；按 `E` 或发送
`RESET` 会把旋转相位拨回 0°，便于逐轮复现。想换前哨站模型，用
`simulator/tools/extract_glb_subtree.py <整场模型.glb> <前哨站节点名> outpost.glb`
重新抽一份即可（细节见 `USAGE.md`）。

### 只看整场模型（base / rmuc2026）

`scenario: base` 加载 `base.glb`（25 赛季飞镖赛场），`scenario: rmuc2026`
加载 `rmuc2026.glb`（官方整场模型，摆位和飞镖场景一致）。两者都是"看模型"用
的，不做仿真逻辑，且**自带地面**（不会叠加通用地面）。这两个场景不锁相机：
鼠标转视角、`W/A/S/D` 平移、`F3` 切换视角模式。

模型从磁盘加载，路径是 `guided_dart/assets/models/`。如果模型放在别处，用
`DAEDALUS_ASSET_ROOT=<含 models/ 的目录>` 指定。

### 整车小陀螺

`armor_chassis` 场景下，整车小陀螺默认关闭，默认转速为 `5 rad/s`
（用 `DAEDALUS_SMALL_GYRO_SPEED=3.0` 可对齐实车常态的 1.5~3 rad/s，用
`DAEDALUS_SMALL_GYRO=1` 可让启动时就处于旋转状态）。进入模拟器后按 `R` 键
开启持续旋转，再按 `R` 键关闭。按 `E` 键可复位相机和机器人到启动状态。

### 接收端网络配置

文件：`example/config/simulator.yaml`

```yaml
host: 127.0.0.1
port: 7666
```

模拟器在其他机器上时，把 `host` 改为模拟器所在机器的 IP。

### FOV 与内参一致性

如果修改了 `fov_degrees`，并且接收端后续使用 PnP，需要同步更新相机内参：

```text
fx = fy = height / (2 * tan(fov_degrees / 2))
cx = width / 2
cy = height / 2
```

默认 640x480、45° FOV 对应：

```text
fx = fy ≈ 579.438
cx = 320
cy = 240
```

## 5. 联调测试

1. 先启动模拟器。
2. 再运行示例：

```bash
cd example
./build/sim_receiver_example ../config/simulator.yaml
```

3. 示例每 30 帧保存一张 `frame_XXXX.jpg`。
4. 如果示例立即退出，检查防火墙、IP 和端口。

Windows 防火墙放行 TCP `7666`：

```powershell
New-NetFirewallRule -DisplayName "Daedalus TCP 7666" -Direction Inbound -Protocol TCP -LocalPort 7666 -Action Allow
```

## 6. 故障排查

| 现象 | 处理 |
|---|---|
| 模拟器窗口黑屏 | 检查 GPU/驱动；VM 使用 `run_vm.sh` |
| `Unable to find a GPU!` | 安装 Mesa Vulkan 或正确显卡驱动 |
| 帧率只有个位数（日志提示"出帧只有 N fps"） | 先 `ls /dev/dri` 看有没有 GPU 设备。内核命令行带 `nomodeset` 时 iGPU 不会被绑定（`lspci -nnk` 里没有 `Kernel driver in use`），去掉它并 `sudo update-grub` 后重启即可；实在没有 GPU 就把仿真器放到有显卡的机器上跑 |
| 进程还在，但接收端**一帧都收不到** | 多半是跑仿真器那台机器的桌面**锁屏/黑屏**了（GNOME 锁屏后合成器停止呈现）。`loginctl unlock-session <id>` 解锁，并按 §2 关掉 `idle-delay` / 锁屏 |
| 用 `xvfb-run` 跑有 GPU 的机器时 abort：`No supported formats for surface` | Xvfb 不支持 DRI3，Intel 驱动没法呈现。改成跑在那台机器自己的 Wayland/X 会话里（见 §2「远程主机上跑」），或者干脆用虚拟机的软件渲染方案 |
| Linux 编译报 `wayland-client.pc not found` | 缺 `libwayland-dev`（还有 `libxkbcommon-dev` / `libasound2-dev`），见 §2「Linux 依赖」 |
| macOS 运行示例报 `Library not loaded: libprotobuf.*` | Homebrew 升级过 protobuf，而 opencv 的动态库还链着旧版（`brew reinstall opencv`）。本仓库的接收库只链 `core` / `imgcodecs`，正常情况下碰不到 dnn/protobuf |
| 示例连接失败 | 确认模拟器已启动、`host/port` 正确 |
| 示例收到空帧退出 | 检查防火墙、网络、模拟器是否停止推流 |
| 修改相机后视角没变 | 确认修改的是 `simulator/config/camera.yaml` 并重启 |
