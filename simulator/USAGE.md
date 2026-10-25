# 使用说明

## 1. 启动模拟器

### Windows

```bat
cd simulator
run_host.bat
```

PowerShell：

```powershell
cd simulator
.\run_host.ps1
```

### macOS / Linux

```bash
cd simulator
./run_host.sh
```

手动等价命令：

```bash
export CARGO_BUILD_JOBS=2
cargo build --release --no-default-features --features rune-only,tcp-only
target/release/daedalus
```

默认监听 `0.0.0.0:7666`，每个数据帧是 **40 字节大端帧头（版本 2）+ JPEG**。
裸跑 `target/release/daedalus` 时采集尺寸兜底为 `640x480`；启动脚本会显式设置
`DAEDALUS_CAPTURE_*`（`run_host.sh` 为 `1440x1080`）。
Windows、macOS 和 Linux 的启动脚本均固定使用 Release 构建，避免同时生成
Debug 与 Release 两套大型缓存。脚本默认使用双任务编译，并在检测到
`sccache` 时启用编译缓存；8 GiB 内存环境不建议超过 `2`，内存更大的环境
可通过 `CARGO_BUILD_JOBS=3` 或更高值缩短编译时间。

## 2. 编译并运行示例

```bash
cd example
cmake -S . -B build
cmake --build build -j4
./build/sim_receiver_example ../config/simulator.yaml
```

示例会持续接收帧，每 30 帧保存一张 `frame_XXXX.jpg`，收到 300 帧后自动退出。

如果模拟器不在本机，修改 `example/config/simulator.yaml`：

```yaml
host: 192.168.1.100
port: 7666
```

## 3. 验证 TCP 推流

模拟器运行后，在宿主机执行：

```bash
nc -w 3 127.0.0.1 7666 | xxd -l 40
```

应看到 `DAED` 帧头；紧跟的 4 字节版本号是 `00 00 00 02`（版本 1 的旧包头只有
32 字节，接收端两种都接受）。

## 4. 场景切换与相机

`simulator/config/camera.yaml` **只提供模式切换**，全文只有一项：

```yaml
scenario: armor_chassis
```

改这一行就决定仿真什么（`energy_rune` / `energy_small` / `energy_large` /
`armor_chassis` / `outpost` / `guided_dart` / `base` / `rmuc2026`），也可以用
环境变量 `DAEDALUS_SCENARIO` 临时覆盖。

能量机关有**两面背靠背**（`FACE_1` 小符/红方/顺时针，`FACE_2` 大符/蓝方/逆时针），
固定相机只能看到 `FACE_1` 那一侧。所以：

- `energy_rune`：模型原样，画面里是**小符**（大符在背面）
- `energy_small` / `energy_large`：两面都按该模式跑，画面里就是对应那种符

开机日志会打印两面的模式/队伍/旋向/位置，并标出"相机看到的是这面"。

相机位姿与 FOV **硬编码在仿真器里**（`simulator/src/util/camera.rs` 的
`view_for()` 与 `FOV_DEGREES = 41°`）：默认流程里没人会调这些参数，放在文件里
只会出现"改了没生效"和"FOV 写错导致图像与视觉端内参对不上"。需要挪相机就改
`view_for()`，重新编译（约 15 s）。

只想改**相机到部件中心的距离**时不用动代码，在 `camera.yaml` 里加一行即可：

```yaml
scenario: outpost
distance: 8.0        # 相机沿水平方向推到距部件中心 8 m（高度差保持默认视角的）
```

各场景默认距离：`energy_rune ≈ 4.9`、`armor_chassis ≈ 1.0`、`outpost ≈ 5.5`、
`rmuc2026 ≈ 10.4`、`base ≈ 21.4`。开机自报里会打印实际生效的
`camera_pos` / `camera_target` / `camera_distance`，可用来核对。

如果确实要用**外部相机预置**（`Ultra_Vision/tools/sim/*.yaml` 就是干这个的），
自定义配置文件里写 `fov_degrees:` 和当前场景块的 `position:` / `target:` 仍然会
覆盖默认值，用 `DAEDALUS_CAMERA_CONFIG` 指定该文件即可。

`DAEDALUS_RUNE_ONLY=1`（`run_host.sh` / `run_vm.sh` 默认设置）会锁定相机输入，
鼠标旋转和 `F3` 视角切换被禁用，相机的采集尺寸/帧率兜底默认值也切到
`640x480@30`。注意它**不再控制渲染对象**——渲染什么由 `scenario` 决定；
手工运行且需要自由视角时，把它设为 `0`。

`armor_chassis` 加载原始整车模型（含四个小装甲板）。

将 `scenario` 改为 `guided_dart` 可进入制导飞镖仿真。该场景使用绑定在飞镖头部的第一人称相机。飞镖的物理常数（气动舵效、舵机行程/速率、PID、镖体与场地摆位、相机参数）**硬编码在** `guided_dart/src/config.rs`；`guided_dart/config/simulator.toml` 只留会拨的开关（靶运动模式、回推频率、连打发数、飞行日志路径）；发射位置、初速度和发射角仍由 `guided_dart/config/launch.json` 提供。启用 TCP 推流后，接收端会收到飞镖相机画面。

飞镖场景按 `F` 发射，按 `R` 复位。

将 `scenario` 改为 `outpost` 可进入前哨站场景。前哨站固定在**世界原点**，用的
是官方整场模型里的前哨站：`guided_dart/assets/models/outpost.glb`，即
`rmuc2026.glb` 里 `001_ASM-2` 那棵子树，用
`simulator/tools/extract_glb_subtree.py` 抽出来（抽取脚本会烘进祖先变换、剔掉
SolidWorks 那些坐标是 1e38 的构造几何，并自检抽出来的顶点数据没有错位）。

模型加载后仿真器会对它做三件事：

1. **摆位**：Z 朝上的 CAD 坐标系转成 Y 朝上，底面贴地、旋转轴对齐世界原点；
2. **分层旋转**：底面离地 `≥ 1 m` 的部件（三块高低错落的装甲板 + 挂板支架 +
   顶盖）挂到一个"旋转轴"节点下跟着转，塔座、立柱、底盘留在原地——真机也是
   只有上面那一圈在转，整塔一起转是错的；
3. **换装甲板**：按 `1.114 / 1.216 / 1.318 m` 三个高度各认出模型上的一块板，
   把它的位置量下来，然后**把模型自带的板（连同它自己的灯条、销钉）删掉，
   换成代码生成的那套装甲板**：正面是官方前哨站图标贴纸、两侧是自发光红条，
   尺寸走 `DAEDALUS_OUTPOST_*` 那套参数（`0.235 × 0.127 m`）。板子放在模型
   实测的板心位置上、按"局部 −Z 朝外 + 配置的板面俯仰角"摆正，节点名仍是
   `ARMOR_1..3` / `ARMOR_<n>_L`，所以 `dataset.csv`（每帧 3 行）和
   `light_bars.csv`（每帧 6 行）照常工作——视觉端的标定尺寸和灯条也就能对上。

实测模型上三块板离地 `1.116 / 1.216 / 1.316 m`、绕轴半径 `0.26~0.30 m`、
互成约 120°，和实车数据（1114 / 1216 / 1318 mm）对得上。默认转速
`0.8π rad/s`，`DAEDALUS_OUTPOST_SPEED` 可覆盖。前哨站场景下 `Space` 发射、
`E` 复位（复位会把旋转相位拨回 0°，便于逐轮可复现）。

把 `outpost.glb` 挪走（或删掉）时，场景会**自动回退到程序化几何**：塔座 +
旋转头 + 三块大装甲板，几何参数由 `DAEDALUS_OUTPOST_*` 决定（见 `API.md` 的
前哨站参数表）；除了塔身是建模出来的，装甲板那一套和模型版完全一样。

把 `scenario` 改为 `base` 或 `rmuc2026` 可以**只看整场模型**（不做仿真逻辑）：

- `base` = `base.glb`，25 赛季的飞镖赛场（整场约 28×15 m）。
- `rmuc2026` = `rmuc2026.glb`，官方整场模型，摆位与飞镖场景一致（基地靶在
  `(−3.27, 1.07, −0.77)`，飞镖发射井在 `(0, 0.02, −25.8)`）。

这两个场景**自带地面**，所以不会再叠加通用地面（否则两块地板会互相穿插、
把基地挡住）。它们也**不锁相机输入**：鼠标转动视角、`W/A/S/D` 平移、
`F3` 切换跟随模式，方便绕着模型看。模型放在 `guided_dart/assets/models/`
下走磁盘加载（`DAEDALUS_ASSET_ROOT` 可指定资源根目录）。

整车小陀螺默认关闭，进入模拟器后按 `R` 键开启/关闭持续旋转，默认转速为
`5 rad/s`（`DAEDALUS_SMALL_GYRO_SPEED=3.0` 可对齐实车常态的 1.5~3 rad/s）。

按 `E` 键可复位模拟器：相机回到配置文件定义的初始朝向，机器人回到启动位置，底盘/云台角度和运动速度清零，小陀螺关闭。

整车支持相对相机移动：`W` 朝相机移动，`S` 远离相机，`A` 朝相机左侧移动，`D` 朝相机右侧移动。

按 `Space` 从相机位置发射弹丸，弹丸受重力、阻尼与碰撞物理影响。

`armor_chassis` 场景运行时会在宿主机生成 `dataset.csv`，记录四块装甲板的相机坐标、颜色、距离和时间。可通过 `DAEDALUS_CSV_PATH` 修改导出路径。

接收端控制接口默认监听 `0.0.0.0:7667`。发送 `FIRE` 可触发飞镖发射；飞镖模式还接受 `DART <pitch> <yaw> <roll>`（推荐，视觉端控制律的正式输出）、`SERVO`（同义）、`GIMBAL <yaw> <pitch>`（手动调试）和遗留的 `szu_uart` 7 字节二进制帧。角度单位均为弧度，直接写进舵面期望角，再经过 `guided_dart/src/config.rs` 里的 PID 控制层与舵机限幅。停止发送超过 `command_timeout_secs`（默认 0.3 s）后舵面回中立——仿真器内部没有制导律，镖按纯弹道飞。仿真器会通过同一条连接按 `state_telemetry_hz` 回推 `STATE` 状态行供视觉端解算。

装甲板/能量机关场景还可以发送 `GIMBAL_VISION <state> <yaw> <pitch>`，按 `wheel_leg_2` 的纯视觉接管策略渐进跟随视觉目标角；`state=0` 时保持当前角度，`state=1/2` 时接管。需要让旧的 `GIMBAL <yaw> <pitch>` 也走该策略时，启动前设置 `DAEDALUS_GIMBAL_MODE=vision`。

如果使用默认 `szu_uart`，仿真器按 640x480 图像中心将 blob 中心偏移转换为舵面角；若 TCP 图像尺寸不同，请同步设置 `DAEDALUS_CAPTURE_WIDTH` 和 `DAEDALUS_CAPTURE_HEIGHT`。

**采集尺寸必须和视觉端的标定一致**（`DAEDALUS_CAPTURE_*`，不是窗口大小 —— 窗口只是预览，
按比例显示）。`run_host.sh` / `run_vm.sh` 默认已经改成 **1440x1080**：Ultra_Vision 用的
海康相机是 1440x1080 / BayerRG8，内参（f≈1447）、装甲板等效尺寸、ROI 阶梯都是按这个
尺寸标定的。给 640x480 的话，7.5 m 处靶面会从 ~26 px 掉到 ~11 px（面积差 2.25 倍），
检测率和 PnP 尺度都会和标定对不上；视觉端启动时会打印
`[input] <宽>x<高> fov=.. f=.. cx=.. cy=..`，尺寸小于标定宽度时会直接告警。
软件渲染（VM 的 `run_vm.sh`）扛不住 30 fps 就调 `DAEDALUS_CAPTURE_FPS`，
**不要**去改 `DAEDALUS_CAPTURE_WIDTH/HEIGHT` 来换帧率。

### 两台主机之间的带宽优化

模拟器和运算端分开运行时，数据流带宽由
`分辨率 × JPEG 质量 × 帧率`共同决定。`1440x1080 + JPEG 95 + 30 FPS`
很容易占满百兆以太网或普通无线链路，表现为接收端只有个位数 FPS。优先使用
以下脚本启动模拟器：

```bash
cd simulator
./run_remote.sh
```

它保持视觉端标定需要的 `1440x1080`，默认使用 JPEG 80 和 20 FPS，并设置
`DAEDALUS_PREVIEW=0` 关闭重复窗口和主相机，只保留 TCP 离屏采集。需要临时
查看模拟器画面时设置 `DAEDALUS_PREVIEW=1`；需要调整链路参数时：

```bash
DAEDALUS_JPEG_QUALITY=70 DAEDALUS_CAPTURE_FPS=15 ./run_remote.sh
```

仿真器跑在**没接显示器的远程主机（NUC / 服务器）**上时用 `run_nuc.sh`：它负责
挑好那台机器自己的图形会话（Wayland 的 `wayland-0` 或 X11 的 `:0`，**不要**用
`xvfb-run`——Xvfb 没有 DRI3，Intel 驱动在它上面无法呈现、会直接 abort），检查
GPU 是否绑定、桌面会话有没有锁屏（GNOME 锁屏后不再合成画面，接收端会一帧都
收不到），然后按 `run_remote.sh` 的参数启动：

```bash
cd simulator
./run_nuc.sh -d       # 后台跑，日志写 nuc_sim.log，打印 PID 与停止命令
```

`run_nuc.sh` 默认还关掉**远端云台控制**（`DAEDALUS_GIMBAL_CONTROL=0`）：视觉端发来的
`GIMBAL` / `GIMBAL_VISION` 被忽略，相机固定在 `camera.yaml` 的开机位姿，
图像包头里的姿态一直是 0（`FIRE` / `RESET` 仍然生效）；需要视觉端控云台时用
`DAEDALUS_GIMBAL_CONTROL=1 ./run_nuc.sh`。

建议同时用 `iperf3` 测试两台主机之间的实际吞吐，而不是只看路由器标称速率。
模拟器会输出 `[tcp] clients=... encoded_fps=... bandwidth=...`，可与接收端
实测 FPS 对照定位是链路、编码还是运算端瓶颈。

### UDP 低延迟模式（仿真器跑在远程 NUC 上时用）

跨机跑（尤其是无线或百兆链路）时 TCP 的队头阻塞/重传会让延迟和抖动都变差，
可以再开一路 UDP 图像流——TCP 那份不受影响，两边可以并存：

```bash
cd simulator
./run_host.sh                          # 记得先把 camera.yaml 的 protocol 改成 udp
```

改完**要重启模拟器**（`protocol:` 只在启动时读一次），然后看启动横幅：
`protocol=udp+tcp（UDP 监听 0.0.0.0:7669…）` 才算切过去了；日志里继续出现
`[tcp] clients=...` 是正常的——UDP 是和 TCP 并存的那一路。接收端也要改成 udp
（示例接收库会每秒自动续订），可以用 `simulator/tools/udp_grab.py <host>` 先验证
仿真器这侧是不是真的在推 UDP。

也就是把 `simulator/config/camera.yaml` 改成：

```yaml
scenario: armor_chassis
protocol: udp          # tcp（默认）/ udp；udp = 在 TCP 之外再开一路，默认端口 7669
```

接收端（`example/config/simulator.yaml`）在配置里写 `protocol: udp`（可选
`udp_port`，默认 `7669`）即可；示例接收库
每秒会自动发一次订阅，不用手工做什么。要点：

- 图像走 UDP、**指令仍走 TCP 7667**（`FIRE` / `GIMBAL` / `DART` 必须可靠）；
- 丢包表现为**掉帧**而不是卡顿：缺片的帧直接丢，并按时间戳丢掉过期帧；
- 带宽和 TCP 是同一量级（1440x1080 / JPEG 80 / 15 FPS ≈ 10 Mbit/s），链路吃紧
  时同样要降 `DAEDALUS_JPEG_QUALITY` 或 `DAEDALUS_CAPTURE_FPS`；
- 仿真器日志里的 `udp_subs=` / `udp_dgram=` 可以看订阅数与每秒报文数。

临时做 A/B 对比时也可以不改文件，直接用 `DAEDALUS_UDP=1 ./run_host.sh` 覆盖。

协议细节（分片头字段、订阅规则）见 `API.md` §1.1。

### 数据集 CSV 格式

CSV 表头：

```csv
x,y,z,color,distance,time_us,plate_id
```

字段含义：

- `x/y/z`：装甲板在相机坐标系中的三维坐标，单位米。其中 `x` 向右、`y` 向下、`z` 为相机前方。
- `color`：当前固定为 `0`，表示红色。
- `distance`：装甲板到相机的距离，单位米。
- `time_us`：Unix 微秒时间戳。
- `plate_id`：同一底盘中的装甲板编号，范围 `0~3`。

同一帧会导出四行数据，它们拥有相同的 `time_us`，可以按 `time_us` 分组还原一个完整的四装甲板底盘快照。

同一场景还会输出 `light_bars.csv`，默认路径可通过
`DAEDALUS_LIGHT_CSV_PATH` 修改。文件表头为：

```csv
x,y,z,time_us,bar_id,plate_id,x0,y0,z0,x1,y1,z1
```

每个装甲板包含两个发光条，因此每个时间戳会输出 8 行。`bar_id` 为 `0~7`，
`plate_id` 为 `0~3`，`x/y/z` 同样采用相机坐标系；末尾的 `x0/y0/z0` 与
`x1/y1/z1` 是同一根灯条两端点，可直接算灯条长轴方向。灯条数量是**自适应**的：
整车 4 板每帧 8 行、前哨站 3 板每帧 6 行，换模型不用改代码。

注意两份 CSV 的 `time_us` **时基不同**：`dataset.csv` 是 Unix 微秒时间戳，
而 `light_bars.csv` 是仿真固定步累计的微秒（从 0 开始，与刚体旋转相位严格
同步）。按 `time_us` 分组还原同一帧都成立，但两者不能互相直接对齐。

```bash
# 画图
python3 plot_results.py dataset.csv

# 只看某一块装甲板
python3 plot_results.py dataset.csv --select 0
```

## 5. 常见问题

- 示例返回空帧：检查 `host` 是否填对、模拟器是否已启动、防火墙是否放行 `7666`。
- VM 无 GPU：使用 `simulator/run_vm.sh` 启用软件 Vulkan。
- 修改 FOV：接收端如果做 PnP，需要同步更新相机内参。
