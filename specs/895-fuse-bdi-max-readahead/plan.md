# Plan: 支持配置 FUSE BDI `max_readahead_kb`

- **Spec**: [./spec.md](./spec.md)
- **Issue**: [#895](https://github.com/CurvineIO/curvine/issues/895)

## 1. 总体方案

在 **三个层面** 协同放开"FUSE 一次性读 1 MiB"链路：

```
┌─────────────────────────────────────────────────────────────────────┐
│                    curvine-cluster.toml                             │
│  [fuse]                                                             │
│  max_readahead_kb = 1024     # ← 新增字段（可选，默认 None）         │
└──────────────────┬──────────────────────────────────────────────────┘
                   │ 序列化加载
                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│ curvine-common::conf::FuseConf                                      │
│   pub max_readahead_kb: Option<u32>          [新增]                 │
│   FuseConf::init() ─ 校验 Some(0) → 报错                            │
└──────────────────┬──────────────────────────────────────────────────┘
                   │
   ┌───────────────┴────────────────┐
   ▼                                ▼
┌─────────────────────────┐   ┌────────────────────────────────────┐
│ FUSE init handshake     │   │ Mount lifecycle                    │
│ curvine_file_system.rs  │   │ session/fuse_session.rs            │
│ async fn init():        │   │   setup_mnts() 之后调用            │
│   max_readahead =       │   │   apply_bdi_max_readahead()  [新增]│
│     max(arg, conf*1024) │   │   ─ stat() 拿 st_dev               │
│                         │   │   ─ 写 /sys/class/bdi/maj:min/...  │
│                         │   │   ─ 失败 WARN 不抛错               │
└─────────────────────────┘   └────────────────────────────────────┘
```

## 2. 涉及文件 & 变更摘要

| # | 文件 | 变更类型 | 说明 |
|---|---|---|---|
| F1 | `curvine-common/src/conf/fuse_conf.rs` | 修改 | `FuseConf` 增加字段、`Default` 实现、`init()` 校验 |
| F2 | `etc/curvine-cluster.toml` | 修改 | `[fuse]` 段增加注释样例 |
| F3 | `curvine-fuse/src/session/bdi.rs` | 新增 | 单独模块封装"由挂载路径解析 BDI 路径并写入" |
| F4 | `curvine-fuse/src/session/mod.rs` | 修改 | `pub mod bdi;` |
| F5 | `curvine-fuse/src/session/fuse_session.rs` | 修改 | `setup_mnts` 完成后调用 `bdi::apply_max_readahead_kb` |
| F6 | `curvine-fuse/src/fs/curvine_file_system.rs` | 修改 | `init` op 中根据 `conf.max_readahead_kb` 调高 `fuse_init_out.max_readahead` |
| F7 | `curvine-fuse/src/session/bdi.rs`（同 F3，含 `#[cfg(test)]`） | 新增 | 单测：路径推导 / sysfs 写入失败软降级 |

## 3. 详细设计

### 3.1 配置项（F1）

```rust
// curvine-common/src/conf/fuse_conf.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(default)]
pub struct FuseConf {
    // ... existing fields ...

    /// Optional override for the FUSE mount BDI `max_readahead_kb` (KB).
    /// When set (recommended: 1024 = 1 MiB), curvine-fuse will write the value
    /// to `/sys/class/bdi/<major>:<minor>/max_readahead_kb` after each mount,
    /// and bump the FUSE init `max_readahead` to at least `value * 1024` bytes,
    /// so the kernel can issue larger sequential read requests.
    /// `None` (default) keeps current behavior.
    pub max_readahead_kb: Option<u32>,
}
```

`Default::default()` 中：`max_readahead_kb: None`。

`init()` 中追加：

```rust
if let Some(v) = self.max_readahead_kb {
    if v == 0 {
        return err_box!("fuse.max_readahead_kb must be > 0 if set");
    }
}
```

### 3.2 BDI 写入模块（F3）

新增 `curvine-fuse/src/session/bdi.rs`：

```rust
// curvine-fuse/src/session/bdi.rs
//
// Linux-only helpers to override the FUSE mount BDI's max_readahead_kb.

#[cfg(target_os = "linux")]
use std::os::unix::fs::MetadataExt;
use std::path::Path;

use log::{info, warn};

#[cfg(target_os = "linux")]
pub fn apply_max_readahead_kb(mnt_path: &Path, kb: u32) {
    let dev = match std::fs::metadata(mnt_path) {
        Ok(meta) => meta.st_dev(),
        Err(e) => {
            warn!(
                "bdi max_readahead_kb skip: stat({:?}) failed: {}",
                mnt_path, e
            );
            return;
        }
    };
    // st_dev encodes major/minor per <sys/sysmacros.h>
    let major = unsafe { libc::major(dev) };
    let minor = unsafe { libc::minor(dev) };
    let bdi_path = format!("/sys/class/bdi/{}:{}/max_readahead_kb", major, minor);

    match std::fs::write(&bdi_path, kb.to_string()) {
        Ok(()) => info!(
            "bdi max_readahead_kb set: path={:?}, bdi={}:{}, value={}",
            mnt_path, major, minor, kb
        ),
        Err(e) => warn!(
            "bdi max_readahead_kb skip: write {} failed: {} (mount continues)",
            bdi_path, e
        ),
    }
}

#[cfg(not(target_os = "linux"))]
pub fn apply_max_readahead_kb(_mnt_path: &Path, _kb: u32) {
    // No-op on non-Linux platforms; sysfs/BDI is Linux-only.
}
```

### 3.3 在 session 启动期挂入（F5）

`fuse_session.rs::setup_mnts` 完成后，对每个 mnt 调用一次：

```rust
// curvine-fuse/src/session/fuse_session.rs
let mnts = Self::setup_mnts(&conf, &fs).await?;

if let Some(kb) = conf.max_readahead_kb {
    for mnt in &mnts {
        crate::session::bdi::apply_max_readahead_kb(&mnt.path, kb);
    }
}
```

放置时点说明：

- `setup_mnts` 内部会走 `FuseMnt::new` → `fuse_mount_pure` → 真实 `mount(2)`，等其全部成功之后 sysfs 路径才出现。
- `restore` 分支同样会把 `mnt.path` 填齐，因此 fork-on-reload 重启场景也覆盖。

### 3.4 调高 FUSE init `max_readahead`（F6）

`curvine_file_system.rs:async fn init` 中 `fuse_init_out` 构造前插入：

```rust
let kernel_ra = op.arg.max_readahead;
let target_ra = match self.conf.max_readahead_kb {
    Some(kb) => kernel_ra.max(kb.saturating_mul(1024)),
    None     => kernel_ra,
};

let out = fuse_init_out {
    // ...
    max_readahead: target_ra,
    // ...
};
```

> 说明：`fuse_conn.max_readahead` 是 FUSE 内核与 daemon 协商出的上限，BDI 真正生效的值是 `min(bdi.max_readahead_kb, fuse_conn.max_readahead)`。把它抬高到 ≥ 配置值，配合 §3.2 的 sysfs 写入，才能让 1 MiB 真正落地。

### 3.5 配置样例（F2）

`etc/curvine-cluster.toml` 的 `[fuse]` 段追加：

```toml
[fuse]
# Optional. Override the FUSE mount BDI `max_readahead_kb` after mount.
# Recommended: 1024 (= 1 MiB) for sequential read workloads.
# Leave commented out to keep kernel default (typically 128 KB).
# max_readahead_kb = 1024
```

## 4. 测试策略

### 4.1 单元测试

`curvine-fuse/src/session/bdi.rs` 内 `#[cfg(test)] mod tests`：

- `apply_max_readahead_kb_no_panic_on_invalid_path`：传入 `/nonexistent`，期望函数返回（不 panic），日志含 WARN 关键词。
- 路径推导：将 `libc::major/minor` 解码逻辑抽出 `fn dev_to_bdi_path(dev: u64) -> String`，单独测 `(8, 1) → /sys/class/bdi/8:1/max_readahead_kb`。

### 4.2 集成测试（手工 / regression）

走 `curvine-tests/regression/build-server.py` 的 FUSE 套件：

1. 启动单机集群，`max_readahead_kb = 1024`；
2. `mount` 后断言 `/sys/class/bdi/<dev>/max_readahead_kb == 1024`；
3. `fio` 顺序读，对比 baseline（`max_readahead_kb = None`）的 IOPS 与请求数；
4. 关闭配置回归默认行为，断言数值与 baseline 完全一致。

### 4.3 矩阵

| 配置 | 平台 | 期望 |
|---|---|---|
| 未设置 | Linux | 行为不变 |
| `= 1024` | Linux + 有写权限 | sysfs 生效，INFO 日志 |
| `= 1024` | Linux + sysfs 只读（容器） | 启动成功，WARN 日志 |
| `= 0` | 任意 | 启动期校验报错退出 |
| `= 1024` | macOS | 编译通过，no-op，无日志 |

## 5. 上线 & 回滚

- **灰度**：默认 `None` 即原地行为，无需灰度开关；用户按需在 `[fuse]` 中加一行。
- **回滚**：注释掉配置或改为 `None`，重启 FUSE 进程即可。BDI 值会随挂载点 unmount 自然回落（sysfs 目录消失）。

## 6. 文档

- 在 PR 中同步更新 `etc/curvine-cluster.toml` 注释；
- 在 [docs/quick-start-guide.md](../../docs/quick-start-guide.md)（若需要）的 FUSE 性能调优段落补一句；
- PR 描述中给出 fio 前后对比数据。
