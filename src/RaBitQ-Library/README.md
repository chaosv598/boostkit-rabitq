# RaBitQ-Library — BoostKit 鲲鹏适配 (v6.5 · 双层形态)

本目录管理 **RaBitQ (NTU SIGMOD 2024)** 的 2 个鲲鹏特定 patch，**自包含**（manifest + patch 在本目录，按 v6.5 双层形态组织）。

> ★ **本目录不含 install/build 字段**。编译由各仓自带脚本负责（业界依据：Buildroot / OpenWrt / Yocto 都分离 patch 治理与编译）。详见仓根 [docs/schemas.md §10](../../docs/schemas.md)。

## 与其他版本共存

```
src/
├── RaBitQ-Library/    ← 本目录 (当前活跃)
│   ├── manifest.yaml          ← ★ series[] + extras[] 双层
│   ├── README.md
│   ├── series/                ← 普通 patches (当前为空, 本仓 2 patch 属 extras)
│   │   └── .gitkeep
│   └── extras/                ← 鲲鹏特定性能优化 (可单独开关)
│       ├── neq/
│       │   └── 0001-neon-simd.patch    (diff -uNr 快照式)
│       └── eqv/
│           └── 0001-soar.patch         (diff -uNr 快照式)
└── RaBitQ-Library-v2/ ← 未来版本, 复制本目录即可
```

## v6.5 双层 schema 字段速查

```yaml
upstream_url: https://github.com/VectorDB-NTU/RaBitQ-Library   # ★ was: repo
release: snapshot-2026-07-25                                    # ★ was: version (不再写 main 分支名)
pin_commit: 540242ea0a68926f1b827bf1f9add844f07a427b           # ★ was: commit (40-char SHA pin)

# ─── 普通 patches (series): 总是 on, 字典序 apply ───
# 当前空: 本仓 2 个 patch 都是鲲鹏特定性能优化, 属 extras
series: []

# ─── 鲲鹏性能优化 (extras): 可开关, 每个 extra 一个子目录 ───
extras:                                                         # ★ was: features
  - extra_id: neq                                               # ★ was: name
    title: 鲲鹏非等价索引优化
    enabled: true                                               # ★ 新: 默认应用开关
    self_contained: false                                       # ★ v6.5 新: false=依赖下游, CI apply/dry-run 默认跳过 (BOOTSTRAP_NON_BUILDABLE=1 override)
    author: codesheepchen@huawei.com                            # ★ was: owner
    date: 2026-02-06
    upstream:
      upstream_status: Inappropriate                            # ★ was: status (字段位置改)
      notes: |
        鲲鹏非等价索引优化 (neq): 引入 FP16 精度 + NEON SIMD 向量化
        + 汇编级 LUT 加速, 提升 ARM64 上非等价索引场景性能。
        上游 NTU 仅支持 x86_64 AVX2, 鲲鹏特定 NEON 优化上游不收。
    files:                                                      # ★ was: patches
      - file: extras/neq/0001-neon-simd.patch                    # ★ was: path

# ★ 无 install 字段. 编译请用上游 VectorDB-NTU/RaBitQ-Library 自带构建方式.
```

> **路径约定**：`series[].file` 与 `extras[].files[].file` 相对**本目录**，不相对仓根。
> 这样 `tools/apply_patch.sh` 与 `tools/lint.py` 解析逻辑统一。
> **src/<V>/ 自包含**，可整体 `cp -r` 搬到新版本。

## patch 格式说明

本仓 2 个 patch 是 **`diff -uNr` 格式**（不是 `git format-patch`），路径映射：
```
./RaBitQ/data/eval.py  →  ./rabitq_neq/data/eval.py
./RaBitQ/data/ivf.py   →  ./rabitq_neq/data/ivf.py
...
```

这是**代码快照**而非典型 patch（上游文件标记 `-`，鲲鹏新文件全 `+`）。

apply 时使用 `patch -p1`（不是 `git apply`），具体由 `tools/apply_patch.sh` 自动选择。

## v6.5 本版本 patch 清单 (2 extras · 1 series demo)

| layer | extra_id | file | 特性 | 行数 | self_contained | 默认 apply |
|-------|----------|------|------|:--:|:--:|:--:|
| extras | `neq` | `extras/neq/0001-neon-simd.patch` | 非等价索引优化 (NEON + FP16 + LUT) | 6470 | `false` | ⏭ skip |
| extras | `eqv` | `extras/eqv/0001-soar.patch` | 等价索引优化 (SOAR + ML nprobe) | 4302 | `false` | ⏭ skip |
| series | `0001-series-fake` | `series/0001-series-fake.patch` | (demo · 给 example.sh 加一行注释) | 10 | (总是 in) | ✓ apply |

> **v6.5 vs v6.0 变化**：原 2 patch 在 v6.0 都在 src/<V>/ 仓根平铺并归为 series；v6.5 重新分类为 extras，并迁入 extras/<extra_id>/ 子目录，反映"这两 patch 实质是鲲鹏特定 extras 而非通用 series"。

## 加新版本步骤 (cp-r 整目录)

```bash
# 1. 复制本目录 (manifest + series + extras 一起搬)
cp -r src/RaBitQ-Library src/RaBitQ-Library-v2
```

## 加新 extra 步骤 (鲲鹏特定优化)

```bash
# 1. 建 extra 子目录 (子目录名 = extra_id, kebab-case)
mkdir -p src/RaBitQ-Library/extras/<my-extra-id>
cp ../upstream-patches/<my-extra>/0001-*.patch src/RaBitQ-Library/extras/<my-extra-id>/

# 2. 在 manifest.yaml 的 extras[] 数组追加:
#    - extra_id: <my-extra-id>
#      title: ...
#      enabled: true
#      self_contained: true|false   # ★ v6.5 新: true=通用 patch 可裸重放, false=依赖 CI 下游环境 (CI 默认跳过)
#      author: ...
#      date: YYYY-MM-DD
#      upstream:
#        upstream_status: Inappropriate  # 或 Pending/Submitted 看是否发上游
#        notes: |
#          ...
#      files:
#        - file: extras/<my-extra-id>/0001-*.patch
#
# 3. 验证 (CI 双跑: series dry-run + extras dry-run)
bash tools/apply_patch.sh verify  # CI 默认: lint + 双跑 dry-run

# self_contained 取值指引:
#   true   = 纯 patch 文件 + 通用工具链就能 apply (e.g. 一个跨平台通用补丁)
#   false  = 依赖下游编译环境 / 上游 build 链 / CI 服务才能跑通
#            (本仓 neq / eqv 就是 false: 它们是 diff -uNr 模块快照,
#             需要 RabitQ 完整 build 链 + 鲲鹏编译工具)
# 用 BOOTSTRAP_NON_BUILDABLE=1 可强制 apply self_contained=false 的 extra
```

## 加新 series patch 步骤 (通用 patch, 总是 on)

```bash
# 1. patch 文件放到 series/ (与 manifest 同目录)
cp ../upstream-patches/0001-foo.patch src/RaBitQ-Library/series/

# 2. 在 manifest.yaml 的 series[] 数组追加:
#    - id: 0001-foo
#      file: series/0001-foo.patch
#      author: ...
#      date: YYYY-MM-DD
#      upstream_status: ...
#      notes: |
#        ...
#      depends_on: []    # 可选, 引用其他 series<id> 或 'series:<id>'
#
# 3. 验证
bash tools/apply_patch.sh verify  # CI 默认: lint + 双跑 dry-run
```
