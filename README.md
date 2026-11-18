# Gripper Action Policy Node

[English](README.md) | [简体中文](README_zh.md)

## BREAKING CHANGE: Forge 2.x / Dora 1.x migration

This migration from Forge 1.x / Dora 0.4.x requires coordinated deployment. Align the Forge packages, Dora CLI/coordinator/daemon, this node and its peer nodes before connecting a dataflow; replacing only one node in a Forge 1.x deployment is unsupported. Rebuild and redeploy executable artifacts with the matching environment. Keep `stable/1.x` for deployments that still use the previous stack.

The validated Python baseline is `forge-msgs==2.1.0`, `Pillow==13.0.0`, `forge-common==2.1.0`, the applicable `forge-robot` / `forge-policy` / `forge-kinematics` / `forge-tool` packages at `2.0.0`, and `dora-rs==1.0.1`. Respect this repository's Python and SDK platform requirements. Forge package families have independent versions. Rust nodes retain the published `forge_msgs` crate `2.0.0` with Dora `1.0.1`; Python Msgs `2.1.0` does not require a nonexistent Rust `2.1.0` release. The `forge_msgs.v1` message namespace remains unchanged.

This notice describes the deployment break from the previous stack; it does not declare hardware, real-model or sustained-run acceptance complete. See [this round's verification report](migration/msgs-2.1.0-report.json) for actual tests and outstanding gates.

Standalone Dora policy Node that adapts the Forge Action Endpoint
`gripper.controller/set_opening` to a `forge_msgs.GripperCommand` controller.

## Node, Endpoint and Tool API

| Item | Current implementation |
| --- | --- |
| Production Nodes | 1: `gripper_action_policy`; source command `gripper-action-policy`, packaged entrypoint `dist/gripper_action_policy` |
| Tool integration | Dora `ToolMessage` provider; caller-facing access through Gateway |
| Direct HTTP Tool API | 0; this Node does not listen on HTTP; Gateway owns the invocation URL |
| Endpoints | 1: `gripper.controller` |
| Operations | 1: `set_opening` (Action, cancellable, status supported, maximum concurrency 1) |
| Gateway Tools | Deployment-defined in `tools.specs`; `robot-control-sim` binds 1 Tool to this operation: `sim.set_gripper_opening` |

| Tool ID (Skill configuration) | Endpoint / operation | Provider Node |
| --- | --- | --- |
| `sim.set_gripper_opening` | `gripper.controller / set_opening` | `gripper_action_policy` |

Arguments are exactly `{"opening_m": 0.04}` (metres); see the [contract](#contract) below.
This Node translates Tool invocations to `GripperCommandGoal`, then translates downstream
feedback/results into Tool status and results. It neither executes the controller nor runs a learned model.

```text
Gateway -> gripper_action_policy -> gripper_action_controller -> MuJoCo / robot Adapter
                                   <- JointState feedback <-
```

[`gripper_action_controller`](https://github.com/PhyAgentOS/operator-controls/tree/dev/packages/gripper_action_controller)
belongs to `operator-controls`. It generates joint-position commands and monitors arrival, stall,
timeout and cancellation; it does not provide a ToolEndpoint itself.
This policy's metre-based opening interface cannot directly serve a radian-based gripper.
Implementation: [descriptor and Action adapter](src/gripper_action_policy_node/action_endpoint.py),
[Dora runner](src/gripper_action_policy_node/runner.py).

Compatibility: current dependency declarations are `dora-rs>=1.0.1,<2`, `forge-msgs>=2.0.0,<3`. See [pyproject.toml](pyproject.toml) and [uv.lock](uv.lock). The Tool protocol implementation comes from the official `forge-tool[dora]` distribution. These declarations do not guarantee cross-version interoperability; also check the build revision and Skill lock when using artifacts.

Changes to Node entrypoints, Endpoints or Tool APIs must update this section and the associated configuration/contracts. Follow the [Node README contract](https://github.com/PhyAgentOS/forge/blob/master/CONTRIBUTING.md#node-readme-contract).

## Contract

- The descriptor exposes `set_opening` as a cancellable Action with status lookup and maximum concurrency 1.
- Caller arguments are exactly `{"opening_m": <finite number>}`.
- The policy validates finiteness and converts the value to
  `GripperCommandGoal(position=opening_m, max_velocity=None, max_effort=None)`.
- Gateway schema and the downstream controller each validate the `0..0.105 m` mechanical bounds. This Node does not duplicate those bounds.
- Each Tool attempt maps to a deterministic, bounded controller `goal_id`; retrying the same attempt does not resend the goal.

## Install and run

Forge messages resolve from public package indexes (`forge-msgs>=2.0.0,<3`);
the Forge Tool protocol is vendored under `src/forge_tool`:

```bash
uv sync
uv run gripper-action-policy
```

## Dora ports

Inputs:

- `tick`: drives Endpoint lease registration and renewal.
- `tool_in`: Forge `ToolMessage`, including registry responses and Action control requests.
- `gripper_feedback`: `forge_msgs.GripperCommandFeedback`, with `goal_id` in metadata.
- `gripper_result`: `forge_msgs.GripperCommandResult`, with `goal_id` and `goal_status` in metadata.

Outputs:

- `tool_out`: registration, unregistration, Action responses, status, results and `ToolEvent`.
- `gripper_goal`: `forge_msgs.GripperCommandGoal`, with `goal_id` in metadata.
- `gripper_cancel`: boolean trigger, with `goal_id` in metadata.

The runner composes Forge's public `ToolEndpointHandler`, `DoraToolEndpointBinding` and
`EndpointLease`; it does not implement its own Tool Wire dispatch. Shutdown first stops
accepting new goals and requests cancellation. Accepted cancellation is not a controller-confirmed
terminal state, so shutdown with an active goal is explicitly treated as ambiguous.

## Test and build

```bash
uv run pytest
uv run ruff check .
./scripts/build_pyinstaller.sh
```

The PyInstaller single-file output is `dist/gripper_action_policy`.

## Development

```bash
uv sync --all-groups
uv run pytest
uv run ruff check .
```

## License

Apache-2.0. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

## Forge 2.x migration

This branch prepares node version `2.0.0` for Dora 1.x, frozen to Dora `1.0.1` for migration validation. It uses the official Forge 2.x distributions; Common is at least `2.1.0`. Regenerated locks and all executable entrypoints are part of the change.

Every change report must list repository, changes and test evidence, including per-node `dora run` records and a chained test containing `adapter-forge-gateway`. Unavailable hardware/model checks remain pending. Source PRs do not publish stable releases or synchronize GitLab; synchronization follows full acceptance.

ToolEndpoint implementations now come from `forge-tool[dora]>=2.0.0,<3`. The former repository-local Tool copy and private message imports are removed; messages use the public `forge_msgs.ToolMessage`.

## Versioned releases

| Release line | Source | Node/package version | Runtime |
| --- | --- | --- | --- |
| Forge 1.x | `stable/1.x` | `1.x`, tag `v1.X.Y` | Forge 1.x / Dora 0.4.x |
| Forge 2.x | `master` | `2.x`, tag `v2.X.Y` | Forge 2.x / Dora 1.x |

**Legacy normalization:** nodes previously at 0.x on `stable/1.x` release as **1.0.0**, retaining Forge 1.x / Dora 0.4.x dependencies and runtime behavior. Existing 1.x package versions are retained. Node/package majors now follow the Forge compatibility line; a 0.x tag is rejected.

Run **Build versioned release** from `master` for either line. First merge the version/lock changes into the selected source branch, then create an unused **annotated** tag on that reviewed commit. The tag must match package metadata. Never move existing tags or reuse a published version. Dispatch uses the workflow's reviewed tooling with the exact tagged source, so legacy tags need not contain the new release helper.

```bash
# Replace with an existing, reviewed and unpublished tag matching source metadata.
TAG=v1.X.Y
gh workflow run release.yml --ref master -f release_line=1.x -f tag="$TAG"
TAG=v2.X.Y
gh workflow run release.yml --ref master -f release_line=2.x -f tag="$TAG"
```

The workflow verifies tag type, source commit, branch ancestry, source version and locked Forge/Dora compatibility. It builds source archives and wheel/sdist where supported, checks Python distribution metadata and SHA-256, and creates or attaches to a **draft GitHub Release**. Published releases/assets are not overwritten; review all build results and acceptance evidence before publishing. This workflow does not publish node packages to PyPI.

Set `build_binaries=true` to call the existing Ubuntu 20.04 binary workflow; its existing architecture, CLI and archive checks remain in force. You can also run `build-ubuntu20.yml` directly with `ref`, `version` and `release_line`; `ref` must be an annotated tag, not a moving branch. Binary artifacts are attached only after all selected jobs succeed.

The source archive preserves the tagged tree; external SDKs/submodules, model downloads and site configuration remain separate prerequisites. A dependency or SDK build failure is not successful release acceptance. Build success does not establish single-node or Gateway-chain `dora run`, hardware, real-model, sustained-run or rollback acceptance. Every acceptance chain must include `adapter-forge-gateway` from the same compatibility line. Forge 2.x is a **breaking change**: align Forge libraries, node versions and Dora across the entire deployment; keep 1.x on its maintenance branch.
