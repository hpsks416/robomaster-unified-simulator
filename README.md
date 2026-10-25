# RoboMaster 大一统模拟器

浏览器**一个面板**：配置 + 相机画面 + 指令遥控；物理、3D 渲染、协议全在 **daedalus（Rust/Bevy）单核心**。

> ⚠️ **当前状态**：仓库暂不构成可完整部署的项目。`simulator/` 目录**只收录 daedalus 的文档**（协议 / 用法 / 部署），**源码未入库**——尊重原作者的小范围传播授权，仅用于交流思路；项目完善后再整体收录源码。

## 仓库结构

| 目录 | 内容 | 许可 |
|---|---|---|
| `frontend/` | 浏览器前端：3D 查看器 + 仿真器控制台（配置面板 / 相机画面 / 指令）+ 桥接 + CAD 流水线 | MIT |
| `simulator/` | daedalus **文档**（README / API / USAGE / TUTORIAL / 仿真器须知），**源码未入库** | 参考 AGPL v3.0（见署名） |
| `docs/` | 大一统模拟器规划 + 架构 / 方法论学习笔记 | MIT |

## 快速开始

```bash
# 前端 + 控制台（不含仿真核心；完整联调需本地 daedalus 源码，见 simulator/仿真器须知.md）
cd frontend
python run.py
```

- 前端 + 控制台用法：见 `frontend/README.md`
- 仿真器文档（协议 / 接口 / 思路）：见 `simulator/仿真器须知.md`、`simulator/API.md`、`simulator/README.md`

## 规划

- [大一统模拟器 · 规划](./docs/unified-simulator-plan.md)
- [学习笔记：Simate 的 Physical AI 研发方法论 → 本项目映射](./docs/physical-ai-simate-study.md)
- [六底座 ↔ 本项目架构对照](./docs/six-pillars-architecture.md)
- [驾驶 + 关节统一状态机](./docs/robot-state-machine.md)

## 许可

- `frontend/`、`docs/` 及本 README：**MIT** © 2026 hpsks416（见 `frontend/LICENSE`）
- `simulator/`：**仅文档**；对应的 daedalus 源码为 **GNU AGPL v3.0**（原作者河北科技大学 刘续，二次开发成都大学 谢金恩；源码未随本仓库发布，见 `simulator/仿真器须知.md`）
- 详见 [NOTICE.md](./NOTICE.md)
