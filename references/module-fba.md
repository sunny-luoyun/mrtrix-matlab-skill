# fba — FBA 纤维分析模块

## 用途

Fixel-Based Analysis（基于固定点的纤维分析）的标准 21 步流程。用于全脑体素水平的纤维特异性分析。

## 主界面

| 函数 | 说明 |
|------|------|
| `fba.m` | FBA 主界面，4 个按钮：个体处理 / 模版构建 / Fixel 分析 / 统计分析 |
| `fba_subject.m` | 个体处理界面（step 1-6） |
| `fba_template.m` | 模版构建界面（step 7） |
| `fba_fixel.m` | Fixel 分析界面（step 8-19） |
| `fba_stats.m` | 统计分析界面（step 20） |

## 21 步流程详解

### 个体处理（步骤 1-6）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 1 | `step1_resp.m` | `dwi2response` | 复制 DWI，估计响应函数（dhollander/tournier） |
| 2 | `step2_respmean.m` | `responsemean` | 计算全组平均响应函数（WM/GM/CSF） |
| 3 | `step3_upsample.m` | `mrgrid regrid` | DWI 上采样到指定体素大小（默认 1.25mm） |
| 4 | `step4_mask.m` | `dwi2mask` | 为上采样 DWI 创建脑掩膜 |
| 5 | `step5_csd.m` | `dwi2fod msmt_csd` / `dwi2fod csd` | 计算 FOD |
| 6 | `step6_normalise.m` | `mtnormalise` | FOD 标准化 |

### 模版构建（步骤 7）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 7 | `step7_template.m` | `population_template` | 构建群体 FOD 模版，选择全部或部分被试 |

### 配准（步骤 8-9）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 8 | `step8_register.m` | `mrregister` | 每个被试 FOD 非线性配准到群体模版 |
| 9 | `step9_mask_inter.m` | — | 个体掩膜变换到模版空间，取交集 |

### Fixel 分析（步骤 10-16）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 10 | `step10_fixel_mask.m` | `fod2fixel` | 从模版 FOD 生成 fixel 掩膜 |
| 11 | `step11_warp_fod.m` | `mrtransform` | 个体 FOD 变换到模版空间 |
| 12 | `step12_fd.m` | `fod2fixel -afd` | 计算 Fiber Density (FD) |
| 13 | `step13_reorient.m` | `fixelreorient` | Fixel 方向重定向 |
| 14 | `step14_corresp.m` | `fixelcorrespondence` | 建立个体与模版 fixel 对应关系 |
| 15 | `step15_fc.m` | `warp2metric -fc` | 计算 Fiber Cross-section (FC) |
| 16 | `step16_log_fdc.m` | — | 计算 log(FC) 和 FDC = FD × FC |

### 纤维追踪与连接（步骤 17-18）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 17 | `step17_tckgen.m` | `tckgen` + `tcksift` | 模版空间纤维追踪 + SIFT |
| 18 | `step18_connect.m` | `fixelconnectivity` | 从纤维束构建 fixel 连接矩阵 |

### 平滑与统计（步骤 19-20）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 19 | `step19_smooth.m` | `fixelfilter smooth` | 对 FD/log_FC/FDC 进行平滑 |
| 20 | `step20_stats.m` | `fixelcfestats` | 基于 fixel 的置换检验统计分析 |

### 可视化（步骤 21）

| 步骤 | 函数 | 命令 | 说明 |
|------|------|------|------|
| 21 | `step21_view.m` | `mrview` | 可视化 FOD 模版 |

## 关键参数

- **上采样体素大小**：默认 1.25mm（与 Brainnetome 图谱匹配）
- **CSD 算法**：msmt_csd 或 csd
- **响应函数算法**：dhollander（推荐，多组织）
- **置换检验次数**：默认 5000 次
- **平滑核大小**：默认 10mm

## 需要注意

- FBA 是独立流程，有自己的预处理子流程，不依赖 pre 模块
- 但依赖 FOD 计算（可在步骤 1 中完成，也可使用已有的 fod 模块结果）
- 步骤 7 模版构建耗时，建议使用服务器
- 步骤 20 统计检验耗时，置换次数越多越准确但越慢
