# Tasks: 支持配置 FUSE BDI `max_readahead_kb`

- **Spec**: [./spec.md](./spec.md)
- **Plan**: [./plan.md](./plan.md)
- **Issue**: [#895](https://github.com/CurvineIO/curvine/issues/895)

> 推荐顺序：T1 → T2 → T3 → T4 → T5 → T6 → T7。T3、T4 可在 T2 之后并行。

---

## T1. 在 `FuseConf` 增加可选字段

**文件**：`curvine-common/src/conf/fuse_conf.rs`

- [ ] 结构体 `FuseConf` 增加 `pub max_readahead_kb: Option<u32>`，附带 doc comment（参见 plan §3.1）。
- [ ] `Default::default()` 中加 `max_readahead_kb: None`。
- [ ] `init()` 中加入 `Some(0)` 校验，返回 `err_box!` 错误。
- [ ] 跑一次 `cargo check -p curvine-common`，确认无编译错误。

**验收**：默认配置下序列化/反序列化保持兼容（旧 toml 仍可加载）。

---

## T2. 新增 BDI 写入辅助模块

**文件**：`curvine-fuse/src/session/bdi.rs`（新建）、`curvine-fuse/src/session/mod.rs`（追加 `pub mod bdi;`）

- [ ] 实现 `pub fn apply_max_readahead_kb(mnt_path: &Path, kb: u32)`，按 plan §3.2 写完两套 cfg 分支（Linux 实写，其它平台 no-op）。
- [ ] 用 `log::{info, warn}`，确保软失败：任何 stat / write 失败仅打 WARN 不返回错误。
- [ ] 在文件末尾加一个 `#[cfg(test)] mod tests`，至少包含一个"传入不存在路径不 panic"的用例。

**验收**：`cargo build -p curvine-fuse` 通过；`cargo test -p curvine-fuse session::bdi` 通过。

---

## T3. 在 FuseSession 启动期挂入 BDI 写入

**文件**：`curvine-fuse/src/session/fuse_session.rs`

- [ ] 在 `FuseSession::new` 中、`Self::setup_mnts(&conf, &fs).await?` 之后立即增加：

  ```rust
  if let Some(kb) = conf.max_readahead_kb {
      for mnt in &mnts {
          crate::session::bdi::apply_max_readahead_kb(&mnt.path, kb);
      }
  }
  ```

- [ ] 确认 `restore` 分支同样会经过 `FuseMnt::from_fd`，`mnt.path` 已被填值，不需要额外处理。

**验收**：`cargo build -p curvine-fuse` 通过。

---

## T4. 在 FUSE init 协议中抬高 `max_readahead`

**文件**：`curvine-fuse/src/fs/curvine_file_system.rs`

- [ ] 定位 `async fn init(&self, op: Init<'_>) -> FuseResult<fuse_init_out>`（约 596 行起），在构造 `fuse_init_out` 前计算：

  ```rust
  let kernel_ra = op.arg.max_readahead;
  let target_ra = match self.conf.max_readahead_kb {
      Some(kb) => kernel_ra.max(kb.saturating_mul(1024)),
      None     => kernel_ra,
  };
  ```

- [ ] 把 `fuse_init_out { max_readahead: op.arg.max_readahead, ... }` 改为 `max_readahead: target_ra`（line 635）。
- [ ] 同步改 `curvine-fuse/src/fs/test_file_system.rs:58` 的同名字段，避免 mock 对照崩。

**验收**：`cargo test -p curvine-fuse` 全绿。

---

## T5. 更新配置样例与文档

**文件**：`etc/curvine-cluster.toml`

- [ ] `[fuse]` 段尾追加注释样例（plan §3.5）：

  ```toml
  # Optional. See specs/895-fuse-bdi-max-readahead/spec.md
  # max_readahead_kb = 1024
  ```

- [ ] 如有 `docs/` 下面向用户的 FUSE 调优 section（首次入门指南），在适当位置增补一行说明（≤2 句）。

**验收**：`grep -n max_readahead_kb etc/curvine-cluster.toml` 出现一行注释样例。

---

## T6. 集成验证（手工 / regression）

**目标**：覆盖 spec §6 的 AC-1 ~ AC-5。

- [ ] **AC-1 默认行为**：不加配置，跑 `make build ARGS="-p core -p fuse"`，启动 FUSE 后 `cat /sys/class/bdi/<dev>/max_readahead_kb` 数值与升级前一致；`fs_test`、`fuse_test` 全绿。
- [ ] **AC-2 开启后生效**：`max_readahead_kb = 1024`，挂载后断言 sysfs 值 = 1024；`dd if=<file> of=/dev/null bs=1M count=128`，开 `fuse.debug=true` 或 `bpftrace` 数 READ opcode 数量，应较默认下降至约 1/8。
- [ ] **AC-3 容错**：`chmod 444 /sys/class/bdi/<dev>/max_readahead_kb` 后启动 FUSE，应继续成功并出现 WARN。
- [ ] **AC-4 参数合法性**：`max_readahead_kb = 0`，启动应直接退出并打印明确错误。
- [ ] **AC-5 多挂载点**：`mnt_number = 2`，启动日志中出现两条 `bdi max_readahead_kb set` INFO。

**验收**：以上 5 条全部通过，并把日志/数据贴入 PR 描述。

---

## T7. 提交 PR

- [ ] 提交分支 `feat/fuse-bdi-max-readahead`，遵循 [`COMMIT_CONVENTION.md`](../../COMMIT_CONVENTION.md)。
- [ ] PR 标题：`feat(fuse): allow configuring BDI max_readahead_kb to enable 1MiB read requests`。
- [ ] PR 正文链接 issue #895，附 fio 前后对比数据 + 关键日志截图。
- [ ] 在 PR 描述中粘贴本 spec/plan 的链接。

---

## 任务依赖图

```
T1 ──► T2 ──► T3 ──► T6 ──► T7
        └─►   T4 ──┘
              T5 ──┘
```
