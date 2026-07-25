# Schema 权威定义 (v6.5 · 双层形态 · **manifest-only** · 业界对齐 · CI 双跑)
# BoostKit RaBitQ 单仓样板

> v6.5 master 分支：**manifest 是 patch 元数据单一权威 + 双层形态**。
> - **总**：顶层 `upstream_url/release/pin_commit`（上游身份基线 + release 字段不再写 main 分支），**不含 install/build**
> - **分** (双层):
>   - `series[]` 数组 — 普通 patches（总是 on，按字典序 apply），每条含 `id/file/author/date/upstream_status/notes`，可选 `depends_on/conflicts_with/upstream_pr/merged_commit`
>   - `extras[]` 数组 — 鲲鹏特定性能优化（可单独开关，每个 extra 一个子目录），每条含 `extra_id/title/enabled/author/date/upstream.{upstream_status,notes,...}` + `files[]`
> - **构建命令不在 manifest**：编译是各仓自带脚本的事（业界依据：Buildroot / OpenWrt / Yocto 都分离 patch 与 build，详见 §10 治理边界）
> - **CI 双跑**：series 与 extras 各自独立在 fresh upstream 上 dry-run（路 A 失败 → 跳路 B，节省 CI 时间）
> - **字段 rename** (vs v6.0): `repo→upstream_url, version→release, commit→pin_commit, name→id, path→file, owner→author, status→upstream_status, depends→depends_on, conflicts→conflicts_with, upstream_commit→merged_commit, features→extras`
>
> ⚠ **patch 文件不被 lint 读取**。业务约束：patch 不能被改。
> 若需恢复 patch-header-as-source-of-truth，切 `v6.0-patchheader-status` 分支。

## 目录结构（多版本演进形态 · Buildroot 模式）

```
boostkit-rabitq/
├── Makefile                    # v6.0 可选入口 (thin wrapper 调 go.sh)
├── README.md / README_en.md
├── LICENSE
├── OWNERS
├── tools/
│   ├── go.sh                   # ★ 零 make 依赖主入口 (WSL2/Windows 友好)
│   ├── apply_patch.sh          # Buildroot 风格 (字典序 apply, 自动识别 patch 格式)
│   ├── lint.py                 # Yocto 6 态校验 (manifest-only)
│   └── verify.sh               # 一键 (CI 默认 = lint + dry-run + status)
├── .github/
│   └── workflows/ci.yml        # CI (调 go.sh verify, 8 分钟内)
├── docs/
│   ├── schemas.md              # 本文档
│   ├── zh/                     # 产品文档 (不动)
│   └── en/
└── src/
    └── RaBitQ-Library/         # 一个上游版本一个子目录 (自包含)
        ├── manifest.yaml       # ★ 唯一配置文件（总 + series + extras）
        ├── README.md           # 版本说明 + 加新版本步骤
        ├── series/             # 普通 patches (总是 on, 字典序 apply); 当前为空
        │   └── .gitkeep
        └── extras/             # 鲲鹏特定性能优化 (可单独开关, 每个 extra 一个子目录)
            ├── neq/
            │   └── 0001-neon-simd.patch
            └── eqv/
                └── 0001-soar.patch
```

**入口优先级**：
1. `bash tools/go.sh <cmd>` — 零依赖主入口（WSL2 / Windows / 裸 Linux）
2. `make <target>` — 可选 wrapper（有 make 时）
3. CI 直接调 `bash tools/go.sh verify`，**不依赖 make**

**未来加版本**（如 RaBitQ-Library v2）：
```bash
cp -r src/RaBitQ-Library src/RaBitQ-Library-v2
# 改 src/RaBitQ-Library-v2/manifest.yaml: version/commit/patches
# 加新 patch 进 src/RaBitQ-Library-v2/ (跟 manifest 同目录)
bash tools/go.sh verify    # 自动发现 src/*/manifest.yaml
```

**关键约定**：
- `src/<V>/` **自包含**（manifest + patch + 编译配置），可整体 `cp -r` 搬到新版本
- `patches[].path` 字段相对 **manifest.yaml 所在目录**，不是仓根
- 仓根永远干净：README / Makefile / tools / docs / src/
- 业界参考：**Buildroot** `package/<name>/`、`OpenWrt` `package/<name>/`

## 1. 总：manifest 顶层（上游身份 · v6.5 rename 后）

| 字段 (v6.5) | 类型 | 必填 | 旧字段 (v6.0) | 语义 |
|------|------|:--:|------|------|
| `upstream_url` | URL | 是 | `repo` | 上游 git URL |
| `release` | string | 是 | `version` | **上游真实 tag (推荐)** 或 `snapshot-YYYY-MM-DD` (**禁止**写 `main` / `develop` / `master` 分支名) |
| `pin_commit` | 40-char SHA | 是 | `commit` | immutable pin — 与 release 对应的固定 commit |
| ~~`install`~~ | ~~dict~~ | ~~否~~ | — | **v6.0 整改后移除**（编译由各仓自带脚本负责，见 §10） |

> **RaBitQ 仓特殊说明 (v6.5)**：上游 `VectorDB-NTU/RaBitQ-Library` 没有稳定 tag，业务原先用 `version: main` 写法。v6.5 后改为 `release: snapshot-2026-07-25` + `pin_commit: 540242e...` 形式——`release` 字段不再漂移。
>
> ⚠ **release 黑名单**：`{main, develop, master, HEAD, trunk, latest}` 全部禁止。lint 校验：写分支名会让 `pin_commit` 与 `release` 失去对应关系，patch 治理无法走双跑 CI。

## 2. 分：双层结构 (series[] + extras[] · Yocto Upstream-Status 6 态)

v6.5 引入双层结构——两类 patch 走两套治理机制：

- **series[]**：通用兼容性 patch，**总是 on**，按字典序 apply，每个 entry 是一个 patch
- **extras[]**：鲲鹏特定性能优化（强硬件依赖、上游不收），**可单独开关**，每个 extra 一个子目录，字典序 apply

CI 跑两路独立验证（双跑）：路 A = series 在 fresh upstream dry-run；路 B = extras 在 fresh upstream dry-run；**不叠加**。

### 2.1 series[] 字段矩阵（普通 patches · 总是 on）

| 字段 (v6.5) | 类型 | 必填 | 联动 `upstream_status` | 旧字段 (v6.0) |
|------|------|:--:|------|------|
| `id` | kebab-case string | 是 | — | `name` |
| `file` | 相对路径 | 是 | — | `path` |
| `author` | email | 是 | — | `owner` |
| `date` | `YYYY-MM-DD` | 是 | — | `date` (语义不变) |
| `upstream_status` | 6 态 enum | 是 | — | `status` |
| `notes` | string ≥10 | 条件 | Inappropriate/Denied/Backport 必填 | `notes` (语义不变) |
| `upstream_pr` | URL | 条件 | Pending/Submitted 必填 | `upstream_pr` (语义不变) |
| `merged_commit` | 40-char SHA | 条件 | Accepted 必填 | `upstream_commit` |
| `depends_on` | list[`series:<id>` \| `<id>`] | 否 | — | `depends` |
| `conflicts_with` | list[string] | 否 | — | `conflicts` |

**series 内允许的 `depends_on`**：
- `<id>` —— 当前 series 内的另一个 entry id
- `series:<id>` —— 显式前缀（推荐，避免歧义）
- **不允许**引用 extras（extra 不能跨层被普通 patch 依赖；如果普通 patch 必须等某个 extra 启用，应迁移到 series）

### 2.2 extras[] 字段矩阵（鲲鹏性能优化 · 可开关）

| 字段 (v6.5) | 类型 | 必填 | 备注 |
|------|------|:--:|------|
| `extra_id` | kebab-case | 是 | 类似 OpenWrt `<package>` / Buildroot `<pkg>`；跨仓全局唯一 |
| `title` | string | 是 | 人类可读描述 |
| `enabled` | bool | 是 | `true` = 默认应用；`false` = 仓库默认禁用；运行时用 `DISABLED_EXTRAS=neq,eqv` env 可覆盖 |
| `self_contained` | bool | 是 | **v6.5 新增**：`true` = 此 extra 是纯 upstream 上可重放的独立补丁（CI 默认 apply/dry-run）；`false` = 依赖下游编译环境 / 上游 build 工具链 / CI 服务，默认 apply/dry-run **自动跳过**（用 `BOOTSTRAP_NON_BUILDABLE=1` 可强制包含）。这是 "特殊 patch" 与 "通用 patch" 的本质区别：标志 `false` 即承认该 patch 无法在干净 upstream 上裸跑 dry-run，需在生产机配齐依赖后再 apply。 |
| `author` | email | 是 | 提交人 |
| `date` | `YYYY-MM-DD` | 是 | 提交日期 |
| `upstream.upstream_status` | 6 态 enum | 是 | extra **整体**的 upstream 状态（**patch 级不独立 status**，避免与 extra 级冲突） |
| `upstream.notes` | string ≥10 | 条件 | Inappropriate/Denied/Backport 必填 |
| `upstream.upstream_pr` | URL | 条件 | Pending/Submitted 必填 |
| `upstream.merged_commit` | 40-char SHA | 条件 | Accepted 必填 |
| `files[]` | list[`{file}`] | 是 | extra 内 patch 文件（**仅含 file 字段**，metadata 由 extra 级统一管理） |

**extra 内 `files[]`**：
- 每个 file 字段就是相对 manifest 所在目录的 path
- 字典序 apply（lint 强制）
- **不允许** `depends_on` / `status`（patch 级无独立 metadata，继承 extra 级）

### 2.3 `upstream_status` 6 态（Yocto `dev-manual/common-tasks` · 应用于 series + extra 两层）

| status | 语义 | manifest 联动必填 | patch header 协同字段 |
|--------|------|-----------------|---------------------|
| `Pending` | 已发上游 PR/邮件，等回复 | `upstream_pr` | `Upstream-Status: Pending` |
| `Submitted` | 上游复核中 | `upstream_pr` | `Upstream-Status: Submitted` |
| `Backport` | 从更高版本反向移植 | `notes`（源头 commit / 新版本号） | `Upstream-Status: Backport` |
| `Denied` | 上游明确拒绝 | `notes`（拒绝原因） | `Upstream-Status: Denied` |
| `Inappropriate` | **不适合上游**（业务强硬件依赖等） | `notes`（不适合原因） | `Upstream-Status: Inappropriate` + `Whitelist-Reason` |
| `Accepted` | 上游已合并 | `merged_commit` | `Upstream-Status: Accepted` + `Upstream-Commit` |

> **字段位置差异**：
> - **series**：字段平铺在 entry 顶层（`entry.upstream_status`, `entry.notes`, ...）
> - **extras**：字段放在 `upstream.` 嵌套块（`extra.upstream.upstream_status`, `extra.upstream.notes`, ...）
> - **extras patch**：无独立 status（继承所属 extra 的 `upstream.upstream_status`）

> **本仓当前状态**：2 patch（neq + eqv）均 `Inappropriate`（鲲鹏 NEON / SOAR / ML-nprobe 上游 NTU 不收），分类在 `extras[]` 下（不强依赖鲲鹏硬件的普通 patch 才是 `series[]`）。

### 2.4 顺序：字典序（Buildroot + OpenWrt 模式）

`apply_patch.sh` 按字典序 apply：

- **series[]**：按 `id` 字典序（lint 强制一致性）
- **extras[]**：按 `extra_id` 字典序
- **extras[i].files[]**：按 `file` 字典序

```
series 字典序:  0001-foo < 0002-bar < 0003-baz
extras 字典序:  eqv < fp16 < neq  (kebab-case lex)
files 字典序:   0001-xxx < 0002-yyy
```

### 2.5 跨层依赖 + 环检测

- series entry 的 `depends_on` 只能引用**其他 series entry**（不允许引用 extras）
- extras 内 patch 无 `depends_on`（依赖关系在 extra 级统一管理）
- DFS 环检测（系列内部 + 跨层 `series:<id>` 引用）
- 环检测同时校验 完整性（引用的 id 必须存在）和 无环（不允许 A→B→A）

> **业务约束**：extras 之间**禁止相互依赖**。如果某个 extra 在功能上必须等另一个 extra，先用 series 引入公共部分，或合并到一个 extra 内。

## 3. Patch 文件（**不校验·不被读取**）

> ⚠ master 分支下：patch 文件**不归 lint 管、不进 CI、不被脚本读**。
> 它们只是 `git apply` 或 `patch -p1` 的输入，业务约束是"patch 不能动"。

### 3.1 支持的 patch 格式（自动检测）

`tools/apply_patch.sh` 在循环里 **head -1** 检测 patch 格式：

| 文件头第一行 | 判定 | apply 命令 |
|--------------|------|-----------|
| `From <40-hex-sha> ...` | `git-format` | `git apply` |
| `diff --git ...` | `git-diff` | `git apply` |
| `diff -...` | `plain-diff` | `patch -p1` |
| 其他 | `unknown` | （报错，需手工处理） |

> **本仓特殊性**：rabitq 的 2 个 patch 是 `diff -uNr` 格式（快照式），路径映射
> `./RaBitQ/data/eval.py → ./rabitq_neq/data/eval.py`，是"代码快照"而非典型 patch。
> apply 时走 `patch -p1`，由 `tools/apply_patch.sh` 自动选择。

如果 patch 头里有 `From / Date / Subject / Description / Upstream-Status / Whitelist-Reason` 等历史字段，**会被忽略**。一旦想恢复"patch 头即治理"的形态，切到 `v6.0-patchheader-status` 分支。

### 3.2 双层目录布局（v6.5）

```
src/RaBitQ-Library/
├── manifest.yaml               # 唯一权威
├── series/                     # 普通 patches (总是 on, 字典序 apply)
│   ├── 0001-foo.patch
│   └── 0002-bar.patch
└── extras/                     # 鲲鹏特定性能优化 (可单独开关, 每个 extra 一个子目录)
    ├── neq/                    # ← extra_id
    │   ├── README.md           # (可选, OpenWrt 风格)
    │   ├── 0001-neon-simd.patch
    │   └── 0002-fp16-lut.patch
    └── eqv/
        ├── README.md           # (可选)
        └── 0001-soar.patch
```

约定：
- **series/<name>.patch** — 与 manifest 中 `series[].file` 对应；目录必须存在（即使为空，留 `.gitkeep`）
- **extras/<extra_id>/** — 每个 extra 一个子目录，子目录名严格等于 `extra_id` (kebab-case)
- **extras/<extra_id>/*.patch** — 与 manifest 中 `extras[extra_id].files[].file` 对应
- patch 文件路径**始终相对 manifest.yaml 所在目录**（Buildroot 模式）
- `src/<V>/` 整体自包含，可 `cp -r src/RaBitQ-Library src/RaBitQ-Library-v2` 演进

## 4. ~~manifest install（可选）~~ — 已废弃

> ⚠ v6.0 整改后**移除 `install` 字段**。
> 整改前 manifest 里有 `install.configure` / `install.build`，由 `apply_patch.sh` apply 后执行；
> 但实际每个 boostkit 仓的构建链路差异极大（autotools / cmake / bazel / python / 跨编译镜像 / 闭源 bin 注入），
> 不可能用两个 string 字段表达。业界三家（Buildroot / OpenWrt / Yocto）都把构建命令放在仓自带脚本里，
> manifest 只管 patch 治理。详见 §10 治理边界。

## 5. 完整模板（本仓实际示例 · v6.5 · `src/RaBitQ-Library/manifest.yaml`）

```yaml
# BoostKit RaBitQ · v6.5 双层形态 (series + extras · CI 双跑 · 字段 rename)
# ============================================================
#
# 形态说明 (master 分支, 对齐 Yocto recipe + Buildroot package/<name>/):
#   - 顶层 (总): 上游身份 baseline (upstream_url/release/pin_commit)
#   - series (分): 普通 patches 数组, 总是 on, 字典序 apply
#   - extras (分): 鲲鹏特定性能优化, 每个 extra 一个子目录, 可单独开关

upstream_url: https://github.com/VectorDB-NTU/RaBitQ-Library
release: snapshot-2026-07-25                          # ★ 改: 旧 version: main 写法禁用
pin_commit: 540242ea0a68926f1b827bf1f9add844f07a427b # ★ 改: 旧 commit

# ─── 普通 patches (series): 总是 on, 字典序 apply ───
# 当前空: 本仓 2 个 patch 都是鲲鹏特定性能优化, 属 extras
series: []                                            # ★ schema 改

# ─── 鲲鹏性能优化 (extras): 可开关, 每个 extra 一个子目录 ───
extras:                                               # ★ schema 改 (was: features)
  - extra_id: neq                                     # ★ 改: kebab-case id
    title: 鲲鹏非等价索引优化
    enabled: true                                     # ★ 新: 默认应用开关
    self_contained: false                             # ★ v6.5 新: 依赖下游编译环境, 默认 apply 跳过
    author: codesheepchen@huawei.com                  # ★ 改: was owner
    date: 2026-02-06
    upstream:
      upstream_status: Inappropriate                  # ★ 改: was status (在 upstream 块下)
      notes: |
        鲲鹏非等价索引优化 (neq): 引入 FP16 精度 + NEON SIMD 向量化
        + 汇编级 LUT 加速, 提升 ARM64 上非等价索引场景性能。
    files:                                            # ★ 改: was patches, 现在只用 file 字段
      - file: extras/neq/0001-neon-simd.patch          # ★ 改: was path

  - extra_id: eqv
    title: 鲲鹏等价索引优化
    enabled: true
    self_contained: false                             # ★ v6.5 新: 同 neq
    author: codesheepchen@huawei.com
    date: 2026-02-25
    upstream:
      upstream_status: Inappropriate
      notes: |
        鲲鹏等价索引优化 (eqv): SOAR 溢出向量分配 + ML 自适应 nprobe,
        在 neq 基础上等价索引场景性能进一步提升。
    files:
      - file: extras/eqv/0001-soar.patch

# ★ v6.5 关键约束:
#   1. release 字段禁止写 main/develop/master 等分支名
#   2. extras 必须能在纯 upstream 上 apply (CI 双跑路 B)
#   3. 跨 extras 引用必须用 series:<id> 前缀 (本仓无 series, 暂无跨层依赖)
```

> ⚠ `series[].file` 与 `extras[].files[].file` 字段**始终相对 manifest.yaml 所在目录**（即 `src/<V>/` 内），
> 这样 `tools/apply_patch.sh` 与 `tools/lint.py` 解析逻辑统一，
> `src/<V>/` 自包含可整体 `cp -r` 搬运。

## 6. 已知 patch 状态（apply dry-run 实测 · v6.5 双层）

| layer | extra_id / id | file | 格式 | 状态 | 说明 |
|-------|---------------|------|------|------|------|
| extras | `neq` | `extras/neq/0001-neon-simd.patch` | plain-diff | ✗ 引用路径不在上游 | patch 引用 `./RaBitQ/data/eval.py` 等路径，但 `VectorDB-NTU/RaBitQ-Library` 仓根**没有 `RaBitQ/` 子目录**，与上游结构不一致 |
| extras | `eqv` | `extras/eqv/0001-soar.patch` | plain-diff | ✗ 同上 + 依赖 neq | 同上；强依赖 neq 成功才能 apply |
| series | — | — | — | — | 本仓 series=空 |

> 这是**真实的治理发现**：样板正确检测出 patch 自身与上游结构不匹配。
> 修复路径：要么重新生成快照（用 apply 后的 working tree），要么上游本来就该有 `RaBitQ/` 子目录。
>
> **v6.5 vs v6.0 差异**：原 2 patch 在 v6.0 都在 `src/<V>/` 仓根平铺；v6.5 迁入 `extras/<id>/` 子目录，反映"这两 patch 实质是 extras 而非通用 series"的分类。

## 7. 校验矩阵（master = manifest-only · v6.5 双跑）

| 校验项 | 命令 | 退出码 | 频率 |
|--------|------|:------:|------|
| **CI 必跑**：manifest 双层 (series+extras) 字段校验 | `bash tools/go.sh lint` | 0/1 | 每 PR |
| **CI 必跑**：series dry-run (路 A, 验可重放) | `bash tools/go.sh apply-series` (设置 `DRY_RUN=1`) | 0/1 | 每 PR |
| **CI 必跑**：extras dry-run (路 B, 验可重放) | `bash tools/go.sh apply-extras` (设置 `DRY_RUN=1`) | 0/1 | 每 PR |
| **CI 必跑**：lint + 双跑 dry-run + status | `bash tools/go.sh verify` | 0/1 | 每 PR |
| 顺序 apply 真写入（开发者本地, series+extras） | `bash tools/go.sh apply` | 0/1 | 开发者本地 |
| 只 apply series | `bash tools/go.sh apply-series` | 0/1 | 开发者本地 |
| 只 apply enabled extras | `bash tools/go.sh apply-extras` | 0/1 | 开发者本地 |
| 跨 layer 灵活命中 (auto-detect) | `bash tools/go.sh apply-layer <name>` | 0/1 | 开发者本地 |
| 运行时禁用某 extras (env) | `DISABLED_EXTRAS=neq,eqv bash tools/go.sh apply` | 0/1 | 开发者本地 / CI 调试 |
| 真编译 | **仓自带脚本**（不在样板内） | — | 开发者本地 / 抽样 CI |
| patch 头 DEP-3 | ~~`python3 tools/lint.py headers ...`~~ | （master 下 no-op，提示切备选分支） | — |

## 8. 业界出处映射（v6.5 双层）

| 维度 | v6.5 双层形态 (manifest-only) | 业界出处 |
|------|------------------------------|---------|
| manifest 顶层（上游身份） | `upstream_url/release/pin_commit`（**无 install**） | **Yocto** recipe (SUMMARY/LICENSE/SRC_URI) |
| manifest 双层 | `series[]` + `extras[]` | **OpenWrt** `patches/series` + **Yocto** `SRC_URI` 多组 extras |
| series 字段 (patch 元数据) | `series[].{id,file,author,date,upstream_status,notes,...}` | **Yocto** Upstream-Status + OpenWrt Config.in |
| extras 字段 | `extras[].{extra_id,title,enabled,upstream.status,files[]}` | **Buildroot** `<pkg>_OVERRIDDEN_SOURCES` + **OpenWrt** `<PACKAGE>-<variant>` |
| extra 开关 | `enabled` 字段 + `DISABLED_EXTRAS` env | **Yocto** `DISTRO_FEATURES` / **Buildroot** `BR2_PACKAGE_*` |
| status 6 态 | Pending/Submitted/Backport/Denied/Inappropriate/Accepted | **Yocto** `dev-manual/common-tasks.html#patches` |
| patch 顺序：字典序 0001-0002 | `apply_patch.sh` 字典序遍历 | **Buildroot** `package/<name>/*.patch` / **OpenWrt** `patches/series` |
| patch 头校验 | **不校验**（业务约束：patch 不能动） | （master 形态独有；Yocto 强制但可选） |
| patch 可重放是独立 stage | `tools/apply_patch.sh` (单 stage) | **Buildroot** `make <pkg>-patch` / **Yocto** `do_patch` / **OpenWrt** `Build/Patch/Default` |
| CI 双跑 (series + extras 独立) | `verify.sh` 路 A + 路 B | **OpenWrt** `package/<name>` 中 `PATCHES` (普通) + `EXTRA_PATCHES` (variant) |
| 依赖解析 | `depends_on` (仅 series 内, 加 `series:` 前缀) + DFS 环检测 | **Linux Kconfig** `depends on` |
| 冲突检查 | `conflicts_with` 集合 | **Kconfig** `depends on !` / **Debian** `Conflicts:` |
| 编译命令载体 | **仓自带 build.sh / Makefile**（不在 manifest） | **Buildroot** `<pkg>.mk` / **OpenWrt** `package/<pkg>/Makefile` / **Yocto** `<recipe>.bb` |
| 入口脚本 (go.sh) | 零 make 依赖 + 双层子命令 (apply-series, apply-extras, apply-layer) | **Kubernetes** `hack/make-rules` + 零依赖兜底 |
| 文档分层 | README → manifest → schemas → docs/zh/en | **Linux kernel** `Kconfig` + `Documentation/` |

## 9. 版本沿革

| 版本 | 形态 | 说明 |
|------|------|------|
| 原始仓 | 平面 patch | `0001-*.patch` `0002-*.patch` 在仓根，无治理元数据 |
| v6.0 早 | 总分形态 + install 字段 | manifest 含 `install.configure` / `install.build`，但实际各仓构建差异极大，install 字段装不下 |
| v6.0 master | 总分形态 (manifest-only) · 业界对齐 | patch 文件**完全不被校验**；manifest `patches[]` 是单一权威；**无 install 字段**；patch 可重放是 CI 必跑；编译由各仓自带 |
| `v6.0-patchheader-status` 备选 | 总分形态（patch-header-source） | 同一套总分 schema，但 patch 头 `Upstream-Status` 重新作为权威源，manifest `patches[].status` 从 patch 头同步读取校验 |
| **v6.5 master 当前** | **双层形态 (series + extras · CI 双跑 · 字段 rename)** | ① 拆分为 series[] (普通 on) + extras[] (可开关)；② 顶层字段 rename `repo→upstream_url`, `version→release`, `commit→pin_commit`；③ `patches[]` 内部字段 rename `name→id/extra_id`, `path→file`, `owner→author`, `status→upstream_status`, `depends→depends_on`, `conflicts→conflicts_with`, `upstream_commit→merged_commit`；④ CI 跑双跑 dry-run (路 A=series, 路 B=extras)；⑤ `version: main` 不允许写，强制 `release: <tag|snapshot>`；⑥ `tools/_patches.py` Python helper 承担重逻辑, `apply_patch.sh` 简化到 ~80 行 (Buildroot 风格)；⑦ `DISABLED_EXTRAS` env 取代 `--disable-extra` CLI |

## 10. 治理边界（关键）

> ★ **样板只负责 patch 治理，编译是各仓自带脚本的事。**

| 样板负责 (v6.5) | 样板不负责 |
|----------|-----------|
| patch 元数据审计（Yocto 6 态 + 字段联动 + 顺序 + 文件存在）— 双层 series+extras | 任何构建命令（autotools / cmake / bazel / python / AOSP / 内核） |
| extra 开关管理（`enabled` 默认值 + `DISABLED_EXTRAS` env 覆盖） | 编译依赖安装（apt/yum/pip/闭源 bin 注入） |
| patch 可重放验证（拉上游 pin commit + 顺序 apply --check；**双跑**: series + extras 各自独立） | 跨编译（AOSP arm64 / 内核 aarch64 / GPU 派发） |
| patch 状态分布报表（双层） | 测试（单元 / 集成 / 性能 / 硬件 gating） |
| 跨层 `depends_on: series:<id>` 完整性 + 环检测 | |
| `release` 黑名单（不允许 `main` 等分支名，避免 pin 漂移） | |

**业界依据**：Buildroot `<pkg>.mk` 的 `FOO_BUILD_CMDS` / OpenWrt `Build/Compile` / Yocto `do_compile` 都是**仓自带**，三家都没有把构建命令塞到 manifest 里。

**extra 开关权责**：
- **业务方**（鲲鹏调优方）：定义 `extra_id` (kebab-case) 与 `enabled` 默认值（`true` 表示默认应用；`false` 表示该 extra 不在主线交付）
- **运行时**：`DISABLED_EXTRAS=neq,eqv` env 临时禁用某些 extra（临时调试、CI 矩阵、跨平台 gate 验证）

**对开发者的含义**：
- `bash tools/go.sh verify` 通过 = patch 元数据 OK + series 与 extras 都能在各自 fresh upstream 上重放。
- 真编译请用仓自带入口（`build.sh all` / `make` / `bazel build` / `mvn clean install` / AOSP `source build/envsetup.sh && lunch ...`）。
- 编译失败 → 找仓自带脚本作者，**不是** patch 治理问题。

**对 CI 含义**：
- 本 CI 跑 `verify` = lint + **apply --check 双跑** + status，**不**跑真编译。
- 双跑结构：路 A (series dry-run) → 路 B (extras dry-run)；路 A 失败时**跳过路 B 节省 CI 时间**，但 err 计数保持。
- 真编译由各仓 CI 自行负责（autobuilder / buildbot / Valkyrie 模式——抽样而非全量）。
- **缓存**：`actions/cache@v4` 缓存 `/tmp/rabitq-build/upstream` clone，key = manifest hash，避免重复 deep clone。