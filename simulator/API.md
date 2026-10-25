# 接口文档

## 1. TCP 图像流

模拟器作为 TCP Server，默认监听：

```text
0.0.0.0:7666
```

每个数据帧由 **40 字节大端帧头 + JPEG 数据**组成。帧头分两段：前 32 字节是
图像必需字段，后 8 字节（版本 2 起）是执行之后的真实云台姿态。接收端必须把
这 8 字节读完再按 `payload_len` 读 JPEG，否则每帧会错位 8 字节、解码全部失败。

两台主机联调时建议使用 `simulator/run_remote.sh`。它保持默认 `1440x1080`
标定尺寸，并以 JPEG 80、20 FPS 启动，降低百兆链路和无线网络的队列压力。

| 偏移 | 长度 | 字段 | 说明 |
|---|---:|---|---|
| 0 | 4 | `magic` | 固定为 `"DAED"` |
| 4 | 4 | `version` | 协议版本，当前为 `2`（`1` = 无云台姿态的旧 32 字节包头） |
| 8 | 4 | `width` | 图像宽度 |
| 12 | 4 | `height` | 图像高度 |
| 16 | 4 | `payload_len` | JPEG 负载字节数 |
| 20 | 8 | `timestamp_us` | Unix 微秒时间戳 |
| 28 | 4 | `seq` | 帧序号 |
| 32 | 4 | `gimbal_yaw` | 执行之后的真实云台水平角（`f32`，弧度，仅在 `version >= 2` 时存在） |
| 36 | 4 | `gimbal_pitch` | 执行之后的真实云台俯仰角（`f32`，弧度，仅在 `version >= 2` 时存在） |

JPEG 负载通过 OpenCV 解码后为 BGR 图像。旧的 32 字节包头仍然被接收端接受，
此时两个云台字段按 `0` 处理。

`gimbal_yaw` / `gimbal_pitch` 是**指令执行之后**的相机姿态，等价实车的 IMU
反馈。视觉端以前只能把"下发指令角"当真实姿态，拿到这个字段才能像实车一样
用真实姿态做闭环。

### 1.1 UDP 图像流（可选，低延迟）

仿真器跑在远程主机（如 NUC）时，TCP 的队头阻塞与重传会让延迟和抖动都变差。
在 `simulator/config/camera.yaml` 里把协议选成 UDP，即可**同时**开一路 UDP 图像流
（TCP 那一份不受影响，两者可并存）：

```yaml
scenario: armor_chassis
protocol: udp
```

默认监听 `0.0.0.0:7669`；端口用环境变量 `DAEDALUS_UDP_ADDR` 改，也可以用
`DAEDALUS_UDP=1` 临时把协议覆盖成 udp（方便脚本做对比测试）。

> ⚠️ `protocol:` **只在启动时读一次**，改完必须重启模拟器才生效；启动横幅里会
> 打印 `protocol=tcp` 或 `protocol=udp+tcp（UDP 监听 0.0.0.0:7669…）`，以它为准。
> 又因为 UDP 是和 TCP **并存**的，日志里继续出现 `[tcp] clients=...` 不代表没切
> 过去——那条是 TCP 那一路的统计。**接收端也要一起改成 udp**（并且先发订阅报文），
> 否则它仍然走 TCP，延迟不会有任何变化。
> 现成的排查/验证工具：`simulator/tools/udp_grab.py <host> [port] [秒] [存图.jpg]`。

- **订阅制**：接收端先往该端口发任意一个小报文（例如 3 字节 `SUB`），仿真器只向
  "订阅过的地址"推流；订阅有效期 3 秒，接收端要周期性续期（示例接收库每秒发一次）。
  这样仿真器侧不用配接收端 IP，接收端掉线也不会一直往黑洞里发。
- **分片**：一帧 JPEG 有 100~300 KB，超过 UDP 单包上限，所以按 1400 字节切片，
  每片带 40 字节头，整个报文 ≤ 1460 字节，不会触发 IP 分片。
- **丢包处理**：缺片的帧直接丢掉，只交付完整帧；同时按 `timestamp_us` 丢掉比已
  交付帧更旧的帧。视觉端本来只要"最新一帧"，所以丢包表现为掉帧而不是卡顿。
- 指令（`FIRE` / `GIMBAL` / `DART`）**仍然走 TCP 7667**，可靠且量小。

每个报文的头（40 字节，大端）：

| 偏移 | 长度 | 字段 |
|---|---:|---|
| 0 | 4 | `magic`，`"DAED"` |
| 4 | 2 | `version`，当前为 `1` |
| 6 | 2 | `header_len`，当前为 `40`（留给以后扩展） |
| 8 | 4 | `frame_id`，帧序号 |
| 12 | 8 | `timestamp_us`，这一帧的曝光时刻（Unix 微秒，与 TCP 包头同一个钟） |
| 20 | 4 | `total_len`，整帧 JPEG 字节数 |
| 24 | 4 | `offset`，本片在整帧中的字节偏移 |
| 28 | 4 | `payload_len`，本片负载字节数 |
| 32 | 4 | `gimbal_yaw`（`f32`，弧度，指令执行后的真实姿态） |
| 36 | 4 | `gimbal_pitch`（`f32`，弧度） |

接收端现成实现见 `example/sim_receiver/src/udp_input.cpp`：配置里写
`protocol: udp` 即启用（可选 `udp_port`，默认 `7669`）。

## 2. 模拟器相机配置

文件：`simulator/config/camera.yaml` —— **只提供模式切换**，全文只有一项：

```yaml
scenario: armor_chassis
```

相机位姿与 FOV **硬编码在仿真器里**（`simulator/src/util/camera.rs` 的
`view_for()` 与 `FOV_DEGREES`）：

- 这些参数没人会去调，放在配置文件里只会造成"改了没生效"（例如 `guided_dart`
  的相机其实绑在飞镖头上）；
- FOV 固定 41°：视觉端内参按 `1440x1080 / 41°`（`f≈1447`）标定，写成 45° 会让
  图像里的靶面像素与内参对不上，距离/PnP 系统性偏掉一个比例。

要调视角请改 `view_for()`，重新编译即可。

另外可以用一项 `distance:`（米）只调**相机到部件中心**（也就是相机看向的那个点）
的距离，不必写绝对坐标：

```yaml
scenario: outpost
distance: 8.0
```

它把相机沿水平方向推到指定距离，**保留相对部件中心的高度差**（所以远距离时俯仰角
自然变小，和实车/相机预置的做法一致）。各场景默认距离：`energy_rune ≈ 4.9`、
`armor_chassis ≈ 1.0`、`outpost ≈ 5.5`、`rmuc2026 ≈ 10.4`、`base ≈ 21.4`；
不写这一项就用默认距离。开机自报会打印实际生效的
`camera_pos` / `camera_target` / `camera_distance`，可直接核对。

`protocol:` 同样是全局键（`tcp` 默认 / `udp`），见 §1.1。

覆盖能力仍然保留：**自定义**配置文件里写了 `fov_degrees:` 或当前场景块的
`position:` / `target:` 时，会盖掉上面的硬编码默认值——`Ultra_Vision/tools/sim/*.yaml`
的相机预置（远/中/近、实车高度，其中 rune 预置用的是 45° FOV）就是靠这个跑的，
路径由 `DAEDALUS_CAMERA_CONFIG` 指定。

`scenario` 支持：

- `energy_rune` / `energy_small` / `energy_large`：仿真能量机关，三种模式共用一个
  场景与同一个相机视角，区别只在两面的符：

  | 场景 | 两面 | 相机看到的那面 |
  |---|---|---|
  | `energy_rune` | `FACE_1` 小符 + `FACE_2` 大符（模型原样） | 小符 |
  | `energy_small` | 两面都当**小符**跑 | 小符 |
  | `energy_large` | 两面都当**大符**跑 | 大符 |

  模型里是**两面背靠背**的机关：`FACE_1`（红方、小符、顺时针）与 `FACE_2`
  （蓝方、大符、逆时针），固定相机在 −X−Z 一侧只能看到 `FACE_1`。所以想测大符
  必须让两面同符（`energy_large`），否则画面里永远是小符。两面各按自己的模式
  独立跑：小符每轮 1 个靶、固定转速；大符每轮 2 个靶、激活时变速。
  开机日志会打印两面的模式/队伍/旋向/位置，并标出"相机看到的是这面"。
- `armor_chassis`：仿真原始整车模型，使用 `armor_chassis` 相机坐标。
- `outpost`：仿真前哨站。前哨站固定在**世界原点**，用的是从 `rmuc2026.glb`
  抽出来的官方前哨站模型（`guided_dart/assets/models/outpost.glb`）：**塔体**
  用它，**装甲板**换成代码生成的那套（官方图标贴纸 + 发光灯条）。只有上面那一圈
  （三块高低错落的装甲板 + 支架 + 顶盖）绕轴旋转，塔座/立柱/底盘固定。模型文件
  缺失时塔体回退到程序化几何。
- `base` / `rmuc2026`：**场地查看场景**，把整场模型直接加载出来看（不做任何仿真逻辑）。
  两者都自带地面，所以不会再叠加通用地面。`rmuc2026` 的摆位与飞镖场景完全一致。
  这两个场景刻意**不锁相机输入**（见下文的 `DAEDALUS_RUNE_ONLY`），鼠标即可转视角。
- `guided_dart`：进入制导飞镖仿真。相机是绑在飞镖头部的第一人称相机，参数在
  `guided_dart/src/config.rs`（硬编码）；`guided_dart/config/simulator.toml`
  只留靶运动模式与回推/日志开关。

也可以用 `DAEDALUS_SCENARIO` 覆盖文件里的 `scenario`。

`outpost` 场景的几何参数（**装甲板那套一直起作用**——模型版的板子就是用这套
尺寸生成的；只有塔体的程序化回退会忽略它们。都可用环境变量覆盖）：

| 参数 | 默认值 | 说明 |
|---|---:|---|
| `DAEDALUS_OUTPOST_RADIUS` | `0.275` | 装甲板到旋转轴的距离（米） |
| `DAEDALUS_OUTPOST_HEIGHT_STEP` | `0.102` | 相邻两块板的高度差（米） |
| `DAEDALUS_OUTPOST_CENTER_HEIGHT` | `1.216` | **中间那块**板离地高度（米），三块板为 1.114 / 1.216 / 1.318 m |
| `DAEDALUS_OUTPOST_ARMOR_PITCH` | `-15` | 板面俯仰角（度） |
| `DAEDALUS_OUTPOST_SPEED` | `0.8π` | 旋转角速度（rad/s），负值反向 |

装甲板按 RMUC 大装甲板尺寸建模：`0.235 × 0.127 m`、板厚 12 mm，两块灯条各
`0.012 × 0.056 m`。**模型版**只从模型上取板的**位置**（实测离地
`1.116 / 1.216 / 1.316 m`、半径 `0.26~0.30 m`、互成约 120°），板子本身还是
上面这套程序化几何；上面那三个高度参数就用来"对号入座"地认出这三块板。两种
方式下三块板的节点名都是 `ARMOR_1..3`（灯条在 `ARMOR_<n>_L` 下），因此命中判定、
`dataset.csv` 导出（每帧 3 行，`plate_id` 为 `0~2`）、`light_bars.csv`
（每帧 6 行）与遥测都复用整车场景那一套。

装甲板正面贴的是官方前哨站图标贴纸，素材在
`simulator/src/assets/outpost_armor_plate.png`（随二进制内嵌）。这张贴图**已经
按板面宽高比（0.235 : 0.127）排版**：图标居中、按板高等比缩放、底色与板面
一致，所以直接用 `Cuboid` 自带 UV 贴上去即可，不需要 `uv_transform`。换图标时
保持同样的排版规则（否则会被拉伸变形）。灯条是单独的发光网格，不是贴图的一部分。

图标**只贴在正面**（局部 −Z，朝外那一面）：`armor_plate_mesh()` 会把背面和四条
窄边的 UV 全部指到贴图左上角的纯底色上。直接用 `Cuboid::mesh()` 会给六个面都
铺同一张图，从背面看就是一张镜像倒过来的贴纸（真机装甲板背面是纯黑的）。
`outpost::tests::armor_plate_sticker_only_on_the_front_face` 盯着这个行为。

整车小陀螺默认关闭，按 `R` 键开启后底盘持续绕 Y 轴旋转，再次按 `R` 关闭。
默认转速为 `5 rad/s`，可用环境变量 `DAEDALUS_SMALL_GYRO_SPEED` 覆盖
（例如 `DAEDALUS_SMALL_GYRO_SPEED=3.0` 对齐实车常态的 1.5~3 rad/s）。
也可以用 `DAEDALUS_SMALL_GYRO=1` 让整车启动时就处于小陀螺状态。

按 `E` 键可将相机朝向、整车位置和朝向、底盘/云台角度及运动速度恢复到模拟器启动状态，同时关闭小陀螺。

整车移动使用 `W/A/S/D`，方向相对固定相机：`W` 朝相机，`S` 远离相机，`A/D` 分别为相机左/右侧。

`armor_chassis` 场景默认在宿主机导出 `dataset.csv`，详细字段说明见“CSV 数据集导出格式”。

## 3. CSV 数据集导出格式

默认输出文件为运行目录下的 `dataset.csv`，可通过 `DAEDALUS_CSV_PATH` 修改路径。文件表头为：

```csv
x,y,z,color,distance,time_us,plate_id
```

每一行代表一块装甲板。每个相机帧会导出四行，`plate_id` 为 `0~3`，对应同一底盘中四块不同位置的装甲板。

### 字段说明

| 字段 | 单位/取值 | 说明 |
|---|---|---|
| `x` | 米 | 装甲板在相机坐标系中的 X 坐标，向右为正 |
| `y` | 米 | 装甲板在相机坐标系中的 Y 坐标，向下为正 |
| `z` | 米 | 装甲板在相机坐标系中的 Z 坐标，向前为正 |
| `color` | 整数 | 当前版本固定为 `0`，表示红色 |
| `distance` | 米 | 装甲板位置到相机光心的欧氏距离 |
| `time_us` | 微秒 | Unix 时间戳，同一帧的四行数据相同 |
| `plate_id` | 整数 | 装甲板编号，范围 `0~3` |

### 坐标方向

坐标以主相机为参考，方向定义如下：

```text
X：相机右侧
Y：相机下方
Z：相机前方
```

原始导出数据不是图像像素坐标，而是相机坐标系下的三维坐标，单位统一为米。

### 示例

```csv
x,y,z,color,distance,time_us,plate_id
0.235000,-0.050000,1.200000,0,1.226000,1788424718000000,0
-0.235000,-0.050000,1.200000,0,1.226000,1788424718000000,1
0.010000,0.120000,0.900000,0,1.010000,1788424718000000,2
-0.010000,0.120000,0.900000,0,1.010000,1788424718000000,3
```

示例中同一 `time_us` 的四行数据代表同一个底盘快照，可按 `time_us` 分组还原一个“四块装甲板单元”。

### 绘图脚本

`plot_results.py` 可直接读取该 CSV：

```bash
python3 plot_results.py dataset.csv
```

也可只查看某块装甲板：

```bash
python3 plot_results.py dataset.csv --select 0
```

同一场景还会默认生成 `light_bars.csv`，可通过 `DAEDALUS_LIGHT_CSV_PATH`
修改路径。该文件把每块装甲板上的发光条拆成独立灯条，**灯条数量是自适应的**
（整车 4 板 × 2 条 = 每帧 8 行，前哨站 3 板 × 2 条 = 每帧 6 行）：

```csv
x,y,z,time_us,bar_id,plate_id,x0,y0,z0,x1,y1,z1
0.180000,-0.020000,1.200000,123456789,0,0,0.175000,-0.020000,1.200000,0.185000,-0.020000,1.200000
0.220000,-0.020000,1.200000,123456789,1,0,0.215000,-0.020000,1.200000,0.225000,-0.020000,1.200000
```

`x/y/z` 仍为相机坐标系下的位置，单位为米；同一帧的所有行拥有相同的
`time_us`。`bar_id = plate_id × 2 + k`（`k` 为该板上的第几根灯条），
同一 `plate_id` 对应同一装甲板上的两个灯条。表头里的 `x0/y0/z0` 与
`x1/y1/z1` 是同一根灯条**两端点**在相机坐标系下的位置，可直接算灯条的
长轴方向与像素长度。

收集器按节点名 `ARMOR_<n>_L` 找装甲板的灯条，并同时支持两种导出习惯：
① 该节点自带一个网格、网格里含两根灯条（官方整车模型就是这样）；② 该节点是
一层空节点、两根灯条各是一个子网格（程序化生成的前哨站就是这样）。因此将来
换模型不必改代码。

### 两份 CSV 的 `time_us` 时基不同

- `dataset.csv`（`export_armor_csv`）：`SystemTime::now()` 的 **Unix 微秒**时间戳。
- `light_bars.csv`（`export_light_bars_csv`）：`Time<Fixed>::elapsed()` 的
  **仿真相对微秒**，从 0 开始。刚体旋转在固定物理步上积分，用挂钟计时会把
  渲染帧抖动写进时间戳，使 θ(t) 相对记录时间非均匀；因此这里刻意改用固定步
  时长，它与旋转相位严格同步。按 `time_us` 分组还原同一帧仍然成立，但**不能**
  拿它和 `dataset.csv` 的时间戳直接对齐。

## 4. 接收端配置

文件：`example/config/simulator.yaml`

```yaml
host: 127.0.0.1
port: 7666
```

`host` 填模拟器所在宿主机 IP，`port` 填模拟器 TCP 监听端口。

## 5. 环境变量

| 环境变量 | 默认值 | 说明 |
|---|---|---|
| `DAEDALUS_SCENARIO` | 取 `camera.yaml` 的 `scenario` | 覆盖场景：`energy_rune` / `armor_chassis` / `guided_dart` |
| `DAEDALUS_ASSET_ROOT` | 自动探测 | 磁盘资源根目录（含 `models/`）。默认依次找 `assets/`、可执行文件向上的 `assets/`、`guided_dart/assets/` |
| `DAEDALUS_RUNE_ONLY` | 启动脚本设为 `1` | **不再影响渲染**。现在只影响两件事：TCP 采集尺寸/帧率的兜底默认值，以及锁定相机输入（禁用 `F3` 视角切换与鼠标旋转）。渲染对象由 `scenario` 决定；`base` / `rmuc2026` 场地查看场景始终不锁 |
| `DAEDALUS_WINDOW_WIDTH` | `640`（`run_host.sh` 为 `960`） | 窗口宽度（仅预览） |
| `DAEDALUS_WINDOW_HEIGHT` | `480`（`run_host.sh` 为 `720`） | 窗口高度（仅预览） |
| `DAEDALUS_CAPTURE_WIDTH` | `640`（`rune-only` 关闭时 `1440`） | TCP 图像宽度，必须与视觉端标定一致 |
| `DAEDALUS_CAPTURE_HEIGHT` | `480`（`rune-only` 关闭时 `1080`） | TCP 图像高度，必须与视觉端标定一致 |
| `DAEDALUS_CAPTURE_FPS` | `30`（`rune-only` 关闭时 `60`） | TCP 推流帧率上限 |
| `DAEDALUS_JPEG_QUALITY` | `95` | JPEG 压缩质量 |
| `DAEDALUS_TCP_ADDR` | `0.0.0.0:7666` | TCP 监听地址 |
| `DAEDALUS_UDP` | `0` | `1` = 临时把协议覆盖成 udp（等价于 `camera.yaml` 里写 `protocol: udp`） |
| `DAEDALUS_UDP_ADDR` | `0.0.0.0:7669` | UDP 图像流监听地址 |
| `DAEDALUS_COMMAND_ADDR` | `0.0.0.0:7667` | 接收端控制指令监听地址 |
| `DAEDALUS_TELEMETRY_ADDR` | `0.0.0.0:7668` | 能量机关 `RUNE` 事件遥测监听地址 |
| `DAEDALUS_CAMERA_CONFIG` | `config/camera.yaml` | 相机配置路径 |
| `DAEDALUS_CSV_PATH` | `dataset.csv` | 宿主机装甲板 CSV 导出路径 |
| `DAEDALUS_LIGHT_CSV_PATH` | `light_bars.csv` | 宿主机 8 灯条 CSV 导出路径 |
| `DAEDALUS_PREVIEW` | `1` | `0` = 隐藏窗口并停用窗口相机，只保留 TCP 离屏采集相机 |
| `DAEDALUS_FPS_PROBE` | 未设置 | 设了就每秒打印一次「渲染帧率」（采集/编码帧率见 `[tcp]` 统计行），用来区分卡在渲染还是采集流水线 |
| `DAEDALUS_SMALL_GYRO` | `0` | 启动时是否直接开启整车小陀螺 |
| `DAEDALUS_SMALL_GYRO_SPEED` | `5.0` | 小陀螺转速（rad/s） |
| `DAEDALUS_MOVE_LOOP` | 未设置 | 循环平移靶：`"持续秒,vx,vy; ..."`，时间为相对仿真启动 |
| `DAEDALUS_MOVE_SCHEDULE` | 未设置 | 绝对时间表平移靶：`"秒: vx,vy; ..."` |
| `DAEDALUS_GIMBAL_MODE` | `vision` | 旧的 `GIMBAL <yaw> <pitch>` 走 `vision`（限速跟随）还是 `direct`（瞬时到位） |
| `DAEDALUS_GIMBAL_CONTROL` | `1` | 远端云台控制总开关：`0` 时 `GIMBAL` / `GIMBAL_VISION` 一律忽略，相机固定在 `camera.yaml` 的开机位姿（包头里的姿态始终为 0）。`run_nuc.sh` 在远程主机上默认设成 `0` |
| `DAEDALUS_GIMBAL_YAW_DEADBAND` | `0.002` | 纯视觉接管 yaw 死区（rad） |
| `DAEDALUS_GIMBAL_YAW_MIDDLE` | `0.5` | yaw 粗/细调分界（rad） |
| `DAEDALUS_GIMBAL_YAW_FINE_RATE` / `_COARSE_RATE` | `3.0` | yaw 细调/粗调最大角速度（rad/s） |
| `DAEDALUS_GIMBAL_PITCH_DEADBAND` | `0.003` | pitch 死区（rad） |
| `DAEDALUS_GIMBAL_PITCH_RATE` | `3.0` | pitch 最大角速度（rad/s） |
| `DAEDALUS_GIMBAL_FINE_GAIN` | `1.0` | 死区内的比例收敛增益 |
| `DAEDALUS_GIMBAL_PITCH_LIMIT` / `_MIN` / `_MAX` | `1.3963`（±80°） | pitch 软限位（rad），实车为非对称 `[-0.49, +0.27]` |
| `DAEDALUS_FIELD_CROP` | `1` | 飞镖场景是否裁剪官方完整场地地面，只保留发射仓到基地的走廊 |
| `DAEDALUS_OUTPOST_*` | 见上文前哨站参数表 | 前哨站半径 / 高度步长 / 中心高度 / 板面俯仰 / 转速 |
| `DAEDALUS_TARGET_MOTION_MODE` | 取 `simulator.toml` | 飞镖靶运动模式：`fixed` / `random_fixed` / `random_moving` / `end_moving` |
| `DAEDALUS_AUTO_SHOT_AFTER_SECS` | 未设置 | 启动 N 秒后自动开始连打（等价按 `T`），用于无人值守回归 |
| `DAEDALUS_RECORD_DIR` / `DAEDALUS_RECORD_EVERY` | 未设置 | 把窗口画面按帧存成 PNG 序列（每 N 帧一张） |
| `WGPU_FORCE_FALLBACK_ADAPTER` | `0` | `1` = 强制软件 Vulkan 适配器（VM 无 GPU 时用） |

## 6. 控制指令接口

模拟器默认在 `0.0.0.0:7667` 监听接收端发回的文本和二进制控制指令。图像流和控制指令使用不同端口。

> **`DAEDALUS_GIMBAL_CONTROL=0` 时下面的云台指令全部被忽略**（相机固定不动、
> `FIRE` / `RESET` 照常生效）。远程主机（NUC）上只想要一个稳定图像源时用得上，
> `simulator/run_nuc.sh` 默认就设成 `0`；要恢复视觉端控云台用
> `DAEDALUS_GIMBAL_CONTROL=1 ./run_nuc.sh`。
`VisionDateRecevier` 会保持一条独立的长连接发送控制指令，避免每帧执行
`connect/send/close` 带来的额外往返和连接风暴；连接被路由器重置后会自动重连。

飞镖模式的设计原则是**仿真器只做物理**：它识别 `DART`/`SERVO` 舵角报文、`GIMBAL`
手动调试报文、`FIRE` 和遗留的 `szu_uart` 二进制帧，把它们写进舵面期望角，再交给
`servo_pid` 控制层与舵机限幅执行（都在 `guided_dart/src/config.rs`）。
**仿真器内部没有任何制导律**——比例导引、视线角速率、
瞄点变换都在视觉端（dart_vision）。没有新指令、或超过
`servo_pid.command_timeout_secs` 未收到指令时，舵面回中立，镖按纯弹道飞行。

仿真器还会通过同一条连接按 `state_telemetry_hz` 回推飞镖状态（`STATE` 行），
供视觉端做坐标变换与闭环解算。

### 6.1 射击

```text
FIRE
```

模拟器收到后从相机位置发射一颗弹丸。

弹丸命中装甲板会被**弹开**（装甲板有回弹系数，弹丸不会再"命中即消失"），并在
几秒后自动回收；**没有弹丸在飞的时候，装甲板的碰撞事件是关掉的**（平时静默等待，
只有打出去的弹丸才会做命中判定）。弹丸寿命、回弹系数分别在
`simulator/src/main.rs` 的 `PROJECTILE_TTL` / `DEFLECTED_PROJECTILE_TTL` /
`ARMOR_RESTITUTION`。

### 6.1.1 复位

```text
RESET
```

把相机朝向、整车位置和朝向、底盘/云台角度、运动速度恢复成模拟器启动状态，
并恢复**启动时的小陀螺状态**（开机是 `DAEDALUS_SMALL_GYRO=1` 的话，复位后仍然
是小陀螺；以前是无条件关掉，远程主机没有键盘就再也开不回来）。作用与本地按 `E`
键完全相同，存在的意义是让视觉端在连续多次
测试之间保证每一轮都从同一个相机位姿和底盘状态开始——否则第二轮的起始状态
会继承上一轮的结果，两轮结果无法比较。本地按 `E` 同样有效。

### 6.1.2 小陀螺（远程脚本切换）

```text
GYRO 0
GYRO 1
```

`GYRO 0` 关闭整车小陀螺、`GYRO 1` 打开；转速由 `DAEDALUS_SMALL_GYRO_SPEED`
决定（默认 `5 rad/s`）。远程主机没有键盘、按不了 `R` 时用它做脚本化启动/切换；
开机默认状态由 `DAEDALUS_SMALL_GYRO` 决定。

### 6.2 云台控制

兼容绝对角直控：

```text
GIMBAL <yaw> <pitch>
```

- `yaw`：相对相机初始固定朝向的水平角，单位弧度，向右为正。
- `pitch`：相对相机初始固定朝向的俯仰角，单位弧度，向上为正。
- 相机位置保持不变，只改变相机姿态。
- 模拟器将 `yaw` 规范到 `[-pi, pi]`，并将 `pitch` 限制在 `[-80°, 80°]`。
- 该接口是绝对角控制；重复发送会覆盖上一目标角，不累积。发送 `GIMBAL 0 0` 可恢复配置文件中定义的初始视角。

纯视觉接管模式：

```text
GIMBAL_VISION <state> <yaw> <pitch>
```

- `state=0`：视觉不控制云台，云台保持当前设定角。
- `state=1`：视觉接管云台但不开火。
- `state=2`：视觉接管云台；是否开火仍需单独发送 `FIRE`。
- `yaw`、`pitch` 是视觉算法给出的绝对目标角，单位弧度。
- 模拟器移植了 `wheel_leg_2` 的纯视觉接管策略：按误差方向分级步进，接近目标时按比例收敛，并限制云台最大角速度。
- 默认参数为 yaw 死区 `0.002 rad`、pitch 死区 `0.003 rad`、最大角速度 `3 rad/s`，与实际电控代码一致。
- 参数可通过 `DAEDALUS_GIMBAL_YAW_DEADBAND`、`DAEDALUS_GIMBAL_YAW_MIDDLE`、`DAEDALUS_GIMBAL_YAW_FINE_RATE`、`DAEDALUS_GIMBAL_YAW_COARSE_RATE`、`DAEDALUS_GIMBAL_PITCH_DEADBAND`、`DAEDALUS_GIMBAL_PITCH_RATE`、`DAEDALUS_GIMBAL_FINE_GAIN`、`DAEDALUS_GIMBAL_PITCH_LIMIT` 调整。
- 如需让旧的两参数报文也走该策略，设置 `DAEDALUS_GIMBAL_MODE=vision`。

示例：

```text
GIMBAL 0.12 -0.05
GIMBAL_VISION 1 0.12 -0.05
```

可通过 `DAEDALUS_COMMAND_ADDR` 修改监听地址。

### 6.3 `DART` 舵角控制（视觉端控制的正式接口）

```text
DART <pitch> <yaw> <roll>
```

- 三个角度单位为**弧度**，分别写入飞镖 pitch / yaw / roll 舵面的期望角。
- `SERVO <pitch> <yaw> <roll>` 是同义别名。
- 滚转通道只有在 `aero.fin_torque_scale > 0`（`guided_dart/src/config.rs`）时才真正产生力矩，否则按 0 处理。
- 四片舵面按 X 字布局的几何混合展开，混合系数由模型铰链方位角自动算出。
- `AIM <axis> <yaw> <pitch> ...` 是旧接口，**已不再被解释**（仿真器不再做比例导引）。

### 6.4 `szu_uart` 二进制控制

```text
0xFF | x_hi | x_lo | y_hi | y_lo | w | h
```

这是 7 字节固定长度帧，`x/y` 为大端 `u16`，`w/h` 为 `u8`。飞镖模式用图像中心 `(width / 2, height / 2)` 与 blob 中心 `(x + w / 2, y + h / 2)` 的归一化偏差生成 yaw/pitch 舵面期望角，水平向右和垂直向上为正。开机握手 `0xFF + "RPDART"` 会被忽略，不会污染后续二进制帧解析。

### 6.5 状态回推（`STATE`）

飞镖模式下，仿真器会向已连接的控制端按 `state_telemetry_hz`
（`guided_dart/config/simulator.toml`，默认 200 Hz）
回推飞行状态，一行一条：

```text
STATE <t_ms> <x> <y> <z> <vx> <vy> <vz> <qw> <qx> <qy> <qz> <launched> <hit> <avail_accel> <wall_us>
```

- `t_ms` 为仿真启动以来的毫秒数；位置单位米、速度单位 m/s，四元数为世界系姿态。
- `launched` / `hit` 为 `0`/`1`。
- `avail_accel` 是**当前速度下满舵能产生的侧向加速度**（m/s²），由 `aero` 的舵效
  参数算出。控制律用它把"想产生的侧向加速度"换算成舵角；放在回推里而不是让
  视觉端各配一份常量，是为了改完 `aero` 舵效后不会出现"指令凭空变弱/一直饱和"。
  旧版视觉端会忽略这个多出来的字段。
- `wall_us` 是本行的**墙钟 Unix 微秒时间戳**，与图像包头 `[20..28]` 用的是同一个钟。
  视觉端靠它把"这一帧图像"和"那一刻的镖姿态"对齐：相机固连在镖头，机身姿态抖动
  会直接进视线角，20 ms 的错配就是约 1° 的视线角误差。按字段数量严格解析的客户端
  需要注意这一行共有 15 个字段（旧的 14 字段版本没有 `wall_us`）。
- 视觉端用它把机体系视线角转换到世界系、计算惯性系 λ̇，再转回机体系输出舵角。
- 把 `state_telemetry_hz` 设为 `0` 可关闭回推（视觉端就只能靠图像自己算）。

## 7. 接收库接口

`sim_receiver::VisionDateRecevier`：

```cpp
struct FrameInfo {
    uint32_t version;      // 1 = 32 字节旧包头，2 = 40 字节包头（含云台姿态）
    uint64_t seq;
    uint64_t timestamp_us;
    float gimbal_yaw;      // 弧度，version 1 的流恒为 0
    float gimbal_pitch;
};

class VisionDateRecevier {
public:
    explicit VisionDateRecevier(const std::string& config_path);
    void overrideEndpoint(const std::string& host, uint16_t port);
    bool connect();
    bool connected() const;
    bool sendFireCommand();
    bool sendGimbalCommand(double yaw, double pitch);
    bool sendVisionGimbalCommand(int state, double yaw, double pitch);
    bool sendDartServoCommand(double pitch, double yaw, double roll);
    cv::Mat getFrame();
    cv::Mat getFrame(FrameInfo* info);
};
```

`getFrame()` 返回下一帧 `cv::Mat`，内部通过移动语义避免复制；返回空 `cv::Mat` 表示 TCP 断开、协议错误或解码失败。
接收端同时接受 40 字节（`version = 2`）与 32 字节（`version = 1`）两种包头。

`getFrame(FrameInfo* info)` 是同一个调用，额外把包头元数据写进 `info`（传 `nullptr` 等价于无参版本）。
其中 `gimbal_yaw` / `gimbal_pitch` 是**指令执行之后**的真实云台姿态，视觉端用它当
`CameraPose` 做闭环，等价实车的 IMU 反馈；遇到旧版包头时这两个字段为 `0`。

`sendFireCommand()` 会向模拟器的 `0.0.0.0:7667` 指令端口发送一行 `FIRE`，用于让接收端回发射击指令。

`sendGimbalCommand(yaw, pitch)` 会向同一指令端口发送 `GIMBAL <yaw> <pitch>`。参数单位为弧度，`yaw` 正向右，`pitch` 正向上；返回 `false` 表示参数非有限值或 TCP 发送失败。Ultra_Vision 的异步接收库会合并高频云台命令，只保留最新目标角，避免阻塞图像处理线程。

`sendVisionGimbalCommand(state, yaw, pitch)` 发送 `GIMBAL_VISION <state> <yaw> <pitch>`，用于复现电控端的纯视觉接管状态机。`state` 只允许 `0`、`1`、`2`。

`sendDartServoCommand(pitch, yaw, roll)` 发送 `DART <pitch> <yaw> <roll>`，三个参数单位均为弧度，直接设置对应舵面的期望角。飞镖模式仍会经过 PID、最大角和最大角速度限幅。

## 8. 平台支持

- Windows：Rust 模拟器端已通过 `x86_64-pc-windows-msvc` 编译检查。
- macOS：已在本机编译运行。
- Linux：已在 Ubuntu VM 中运行。
- C++ 接收端当前使用 POSIX socket，适用于 Linux/macOS；Windows 原生接收端需要 Winsock 适配。
