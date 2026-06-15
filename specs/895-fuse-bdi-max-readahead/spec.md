# Spec: 支持配置 FUSE BDI `max_readahead_kb` 以启用 1 MiB 读请求

- **Issue**: [CurvineIO/curvine#895](https://github.com/CurvineIO/curvine/issues/895)
- **Status**: Draft
- **Owner**: shaynewq
- **Created**: 2026-06-14

---

## 1. 背景（Context）

Linux FUSE 内核模块基于挂载点 BDI（Backing Device Info）的 `max_readahead_kb` 决定一次最多发起多大的读请求。该值默认通常为 **128 KB**，因此即便用户态发起一个 1 MiB 甚至更大的顺序读，FUSE 内核也会把请求 **拆分** 成多个 ≤128 KB 的子请求再分发到 `curvine-fuse`。

实测影响：

- 单次读请求所伴随的上下文切换 / 内核态 ↔ 用户态切换次数线性增长；
- 派发到底层 Worker / UFS 的请求被切碎，预读和零拷贝优化空间被压缩；
- 顺序读吞吐受限，FIO/LTP 顺序读 Benchmark 显示明显瓶颈。

当前代码现状（`curvine-fuse`）：

| 位置 | 现状 |
|---|---|
| `curvine-common/src/conf/fuse_conf.rs` `FuseConf::MAX_READ_AHEAD` | 常量 `128 * 1024`，未作为配置项暴露 |
| `curvine-fuse/src/fs/curvine_file_system.rs:635` | `init` 处理器中 `max_readahead = op.arg.max_readahead`，**完全沿用内核传入值**，不主动覆盖 |
| `curvine-fuse/src/raw/fuse_pure.rs:fuse_mount_sys` | 仅完成 `mount(2)`，**未触碰 `/sys/class/bdi/.../max_readahead_kb`** |
| `curvine-fuse/src/session/fuse_session.rs:237` `setup_mnts` | 串行为每个挂载点构造 `FuseMnt`，**挂载成功后没有任何 post-mount 钩子** |

由于上述两处都没有处理，用户即便希望开大读请求也无路径可走。

## 2. 目标（Goals）

- **G1**：允许通过配置（`fuse.max_readahead_kb`）让 `curvine-fuse` 在每个挂载点 mount 成功后，写入对应 BDI 的 `max_readahead_kb`，从而让 FUSE 内核可以发起最大 1 MiB（默认建议值）的读请求。
- **G2**：在 FUSE `init` 协议握手时把 `fuse_init_out.max_readahead` 设置为 ≥ 配置值，避免被 `min(bdi.max_readahead_kb, fuse_conn.max_readahead)` 这条规则反向卡死。
- **G3**：保持 100% 向后兼容：未显式配置时行为与今日完全一致（不写 `/sys/class/bdi/...`，`max_readahead` 沿用内核值）。
- **G4**：操作失败（权限不足、文件不存在等）必须 **降级为日志告警**，不影响挂载主流程。

## 3. 非目标（Non-Goals）

- 不在本次改动里调整 `MAX_READ` / `MAX_WRITE`（128 KB → 1 MiB 涉及缓冲区栈、splice 路径，需独立设计）。
- 不引入针对 `max_readahead_kb` 的运行时动态调整 / API；只支持启动期一次性写入。
- 不覆盖 macOS / FreeBSD：`/sys/class/bdi` 是 Linux 专有路径。
- 不调整 Worker 端 / Master 端默认参数。

## 4. 用户故事（User Stories）

- **US-1**（运维）作为部署 Curvine FUSE 的 SRE，我希望在 `curvine-cluster.toml` 中加一行配置即可让顺序读 IO 大小翻 8 倍，无需手动 `echo > /sys/class/bdi/...`，也无需重新编译。
- **US-2**（开发者）作为只想验证默认行为不被破坏的开发者，我升级到带有该改动的版本后，**不修改任何配置**，挂载行为与读写性能与升级前一致。
- **US-3**（容器场景使用者）作为在受限容器内运行 FUSE 的用户，当对 `/sys/class/bdi` 没有写权限时，我希望挂载继续成功，仅在日志看到一行 WARN 提示。

## 5. 需求（Requirements）

### 功能性需求

- **FR-1** `FuseConf` 新增字段 `max_readahead_kb: Option<u32>`（KB 单位）。
  - `None`（默认）= 不启用本特性。
  - `Some(v)` 时，**v 必须 > 0**，否则启动期校验失败并报错退出。
  - 推荐配置值：`1024`（即 1 MiB）。
- **FR-2** 当 `max_readahead_kb = Some(v)` 时，`fuse_init_out.max_readahead` 应被设置为 `max(op.arg.max_readahead, v * 1024)`。
- **FR-3** 当 `max_readahead_kb = Some(v)` 时，挂载成功后 **针对每个挂载点** 解析其 `st_dev`（`major:minor`），向 `/sys/class/bdi/<major>:<minor>/max_readahead_kb` 写入十进制字符串 `v`。
- **FR-4** 写入失败（路径不存在 / `EACCES` / `EPERM` / `ENOENT` 等）时，记录 `WARN` 级日志，包含挂载点路径、目标 sysfs 路径、错误原因，**不影响 mount 结果**。
- **FR-5** 写入成功后输出 `INFO` 级日志，包含挂载点、`major:minor`、写入值。
- **FR-6** `curvine-fuse list-config-flags` 子命令需要列出新字段（已自动通过 serde 派生）。

### 非功能性需求

- **NFR-1** 仅在 `cfg(target_os = "linux")` 下编译特性代码；macOS 构建不引入新依赖、不暴露字段。
- **NFR-2** 不增加新的运行时依赖（`std::fs::write` 即可）。
- **NFR-3** 关键路径（read 热路径）零额外开销：BDI 写入仅在挂载启动期发生一次。

## 6. 验收标准（Acceptance Criteria）

- **AC-1**（默认行为不变）：测试集群在 `curvine-cluster.toml` 中 **不配置** `fuse.max_readahead_kb` 时：
  - `cat /sys/class/bdi/<dev>/max_readahead_kb` 数值与升级前一致；
  - `fs_test`/`fuse_test` 全部通过。
- **AC-2**（开启后生效）：在 `[fuse]` 段加上 `max_readahead_kb = 1024` 后：
  - 启动日志中出现 `INFO ... bdi max_readahead_kb set: path=..., bdi=<maj:min>, value=1024`；
  - `cat /sys/class/bdi/<maj>:<min>/max_readahead_kb` 输出 `1024`；
  - 用 `dd if=<file> of=/dev/null bs=1M count=128` 触发顺序读，通过 `bpftrace -e 'tracepoint:fuse:fuse_request_send { @[args->opcode] = count(); }'`（或 `fuse.debug=true` 日志统计）观测到 `READ` opcode 的请求个数相对默认配置 **下降至约 1/8**。
- **AC-3**（容错）：把 `/sys/class/bdi/<dev>/max_readahead_kb` 临时 `chmod 444` 模拟无写权限，启动应继续成功，并在日志中出现一条 WARN，不出现 panic。
- **AC-4**（参数合法性）：配置 `max_readahead_kb = 0` 时，`curvine-fuse` 启动直接退出并打印明确错误。
- **AC-5**（多挂载点）：`mnt_number = 2` 场景下，两个挂载点对应的 BDI 都被设置成功（日志各出一条 INFO）。

## 7. 风险与权衡（Risks & Trade-offs）

| 风险 | 缓解 |
|---|---|
| `/sys/class/bdi/...` 路径在不同内核版本下命名方式（`<major>:<minor>` vs `0:N`）可能差异 | 通过 `statfs(mnt) → f_fsid` / `stat(mnt).st_dev` 推导 `major:minor`，并对返回路径不存在做软失败处理 |
| 容器内（rootless / 无 `CAP_SYS_ADMIN`）写 sysfs 失败 | FR-4 已保证软失败：仅 WARN 不阻断 |
| 设置过大值会增加内核预读内存压力 | 文档显式标注推荐值 1024，不在代码侧硬限上限，由用户自负责 |
| 与未来"提升 `MAX_READ` 到 1 MiB"改动耦合 | 本 spec 只涉及 BDI 与 init 协议字段，二者解耦；后续提升 `MAX_READ` 时只需复用同一字段 |

## 8. 影响面（Impact）

- **配置兼容性**：新增可选字段，旧 `curvine-cluster.toml` 无需任何修改。
- **二进制兼容性**：仅修改 `curvine-fuse` 内部，与 master/worker、客户端 SDK 无关。
- **平台**：Linux 单边收益；macOS 编译保持绿色，但字段无效（编译期 `cfg` 屏蔽实际写 sysfs 的代码路径）。
- **文档**：需要更新仓库根 `README.md` 的 FUSE 调优段落（如果有）以及 `etc/curvine-cluster.toml` 的注释样例。

## 9. 开放问题（Open Questions）

- **OQ-1** 是否需要一并把 `MAX_READ_AHEAD` 这个常量从 128KB 提到默认值更大？— 倾向 **不动**，让常量作为下界。
- **OQ-2** 是否要支持 `max_readahead_bytes`（更精细单位）而不是 `_kb`？— 倾向沿用内核 sysfs 的 KB 单位，避免单位换算分歧。
