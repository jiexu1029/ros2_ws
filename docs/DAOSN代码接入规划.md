# DAOSN 代码接入规划

## 1. 目标

在 `ros2_ws` 中建立一个可持续维护的 DAOSN ROS2 集成工作区，统一承载：

- 工作区初始化方式
- 多仓库源码接入方式
- 编译与运行入口
- 分阶段迁移和联调计划
- 风险清单与验收标准

本规划的重点不是重写现有业务代码，而是让当前分散的 `daosnrs_*` 代码能稳定进入一个新的工作区并具备持续集成条件。

## 2. 当前现状

截至 2026-03-28，`/home/w/xj_ws/src` 下已存在多个独立 Git 仓库：

- `daosnrs_common`
- `daosnrs_interfaces`
- `daosnrs_driver`
- `daosnrs_craft`
- `ros2_ws`

其中 `ros2_ws` 目前几乎为空仓库，仅有 `README.md`，尚未承载实际源码、脚本或集成说明。

现有 ROS2 包盘点如下：

| 代码仓库 | 包名/模块 | 角色 |
| --- | --- | --- |
| `daosnrs_interfaces` | `daosnrs_interfaces`、`system_interfaces`、`sensor_interfaces`、`driver_interfaces`、`nav_interfaces` | 消息/服务/动作接口层 |
| `daosnrs_common` | `daosnrs_common` | 公共工具、日志、几何能力、基础配置 |
| `daosnrs_driver` | `wheels_driver_lib`、`wheels_driver_node`、`beiwei_imu_driver`、`range_sensor_tfmini`、`ultrasound_ks104`、`DSNsControl`、`DSLocationControl` | 驱动和定位接入层 |
| `daosnrs_craft` | `daosnrs_craft_controller`、`daosnrs_craft_server` | 上装控制、任务执行 |

## 3. 已识别的关键问题

### 3.1 缺少核心依赖仓库

当前代码显式依赖 `daosnrs_planning`，但在 `/home/w/xj_ws/src` 中未发现对应仓库或包。以下模块均依赖它：

- `daosnrs_craft_server`
- `DSLocationControl`
- `daosnrs_common` 中的路径相关实现

这意味着：

- 当前代码无法在完整依赖闭环下直接编译
- `ros2_ws` 的首个阻塞项不是脚手架，而是补齐 `daosnrs_planning`

### 3.2 代码当前是“多仓库”而不是“单仓库”

现状已经天然按领域拆成多个 Git 仓库，因此不建议在 `ros2_ws` 中直接复制源码形成双份维护。推荐方式是：

- `ros2_ws` 只管理工作区级内容
- 通过 `.repos` 或其他清单方式拉取各业务仓库
- 各功能仓库继续独立演进

### 3.3 包和依赖存在历史遗留问题

从当前代码可见，后续接入时需要顺手清理几类问题：

- 包命名不统一，如 `DSLocationControl`、`DSNsControl` 使用大写风格
- 个别依赖声明疑似错误，如 `beiwei_imu_driver/package.xml` 中的 `geomery_msgs`
- 个别包的 `package.xml`/构建声明存在历史痕迹，需要重新核对实际用途

这些问题不一定阻塞第一天的接入，但会影响持续集成和后续规范化。

## 4. 推荐的 ros2_ws 目标结构

建议将 `ros2_ws` 定位为“DAOSN ROS2 集成仓库”，结构如下：

```text
ros2_ws/
├── README.md
├── docs/
│   └── DAOSN代码接入规划.md
├── repos/
│   └── daosnrs.repos
├── scripts/
│   ├── bootstrap.sh
│   ├── build.sh
│   └── run_env.sh
├── config/
│   └── colcon.defaults.yaml
└── src/
    ├── daosnrs_interfaces
    ├── daosnrs_common
    ├── daosnrs_driver
    ├── daosnrs_craft
    └── daosnrs_planning
```

说明：

- `src/` 中的目录由 `vcs import` 或统一脚本拉取，不手工复制
- `repos/daosnrs.repos` 用来锁定仓库地址和分支
- `scripts/` 负责环境初始化、编译和联调入口
- `config/` 负责 `colcon` 默认参数、日志输出和覆盖编译配置

## 5. 推荐接入策略

### 5.1 总体策略

采用“先建工作区，再分层接入，最后联调”的方式：

1. 先把 `ros2_ws` 搭成标准工作区骨架
2. 先接入基础层：`interfaces` + `common`
3. 再接入驱动层：`driver`
4. 补齐 `daosnrs_planning`
5. 最后接入 `craft` 并完成整链联调

### 5.2 为什么不建议直接复制代码

如果把 `daosnrs_*` 代码直接复制到 `ros2_ws`：

- 同一模块会有两份源码
- 分支和提交历史会断裂
- 后续修复无法判断应该改哪一份
- 多产品线分支管理会失控

因此，`ros2_ws` 应该做“集成编排”，不做“源码重复托管”。

## 6. 分阶段实施计划

### P0：前置确认

目标：确认接入边界和阻塞项。

输出物：

- `daosnrs_planning` 仓库地址、分支和负责人
- 本次接入的产品线分支，例如 `m10-dev` / `m15-dev` / `q5-dev`
- ROS2 版本、Ubuntu 版本、编译器版本
- 必备三方依赖清单

验收标准：

- 能明确列出完整仓库集合
- 能确认 `daosnrs_planning` 是否存在、在哪个分支、是否可用

预计工作量：0.5 到 1 天

### P1：搭建 ros2_ws 工作区骨架

目标：让 `ros2_ws` 成为可复用的集成仓库。

工作项：

- 新增 `repos/daosnrs.repos`
- 新增 `scripts/bootstrap.sh`
- 新增 `scripts/build.sh`
- 新增 `config/colcon.defaults.yaml`
- 在 `README.md` 中补充初始化、导入和编译说明

验收标准：

- 新机器上可通过一套命令完成工作区初始化
- `src/` 目录可自动拉起所需仓库

预计工作量：0.5 到 1 天

### P2：接入基础层

目标：先确保接口层和公共层稳定编译。

范围：

- `daosnrs_interfaces`
- `daosnrs_common`

工作项：

- 校验接口包可独立编译
- 校验 `daosnrs_common` 对接口包的依赖闭环
- 整理公共配置文件和日志目录约定

验收标准：

- `colcon build --packages-up-to daosnrs_common` 成功
- 基础头文件、消息和公共库能被其他包引用

预计工作量：1 天

### P3：接入驱动层

目标：把底盘、传感器、定位接入到统一工作区。

范围：

- `wheels_driver_lib`
- `wheels_driver_node`
- `beiwei_imu_driver`
- `range_sensor_tfmini`
- `ultrasound_ks104`
- `DSNsControl`
- `DSLocationControl`

工作项：

- 校验驱动类包的依赖声明
- 修复显式构建错误和明显的包声明问题
- 梳理每个驱动的启动参数、串口/CAN/网络依赖
- 建立最小启动链路

验收标准：

- 驱动层可独立编译
- 核心驱动节点可启动
- 关键 topic/service/action 能正常注册

预计工作量：2 到 3 天

### P4：补齐规划层

目标：恢复或接入 `daosnrs_planning`，打通上层依赖。

工作项：

- 找回 `daosnrs_planning` 源码仓库
- 确认其 API 是否与现有 `craft`、`driver` 代码匹配
- 补齐缺失头文件、库和运行参数
- 验证 `FollowPath` 相关动作链路

验收标准：

- `daosnrs_planning` 可被 `DSLocationControl`、`daosnrs_craft_server` 链接
- 相关路径对象和动作调用链不再缺失

预计工作量：1 到 3 天

说明：这是当前最关键的阻塞阶段。

### P5：接入工艺层

目标：完成业务执行层接入。

范围：

- `daosnrs_craft_controller`
- `daosnrs_craft_server`

工作项：

- 校验对 `driver/common/planning/interfaces` 的依赖
- 整理作业流程相关配置
- 确认任务入口、动作入口和状态回传链路

验收标准：

- `craft` 相关包可完整编译
- 主任务流程能完成基础启动和接口联通

预计工作量：1 到 2 天

### P6：联调与交付

目标：形成可交付、可复用、可持续维护的工作区。

工作项：

- 固化编译命令和启动命令
- 增加最小 smoke test
- 输出问题清单和已知限制
- 形成版本冻结说明

验收标准：

- 新成员可按文档在新环境完成拉取、编译和启动
- 核心链路具备最小回归验证手段

预计工作量：1 到 2 天

## 7. 建议优先级

优先级建议如下：

1. 优先找回并确认 `daosnrs_planning`
2. 在 `ros2_ws` 中建立 `.repos + scripts + colcon` 基础设施
3. 先打通 `interfaces/common`
4. 再做 `driver`
5. 最后收口 `craft`

如果第 1 步无法完成，后续只能做到“部分工作区可编译”，无法完成完整业务链路。

## 8. 风险清单

| 风险 | 影响 | 应对方式 |
| --- | --- | --- |
| `daosnrs_planning` 缺失 | 无法打通完整编译和运行链路 | 作为 P0 阻塞项优先补齐 |
| 多产品线分支不统一 | 难以确认应接入哪个版本 | 在 P0 固定产品线和基线分支 |
| 驱动依赖外设 | 编译通过但运行失败 | 分离“编译验收”和“真机验收” |
| 历史包声明不规范 | CI 不稳定、构建报错 | 在 P3 前集中修正明显问题 |
| 配置文件分散 | 启动环境不可复现 | 在 `ros2_ws` 统一脚本和配置入口 |

## 9. 交付物建议

建议 `ros2_ws` 第一阶段至少交付以下文件：

- `docs/DAOSN代码接入规划.md`
- `repos/daosnrs.repos`
- `scripts/bootstrap.sh`
- `scripts/build.sh`
- `config/colcon.defaults.yaml`
- 完整的 `README.md`

## 10. Git 执行建议

结合当前工作区 Git 规范，后续如果开始真正落地实现，建议：

- 从目标产品线开发分支拉出功能分支
- 分支命名采用 `feature/{product}/{developer}/ros2-ws-bootstrap`
- 提交信息采用 Conventional Commits，例如：
  - `docs(ros2_ws): add daosn integration plan`
  - `chore(ros2_ws): add workspace bootstrap scripts`

## 11. 结论

`ros2_ws` 的正确角色应是 DAOSN 多仓库 ROS2 集成入口，而不是新的源码复制仓库。

当前最现实的路线是：

1. 先补齐缺失的 `daosnrs_planning`
2. 再把 `ros2_ws` 搭成标准工作区骨架
3. 按 `interfaces -> common -> driver -> planning -> craft` 顺序分层接入

只要 `daosnrs_planning` 能补齐，这条路线可以较低风险地把现有 DAOSN 代码收口到一个可维护的 ROS2 工作区中。
