# fod — 反卷积响应函数计算模块

## 用途

从预处理后的 DWI 数据估计响应函数，进行球面反卷积（CSD）计算纤维方向分布（FOD），并进行标准化和配准。

## 文件清单

### fod.m（App Designer 主界面）

**入口函数**。运行后选择响应函数算法和 CSD 算法，配置参数并批量计算。

### 响应函数算法

| 函数 | 对应命令 | 适用场景 | 输出 |
|------|---------|---------|------|
| `dhollander.m` | `dwi2response dhollander` | 多组织（WM/GM/CSF） | 3 个响应函数 |
| `tournier.m` | `dwi2response tournier` | 仅 WM，自动选体素 | 1 个响应函数 |
| `fa.m` | `dwi2response fa` | 仅 WM，FA 阈值选体素 | 1 个响应函数 |
| `tax.m` | `dwi2response tax` | WM，Tax 算法 | 1 个响应函数 |
| `msmt_5tt_tournier.m` | 5tt + tournier | 需要 T1corg 的 5tt 结果 | 多组织 |
| `msmt_5tt_fa.m` | 5tt + FA | 需要 T1corg 的 5tt 结果 | 多组织 |
| `msmt_5tt_tax.m` | 5tt + Tax | 需要 T1corg 的 5tt 结果 | 多组织 |

### CSD 算法

| 函数 | 对应命令 | 说明 |
|------|---------|------|
| `csd.m` | `dwi2fod csd` | 单组织 CSD，用 WM 响应函数 |
| `msmt.m` | `dwi2fod msmt_csd` | 多组织 CSD，需 WM/GM/CSF 三个响应函数 |

### 后处理

| 函数 | 对应命令 | 说明 |
|------|---------|------|
| `normal.m` | `mtnormalise` | FOD 强度标准化，组分析必须 |
| `fodtoMNI.m` | `mrtransform` | FOD 线性配准到 MNI152，带 `-reorient_fod yes` |

## 推荐组合

| 数据情况 | 响应函数 | CSD | 标准化 | MNI |
|---------|---------|-----|--------|-----|
| 常规 DWI | dhollander | msmt_csd | 是 | 可选 |
| 无多组织需求 | tournier | csd | 是 | 可选 |
| 有 5tt 分割 | msmt_5tt_tournier | msmt_csd | 是 | 可选 |
