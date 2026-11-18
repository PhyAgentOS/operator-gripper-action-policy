# 夹爪动作策略节点

[English](README.md) | [简体中文](README_zh.md)

独立 Dora policy node，为 Forge Action endpoint
`gripper.controller/set_opening` 提供 `forge_msgs.GripperCommand` 控制器适配。

## Node、Endpoint 与 Tool API

| 项目 | 当前实现 |
| --- | --- |
| 生产 Node | 1：`gripper_action_policy`；源码命令 `gripper-action-policy`，打包入口 `dist/gripper_action_policy` |
| Tool 接入方式 | Dora `ToolMessage` provider；通过 Gateway 暴露调用入口 |
| 直接 HTTP Tool API | 0；本 Node 不监听 HTTP，调用地址属于 Gateway |
| Endpoint 数 | 1：`gripper.controller` |
| operation 数 | 1：`set_opening`（Action，可取消、可查状态，最大并发 1） |
| Gateway Tool 数 | 由部署的 `tools.specs` 决定；`robot-control-sim` 为此 operation 配置 1 个 Tool：`sim.set_gripper_opening` |

| Tool ID（Skill 配置） | Endpoint / operation | provider Node |
| --- | --- | --- |
| `sim.set_gripper_opening` | `gripper.controller / set_opening` | `gripper_action_policy` |

调用参数严格为 `{"opening_m": 0.04}`（单位 m）；完整行为见下方[契约](#契约)。
本节点把 Tool 调用转换为 `GripperCommandGoal`，再将下游 feedback/result 转回 Tool 状态与结果。
它不承担控制器执行，也不运行学习模型。

```text
Gateway -> gripper_action_policy -> gripper_action_controller -> MuJoCo / robot Adapter
                                   <- JointState feedback <-
```

[`gripper_action_controller`](https://github.com/PhyAgentOS/operator-controls/tree/dev/packages/gripper_action_controller)
属于 `operator-controls`，按目标生成关节位置命令并监视到位、卡滞、超时和取消；它自身不提供 ToolEndpoint。
本 policy 的米制 opening 接口不能直接当作弧度制夹爪接口使用。
实现：[descriptor 与 Action 适配](src/gripper_action_policy_node/action_endpoint.py)、
[Dora runner](src/gripper_action_policy_node/runner.py)。

兼容性：当前依赖声明为 `dora-rs==0.4.1`、`forge-msgs==1.0.1`；以 [pyproject.toml](pyproject.toml) 与 [uv.lock](uv.lock) 为准。 Tool 协议实现为仓库内 vendored 代码。 这些声明不构成跨版本互操作保证；使用制品时还需核对 build revision 和 Skill lock。

后续修改 Node 入口、Endpoint 或 Tool API 时，请同步更新本节及对应配置/契约，遵循 [Node README 规范](https://github.com/PhyAgentOS/forge/blob/master/CONTRIBUTING.md#node-readme-contract)。

## 契约

- endpoint descriptor：`set_opening` 是可取消、支持状态查询、最大并发为 1 的 Action。
- caller 参数严格为 `{"opening_m": <finite number>}`。
- policy node 只校验有限数并转换为
  `GripperCommandGoal(position=opening_m, max_velocity=None, max_effort=None)`。
- `0..0.105 m` 的机械边界由 Gateway schema 与下游 controller 分别验证。本节点不复制该边界。
- Tool attempt 映射到确定且有界的 controller `goal_id`；同一 attempt 重试不会重复下发。

## 安装与运行

Forge 消息从公共包索引解析（`forge-msgs==1.0.1`）；Forge Tool 协议以 `src/forge_tool` 内置（vendored）：

```bash
uv sync
uv run gripper-action-policy
```

## Dora 端口

输入：

- `tick`：驱动 endpoint lease 注册和续租。
- `tool_in`：Forge `ToolMessage`，包括 registry response 与 Action 控制请求。
- `gripper_feedback`：`forge_msgs.GripperCommandFeedback`，metadata 带 `goal_id`。
- `gripper_result`：`forge_msgs.GripperCommandResult`，metadata 带 `goal_id` 和
  `goal_status`。

输出：

- `tool_out`：注册、注销、Action 响应、状态、结果及 `ToolEvent`。
- `gripper_goal`：`forge_msgs.GripperCommandGoal`，metadata 带 `goal_id`。
- `gripper_cancel`：布尔触发值，metadata 带 `goal_id`。

runner 组合 Forge 公共 `ToolEndpointHandler`、`DoraToolEndpointBinding` 与
`EndpointLease`，不自行实现 Tool Wire dispatch。进程停止时先停止接收新 goal 并请求取消；
cancel 被接收不代表 controller 已确认终态，因此活跃 goal 的 shutdown 状态明确视为 ambiguous。

## 测试与构建

```bash
uv run pytest
uv run ruff check .
./scripts/build_pyinstaller.sh
```

PyInstaller 单文件产物为 `dist/gripper_action_policy`。

## 开发

```bash
uv sync --all-groups
uv run pytest
uv run ruff check .
```

## 许可证

本仓库使用 Apache-2.0，详见 [LICENSE](LICENSE) 与 [NOTICE](NOTICE)。

## 分版本发布

| 维护线 | 源码 | 节点/包版本 | 运行时 |
| --- | --- | --- | --- |
| Forge 1.x | `stable/1.x` | `1.x`，标签 `v1.X.Y` | Forge 1.x / Dora 0.4.x |
| Forge 2.x | `master` | `2.x`，标签 `v2.X.Y` | Forge 2.x / Dora 1.x |

**旧版版本号统一：** `stable/1.x` 原为 0.x 的节点以 **1.0.0** 发布，保留 Forge 1.x / Dora 0.4.x 依赖与运行行为。已有 1.x 版本保留。节点/包主版本与 Forge 兼容线一致，不再接受 0.x 发布标签。

两条线均从 `master` 运行 **Build versioned release**。先将版本和锁文件修改通过 PR 合并至对应源码分支，再在已审查提交上创建未占用的 **annotated** 标签，标签必须匹配包元数据。禁止移动已有标签或复用已发布版本。工作流使用自身已审查的发布工具与精确标签源码，旧标签无需包含新发布脚本。

```bash
# 替换为已存在、已审查、未发布且与源码元数据一致的标签。
TAG=v1.0.0
gh workflow run release.yml --ref master -f release_line=1.x -f tag="$TAG"
TAG=v2.X.Y
gh workflow run release.yml --ref master -f release_line=2.x -f tag="$TAG"
```

流水线校验标签类型、源码提交、分支归属、源码版本及锁定的 Forge/Dora 兼容性；构建源码归档及项目支持的 wheel/sdist，检查 Python 分发元数据与 SHA-256，并创建或附加至 **GitHub Release 草稿**。不覆盖已发布 Release 或产物；发布前检查全部构建结果与验收依据。该流程不发布节点包至 PyPI。

设置 `build_binaries=true` 复用已有 Ubuntu 20.04 二进制工作流，保留原架构、CLI 和归档校验。也可直接运行 `build-ubuntu20.yml`，传入 `ref`、`version`、`release_line`；`ref` 必须为 annotated 标签，不能为移动分支。仅在所有选定任务通过后附加二进制产物。

源码归档保留标签中的文件；外部 SDK/submodule、模型下载与现场配置仍是独立前置条件。依赖或 SDK 构建失败不能视为发布验收通过。构建成功不代表单节点或串联 `dora run`、真机、真实模型、长稳态或回滚验收通过；每条验收链必须包含相同兼容线的 `adapter-forge-gateway`。Forge 2.x 属于 **breaking change**，整个部署的 Forge 库、节点版本与 Dora 必须对齐；1.x 保留在维护分支。
