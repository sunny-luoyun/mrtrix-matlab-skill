---
name: mrtrix3-cli
description: >-
  MRtrix3 扩散磁共振图像处理 CLI 技能。直接在终端中运行 MRtrix3 命令进行数据预处理、DTI 指标计算、FOD/CSD 反卷积、
  纤维追踪、FBA 纤维分析、连接组矩阵构建和统计分析。用户提及弥散/扩散/dMRI/DTI/DKI/FBA/FOD/纤维/MRtrix3 时自动触发。
---

# MRtrix3 CLI 扩散磁共振图像处理技能

## 环境检测

开始处理前，先确认 MRtrix3 可用：

```bash
# 检查版本
mrinfo --version

# 确认关键命令可用
which mrconvert dwi2tensor dwi2fod tckgen
```

如果命令找不到，提示用户安装 MRtrix3（https://www.mrtrix.org/download/）。

## 数据组织约定与输入格式

处理前先与用户确认数据格式和目录结构。

### 格式检测

判断方式：询问用户数据文件后缀，或直接 `ls` 查看目录内容。

| 格式 | 特征 | 处理方式 |
|------|------|---------|
| `.dcm` / `.IMA` / DICOM 目录 | 目录内含大量 `.dcm` 文件 | `mrconvert dicom_dir/ dwi.mif` 一步到位 |
| `.nii` / `.nii.gz` | 含 `bvecs` `bvals` 文件 | `mrconvert dwi.nii.gz dwi.mif -fslgrad bvecs bvals` |
| `.mif` / `.mif.gz` | MRtrix3 原生格式，无需转换 | 直接使用 |

### NIfTI 输入推荐结构

```
workdir/
├── sub-001/
│   ├── dwi.nii.gz          # DWI 数据
│   ├── bvecs               # 梯度方向
│   ├── bvals                # b 值
│   ├── T1.nii.gz            # T1 结构像（可选）
│   ├── ap_b0.nii.gz         # AP 相位编码 b0（可选，eddy 用）
│   └── pa_b0.nii.gz         # PA 相位编码 b0（可选，eddy 用）
├── sub-002/
│   └── ...
└── templates/
    └── MNI152.nii.gz        # 标准空间模板
```

### DICOM 输入推荐结构

```
rawdata/
├── sub-001/
│   ├── DICOM/               # 原始 DICOM 目录（含 .dcm 文件）
│   │   ├── IM-0001.dcm
│   │   ├── IM-0002.dcm
│   │   └── ...
│   └── T1_DICOM/            # T1 结构像 DICOM 目录
│       ├── IM-0001.dcm
│       └── ...
├── sub-002/
│   └── ...
└── templates/
    └── MNI152.nii.gz
```

如果数据是散乱的 DICOM 文件夹，先用 `dcminfo` 查看系列信息，再按被试整理。

### 格式转换入口

所有管线开始前，统一执行此格式转换入口：

```bash
# ─────────────────────────────────────────────
# 格式转换统一入口（根据输入格式选择其一）
# ─────────────────────────────────────────────

# 场景 A: DICOM 目录 → .mif（推荐，自动读取梯度表）
mrconvert rawdata/sub-001/DICOM/ sub-001/dwi.mif -datatype float32

# 场景 B: DICOM → 有损压缩 .mif.gz
mrconvert rawdata/sub-001/DICOM/ sub-001/dwi.mif.gz -datatype float32

# 场景 C: NIfTI + bvecs/bvals → .mif
mrconvert rawdata/sub-001/dwi.nii.gz sub-001/dwi.mif \
  -fslgrad rawdata/sub-001/bvecs rawdata/sub-001/bvals \
  -datatype float32

# 场景 D: NIfTI（梯度内嵌于 nii 头文件）
mrconvert rawdata/sub-001/dwi.nii.gz sub-001/dwi.mif -datatype float32

# 场景 E: 已有 .mif 文件，无需转换，直接使用
# (跳过此步, 用 ln -s 或直接 cp)
```

> **注意**：`mrconvert` 读取 DICOM 时会自动解析梯度表（bvecs/bvals/Philips/GE/Siemens 私有头均可），这是最省心的方式。如遇无法解析的情况，再用 `-fslgrad` 手动指定。

## 标准处理管线

### 1. DTI 指标计算

```bash
# Step 1: 格式转换 ← 详见「格式转换入口」，根据输入格式选择对应命令
mrconvert <input> dwi.mif -datatype float32 ...

# Step 2: 降噪
dwidenoise dwi.mif dwi_den.mif -noise noise.mif

# Step 3: Gibbs 校正
mrdegibbs dwi_den.mif dwi_den_unr.mif

# Step 4: 脑掩膜
dwi2mask dwi_den_unr.mif mask.mif

# Step 5: 张量拟合
dwi2tensor dwi_den_unr.mif dt.mif -mask mask.mif

# Step 6: 张量指标
tensor2metric dt.mif \
  -fa fa.nii.gz \
  -ad ad.nii.gz \
  -rd rd.nii.gz \
  -adc adc.nii.gz \
  -cl cl.nii.gz \
  -cp cp.nii.gz \
  -cs cs.nii.gz \
  -mask mask.mif

# Step 7 (可选): DTI 张量配准到 T1 空间（如需要 T1 空间做后续分析）
# 用 flirt 将张量图配准到 mean_b0，再应用到 DTI 指标
# mrconvert dt.mif dt.nii.gz -force
# flirt -in dt.nii.gz -ref mean_b0.nii.gz -dof 12 -omat tensor2T1_fsl.mat
# mrtransform fa.nii.gz fa_T1.nii.gz -linear tensor2T1_fsl.mat -template T1.nii.gz

# Step 8 (可选): 配准到 MNI
# 详见 references/register.md「阶段一：线性配准到 MNI」
# 分两步：(1) T1→MNI (flirt 12DOF) 和 DWI b0→MNI (flirt 6DOF)
#         (2) DTI 指标复用 dwi_to_MNI 矩阵批量变换

# 7a. T1→MNI（flirt 12 DOF）+ 矩阵转换
flirt -in T1.nii.gz -ref Templates/MNI152.nii.gz -dof 12 -omat T1_to_MNI_fsl.mat
transformconvert T1_to_MNI_fsl.mat T1.nii.gz Templates/MNI152.nii.gz flirt_import T1_to_MNI_mrtrix.txt -force

# 7b. DWI mean b0 → MNI（flirt 6 DOF 刚性）+ 矩阵转换
dwiextract dwi_den_unr.mif - -bzero | mrmath - mean mean_b0.mif -axis 3 -force
mrconvert mean_b0.mif mean_b0.nii.gz -force
flirt -in mean_b0.nii.gz -ref Templates/MNI152.nii.gz -dof 6 -omat dwi_to_MNI_fsl.mat
transformconvert dwi_to_MNI_fsl.mat mean_b0.nii.gz Templates/MNI152.nii.gz flirt_import dwi_to_MNI_mrtrix.txt -force

# 7c. 所有 DTI 指标复用 dwi_to_MNI 矩阵批量变换
for metric in fa ad rd adc cl cp cs; do
  mrtransform ${metric}.nii.gz ${metric}_MNI.nii.gz \
    -linear dwi_to_MNI_mrtrix.txt \
    -template Templates/MNI152.nii.gz \
    -force
done
```

### 2. FOD/CSD 反卷积

```bash
# Step 1: 格式转换 ← 详见「格式转换入口」，根据输入格式选择对应命令
# Step 2: 降噪（dwidenoise）
# Step 3: Gibbs 校正（mrdegibbs）
# Step 4: 脑掩膜（dwi2mask）

# ─── Step 5: 响应函数估计 ───
# 方法 A（推荐，多壳层多组织）: dhollander 全自动
dwi2response dhollander dwi_den_unr.mif wm.txt gm.txt csf.txt -mask mask.mif

# 方法 B（5TT 引导，适合 T1 质量好的数据）:
#   msmt_5tt 利用 5tt 分割结果约束 WM/GM/CSF 组织 mask 估计响应函数
#   支持三种 WM 算法：
#   B1: msmt_5tt + tournier（迭代选择单纤维体素）
#   B2: msmt_5tt + tax（阈值法选择单纤维体素）
#   B3: msmt_5tt + fa（FA 阈值选择单纤维体素）
5ttgen fsl T1.nii.gz 5tt.mif
dwi2response msmt_5tt dwi_den_unr.mif wm.txt gm.txt csf.txt \
  -mask mask.mif -5tt 5tt.mif -wm_algo tournier  # B1
# dwi2response msmt_5tt dwi_den_unr.mif wm.txt gm.txt csf.txt \
#   -mask mask.mif -5tt 5tt.mif -wm_algo tax       # B2
# dwi2response msmt_5tt dwi_den_unr.mif wm.txt gm.txt csf.txt \
#   -mask mask.mif -5tt 5tt.mif -wm_algo fa        # B3

# 方法 C（单壳层/临床数据）: tournier 迭代法
# dwi2response tournier dwi_den_unr.mif response.txt -mask mask.mif

# 方法 D（临床数据快速法）: tax
# dwi2response tax dwi_den_unr.mif response.txt -mask mask.mif

# ─── Step 6: FOD 计算 ───
# 多壳层多组织 MSMT-CSD（与 Step 5 方法 A/B 搭配）
dwi2fod msmt_csd dwi_den_unr.mif \
  wm.txt wm_fod.mif \
  gm.txt gm.mif \
  csf.txt csf.mif \
  -mask mask.mif

# 单组织 CSD（与 Step 5 方法 C/D 搭配，单壳层）
# dwi2fod csd dwi_den_unr.mif response.txt fod.mif -mask mask.mif

# ─── Step 7: 标准化 ───
# 多组织标准化（与 MSMT-CSD 搭配）
mtnormalise wm_fod.mif wm_fod_norm.mif \
  gm.mif gm_norm.mif \
  csf.mif csf_norm.mif \
  -mask mask.mif

# 单组织标准化（与单组织 CSD 搭配）
# mtnormalise fod.mif fod_norm.mif -mask mask.mif

# Step 8 (可选): FOD → MNI
# 详见 references/register.md「阶段一：FOD→MNI」
# 复用 DWI→MNI 的预计算矩阵，带 FOD 重定向
mrtransform wm_fod_norm.mif wm_fod_norm_MNI.mif \
  -linear dwi_to_MNI_mrtrix.txt \
  -template Templates/MNI152.nii.gz \
  -reorient_fod yes \
  -force
```

### 3. 纤维追踪

```bash
# 前置条件：已有 wm_fod_norm.mif（来自 FOD/CSD 管线）

# Step 1: 5TT 生成（需要 T1）
5ttgen fsl T1.nii.gz 5tt.mif
5tt2gmwmi 5tt.mif gmwmi.mif

# Step 2: 全脑纤维追踪
# -algorithm 可选: iFOD2(默认) / SD_Stream / Tensor_Det / Tensor_Prob / FACT
# 详见 references/module-fiber.md
tckgen wm_fod_norm.mif tracks.tck \
  -algorithm iFOD2 \
  -act 5tt.mif \
  -backtrack \
  -seed_gmwmi gmwmi.mif \
  -select 1000000 \
  -maxlength 250 \
  -minlength 10 \
  -step 0.5 \
  -angle 45

# Step 3 (可选): SIFT 滤波（可指定保留纤维数）
tcksift tracks.tck wm_fod_norm.mif tracks_sift.tck \
  -act 5tt.mif \
  -term_number 200000

# Step 4 (可选): SIFT2 权重
tcksift2 tracks.tck wm_fod_norm.mif weights.csv \
  -act 5tt.mif

# Step 5: TDI 图（5 种对比度可选）
# -contrast: tdi(默认)/length/invlength/fod_amp/curvature
# 详见 references/module-fiber.md
tckmap tracks_sift.tck tdi.mif \
  -vox 1.0 \
  -dec \
  -template wm_fod_norm.mif

# 其他对比度示例:
# tckmap tracks_sift.tck length.mif -contrast length -vox 1.0 -template wm_fod_norm.mif
# tckmap tracks_sift.tck fod_amp.mif -contrast fod_amp -vox 1.0 -template wm_fod_norm.mif
# tckmap tracks_sift.tck curvature.mif -contrast curvature -vox 1.0 -template wm_fod_norm.mif
```

### 4. FBA 纤维分析（21 步）

完整的 Fixel-Based Analysis CLI 流程。分四个阶段：

> **入口**：开始前先执行「格式转换入口」，确保每个被试目录下有 `dwi.mif`。
> 若已有 `.mif` 文件跳过格式转换。FBA 有独立的上采样步骤（Step 3），
> 不在预处理环节做上采样。

#### 阶段 A：个体处理（步骤 1-6）

```bash
# Step 0: 格式转换 ← 详见「格式转换入口」
#         确保每个 sub-xxx/ 下有 dwi.mif 和 mask.mif

SUBJECTS="sub-001 sub-002 sub-003"
for sub in $SUBJECTS; do
  cd $sub

  # Step 1: 估计响应函数
  dwi2response dhollander dwi.mif wm.txt gm.txt csf.txt -mask mask.mif

  # Step 2: 在上层目录累积响应函数文件（全部被试跑完后才执行）
  # 将所有 wm.txt 放到同一个目录，然后：
  # responsemean wm_*.txt group_average_wm.txt

  # Step 3: 上采样到 1.25mm
  mrgrid dwi.mif regrid -vox 1.25 dwi_upsampled.mif

  # Step 4: 上采样后掩膜
  dwi2mask dwi_upsampled.mif mask_upsampled.mif

  # Step 5: MSMT CSD
  dwi2fod msmt_csd dwi_upsampled.mif \
    group_average_wm.txt wm_fod.mif \
    group_average_gm.txt gm.mif \
    group_average_csf.txt csf.mif \
    -mask mask_upsampled.mif

  # Step 6: 标准化
  mtnormalise wm_fod.mif wm_fod_norm.mif gm.mif gm_norm.mif csf.mif csf_norm.mif -mask mask_upsampled.mif

  cd ..
done

# Step 2 (续): 计算平均响应函数
responsemean sub-*/wm.txt group_average_wm.txt
responsemean sub-*/gm.txt group_average_gm.txt
responsemean sub-*/csf.txt group_average_csf.txt
```

#### 阶段 B：模版构建与配准（步骤 7-9）

> 详见 `references/register.md`「阶段二：非线性配准到群体模板」

```bash
# Step 7: 构建群体 FOD 模版（推荐 symlink 方式）
mkdir -p templates/fod_input templates/mask_input
for sub in $SUBJECTS; do
  ln -sf ${PWD}/${sub}/wmfod_norm.mif templates/fod_input/${sub}.mif
  ln -sf ${PWD}/${sub}/dwi_mask_upsampled.mif templates/mask_input/${sub}.mif
done
population_template \
  templates/fod_input/ \
  -mask_dir templates/mask_input/ \
  templates/fod_template.mif \
  -voxel_size 1.25

# Step 8: 非线性配准 - 每个被试 FOD → 模版
# -nl_scale 0.5,0.75,1.0: 三阶段多分辨率
# -nl_niter 5,5,15: 每层迭代次数
for sub in $SUBJECTS; do
  mrregister $sub/wmfod_norm.mif \
    -mask1 $sub/dwi_mask_upsampled.mif \
    templates/fod_template.mif \
    -nl_warp $sub/subject2template_warp.mif \
              $sub/template2subject_warp.mif \
    -nl_scale 0.5,0.75,1.0 \
    -nl_niter 5,5,15 \
    -force
done

# Step 9: 掩膜交集 — 个体 mask 变换到模版空间后取交集
for sub in $SUBJECTS; do
  mrtransform $sub/dwi_mask_upsampled.mif \
    -warp $sub/subject2template_warp.mif \
    -interp nearest -datatype bit \
    $sub/dwi_mask_in_template_space.mif -force
done
mrmath sub-*/dwi_mask_in_template_space.mif min \
  templates/template_mask.mif -datatype bit -force
```

#### 阶段 C：Fixel 分析（步骤 10-16）

```bash
# Step 10: 从模版 FOD 生成 fixel 掩膜
# -fmls_peak_value: FOD 峰值阈值，低于此值不生成 fixel（默认 0.1，可调）
fod2fixel \
  -mask templates/template_mask.mif \
  -fmls_peak_value 0.1 \
  templates/fod_template.mif \
  templates/fixel_mask \
  -force

# Step 11: 个体 FOD 变换到模版空间（不重定向！）
# -reorient_fod no: 暂不重定向，由 step13 fixelreorient 独立处理
for sub in $SUBJECTS; do
  mrtransform $sub/wmfod_norm.mif \
    -warp $sub/subject2template_warp.mif \
    -reorient_fod no \
    $sub/fod_in_template_space_NOT_REORIENTED.mif \
    -force
done

# Step 12: 计算 FD (Fiber Density) — 用未重定向的 FOD
for sub in $SUBJECTS; do
  fod2fixel -afd $sub/fod_in_template_space_NOT_REORIENTED.mif \
    templates/fixel_mask \
    $sub/fixel_in_template_space_NOT_REORIENTED \
    -force
done

# Step 13: Fixel 方向重定向 — 用 warp 场重定向 fixel 方向
for sub in $SUBJECTS; do
  fixelreorient \
    $sub/fixel_in_template_space_NOT_REORIENTED \
    $sub/subject2template_warp.mif \
    $sub/fixel_in_template_space \
    -force
done

# Step 14: Fixel 对应关系 — 建立个体 fixel 到模版 fixel 的映射
mkdir -p templates/fd
for sub in $SUBJECTS; do
  fixelcorrespondence \
    $sub/fixel_in_template_space/fd.mif \
    templates/fixel_mask \
    templates/fd \
    $sub.mif \
    -force
done

# Step 15: 计算 FC (Fiber Cross-section) — 用 warp 场提取
mkdir -p templates/fc
for sub in $SUBJECTS; do
  warp2metric \
    $sub/subject2template_warp.mif \
    -fc templates/fixel_mask \
    templates/fc \
    $sub.mif \
    -force
done

# Step 16: 计算 log(FC) 和 FDC
mrcalc $sub/fc.mif -log $sub/log_fc.mif
# FDC = FD × FC（在 fixel 空间逐元素相乘）
mrcalc $sub/fd_corresp.mif $sub/fc.mif -multiply $sub/fdc.mif
```

#### 阶段 D：纤维追踪、连接与统计（步骤 17-20）

```bash
# Step 17: 模版空间纤维追踪
tckgen templates/fod_template.mif templates/tracks.tck \
  -seed_dynamic templates/fod_template.mif \
  -select 20000000 \
  -maxlength 250 \
  -minlength 10 \
  -step 0.5

tcksift templates/tracks.tck templates/fod_template.mif templates/tracks_sift.tck \
  -term_number 2000000

# Step 18: Fixel 连接矩阵
fixelconnectivity templates/fixel_mask templates/tracks_sift.tck templates/connectivity/

# Step 19: 平滑
for metric in fd log_fc fdc; do
  fixelfilter $sub/${metric}_corresp.mif smooth $sub/${metric}_smooth.mif \
    -connectivity templates/connectivity/ \
    -template templates/fixel_mask/ \
    -kernel 10mm
done

# Step 20: 统计（需要设计矩阵文件 design.txt 和对比矩阵 contrast.txt）
# CFE 参数:
#   -cfe_h: 高度参数（默认 2.0），控制簇增强的敏感度
#   -cfe_e: 延伸参数（默认 0.5），控制簇的范围
#   -cfe_c: 连接参数（默认 0.5），控制连接强度
#   -nshifts: 置换次数（默认 5000），越多越稳定但越慢
fixelcfestats \
  sub-*/(fd|log_fc|fdc)_smooth.mif \
  design.txt \
  contrast.txt \
  templates/fixel_mask/ \
  templates/connectivity/ \
  stats/ \
  -cfe_h 2.0 \
  -cfe_e 0.5 \
  -cfe_c 0.5 \
  -nshifts 5000
```

### 5. 连接组矩阵构建

```bash
# 前置条件：已有 .tck 文件和节点图谱

# 生成连接组矩阵
tck2connectome tracks_sift.tck atlas.mif connectome.csv \
  -scale_invnodevol \
  -tck_weights_in weights.csv

# 连接组编辑（可选）
connectomeedit connectome.csv connectome_sym.csv -symmetrise

# 连接组统计
connectomestats connectome.csv design.txt contrast.txt stats/ \
  -options connection
```

### 6. 统计分析

```bash
# 图像 ROI 统计
mrstats image.mif -mask mask.mif -output mean,std,min,max,count

# 簇统计（voxel 级别）
mrclusterstats input.mif design.txt contrast.txt mask.mif stats/ \
  -nshifts 5000 \
  -threshold 2.3

# Fixel 统计
fixelcfestats input_fixel.mif design.txt contrast.txt fixel_mask/ connectivity/ stats/ \
  -nshifts 5000 \
  -connectivity connectivity/

# 纤维统计
tckstats tracks.tck -output length,weighted_length
```

## MRtrix3 命令分组速查

### 格式与转换
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `mrconvert` | 格式转换 | `mrconvert input.nii output.mif -fslgrad bvecs bvals` |
| `mrinfo` | 查看图像信息 | `mrinfo dwi.mif` |
| `mrcat` | 拼接图像 | `mrcat img1.mif img2.mif out.mif -axis 3` |
| `mrmath` | 数学运算 | `mrmath sub-*/fa.mif mean mean_fa.mif -axis 3` |
| `mrcalc` | 逐体素运算 | `mrcalc a.mif b.mif -add c.mif` |
| `mrgrid` | 重采样/裁剪 | `mrgrid in.mif regrid -vox 1.25 out.mif` |
| `mrtransform` | 空间变换 | `mrtransform in.mif -linear mat.txt out.mif` |
| `mrthreshold` | 阈值化 | `mrthreshold in.mif -abs 0.5 out.mif` |
| `mrfilter` | 滤波 | `mrfilter in.mif smooth out.mif -stdev 1.0` |
| `mrhistmatch` | 直方图匹配 | `mrhistmatch source.mif target.mif out.mif` |
| `mrstats` | 图像统计 | `mrstats in.mif -mask mask.mif` |
| `mrregister` | 图像配准 | `mrregister src.mif tgt.mif -type nonlinear -nl_warp warp.mif` |
| `mrview` | 可视化 | `mrview dwi.mif -overlay.load fa.mif` |

### DWI 预处理
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `dwidenoise` | 降噪（MPPCA） | `dwidenoise dwi.mif out.mif -noise noise.mif` |
| `mrdegibbs` | Gibbs 伪影消除 | `mrdegibbs dwi.mif out.mif` |
| `dwifslpreproc` | eddy 头动/涡流校正 | `dwifslpreproc dwi.mif out.mif -rpe_pair -se_epi ap.mif pa.mif` |
| `dwi2mask` | 脑掩膜 | `dwi2mask dwi.mif mask.mif` |
| `dwibiascorrect` | B1 偏差场校正 | `dwibiascorrect ants dwi.mif out.mif -mask mask.mif` |
| `dwifslpreproc` | 全套预处理 | (见上文) |
| `dwiextract` | 提取特定 shell | `dwiextract dwi.mif -bzero b0.mif` |
| `dwi2adc` | 转 ADC 图 | `dwi2adc dwi.mif adc.mif` |

### DTI / DKI
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `dwi2tensor` | 张量拟合 | `dwi2tensor dwi.mif dt.mif -mask mask.mif` |
| `tensor2metric` | 张量指标 | `tensor2metric dt.mif -fa fa.mif -ad ad.mif -rd rd.mif` |
| `dwi2fod` | FOD 估计 | (见 FOD 管线) |

### FOD / CSD
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `dwi2response` | 响应函数估计 | `dwi2response dhollander dwi.mif wm.txt gm.txt csf.txt -mask mask.mif` |
| `dwi2fod` | FOD 计算 | `dwi2fod msmt_csd dwi.mif wm.txt wm_fod.mif ...` |
| `mtnormalise` | 多组织强度标准化 | `mtnormalise wm_fod.mif wm_norm.mif gm.mif gm_norm.mif csf.mif csf_norm.mif` |
| `responsemean` | 平均响应函数 | `responsemean wm_*.txt group_average.txt` |
| `sh2peaks` | SH 峰值提取 | `sh2peaks fod.mif peaks.mif` |
| `fod2fixel` | FOD → fixel | `fod2fixel fod.mif fixel_mask -mask mask.mif` |
| `fod2dec` | FOD → DEC 图 | `fod2dec fod.mif dec.mif` |
| `amp2sh` | 幅值 → SH | (较少独立使用) |
| `sh2amp` | SH → 幅值 | (较少独立使用) |
| `sh2power` | SH → 功率 | (较少独立使用) |
| `shbasis` | SH 基检查 | `shbasis fod.mif` |

### 纤维追踪
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `tckgen` | 纤维追踪 | `tckgen fod.mif tracks.tck -select 1000000 -seed_gmwmi gmwmi.mif` |
| `tckglobal` | 全局追踪 | (多壳层多组织全局追踪，较慢但更精确) |
| `tcksift` | SIFT 滤波 | `tcksift tracks.tck fod.mif tracks_sift.tck -term_number 200000` |
| `tcksift2` | SIFT2 权重 | `tcksift2 tracks.tck fod.mif weights.csv` |
| `tckedit` | 编辑纤维 | `tckedit tracks.tck out.tck -tck_weights_in weights.csv -endpoints roi.mif` |
| `tckmap` | TDI 图 | `tckmap tracks.tck tdi.mif -vox 1.0 -dec` |
| `tck2connectome` | 连接组矩阵 | `tck2connectome tracks.tck atlas.mif connectome.csv` |
| `tck2fixel` | Fixel TDI | `tck2fixel tracks.tck fixel_mask out/` |
| `tckinfo` | 纤维信息 | `tckinfo tracks.tck` |
| `tckstats` | 纤维统计 | `tckstats tracks.tck` |
| `tckconvert` | 格式转换 | `tckconvert input.tck output.vtk` |
| `tcktransform` | 纤维空间变换 | `tcktransform tracks.tck warp.mif out.tck` |
| `tckresample` | 重采样 | `tckresample tracks.tck out.tck -step 0.5` |
| `tcksample` | 沿纤维采样 | `tcksample tracks.tck image.mif values.csv` |

### Fixel 分析
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `fod2fixel` | FOD → fixel | `fod2fixel fod.mif fixel_dir/ -mask mask.mif` |
| `fixel2voxel` | Fixel → 体素 | `fixel2voxel fixel_data.mif out.mif` |
| `fixel2peaks` | fixel → 峰值方向 | `fixel2peaks fixel_dir/ peaks.mif` |
| `fixel2sh` | fixel → SH | (转换回 SH 表示) |
| `fixel2tsf` | fixel → 纤维标量 | (映射到纤维上的标量文件) |
| `fixelconvert` | fixel 格式转换 | (新旧格式互转) |
| `fixelcrop` | 裁剪 fixel | `fixelcrop fixel_dir/ mask_dir/ out_dir/` |
| `fixelfilter` | Fixel 滤波 | `fixelfilter in_dir/ smooth out_dir/ -kernel 10mm` |
| `fixelreorient` | Fixel 方向重定向 | `fixelreorient fod.mif warp.mif fixel_dir/` |
| `fixelcorrespondence` | Fixel 对应关系 | `fixelcorrespondence subject_index.mif template_fixel_mask/ subject_data.mif out.mif` |
| `fixelconnectivity` | Fixel 连接矩阵 | `fixelconnectivity fixel_dir/ tracks.tck connectivity_dir/` |
| `fixelcfestats` | Fixel 统计 | `fixelcfestats subject_files design.txt contrast.txt fixel_mask/ connectivity/ stats/` |

### ACT / 5TT / 标签
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `5ttgen` | 5TT 生成 | `5ttgen fsl T1.nii.gz 5tt.mif` |
| `5tt2gmwmi` | GM-WM 边界 | `5tt2gmwmi 5tt.mif gmwmi.mif` |
| `5ttcheck` | 5TT 检查 | `5ttcheck 5tt.mif` |
| `5ttedit` | 5TT 编辑 | `5ttedit 5tt.mif -mask mask.mif` |
| `5tt2vis` | 5TT 可视化 | `5tt2vis 5tt.mif vis.mif` |
| `labelconvert` | 图谱转换 | `labelconvert atlas.mif LUT_in.txt LUT_out.txt atlas_converted.mif` |
| `label2colour` | 标签→彩色图 | `label2colour atlas.mif LUT.txt colour.mif` |
| `label2mesh` | 标签→网格 | `label2mesh atlas.mif mesh.obj` |
| `labelstats` | 标签统计 | `labelstats atlas.mif -output volume` |

### 变形场 / 配准
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `flirt` (FSL) | 线性配准（T1/DWI→MNI） | `flirt -in src.nii -ref MNI152.nii.gz -dof 12 -omat mat.mat` |
| `mrregister` | 图像配准（原生支持 .mif 和 FOD） | `mrregister src.mif tgt.mif -mask1 mask.mif -nl_warp warp.mif -nl_scale 0.5,0.75,1.0 -nl_niter 5,5,15` |
| `mrtransform` | 用矩阵或变形场变换图像 | `mrtransform in.mif -linear mat.txt out.mif -template ref.mif` |
| `transformconvert` | FSL/MRtrix/ITK 矩阵格式互转 | `transformconvert fsl.mat src.nii ref.nii flirt_import mrtrix.txt` |
| `population_template` | 构建群体模板 | `population_template fod_input/ -mask_dir mask_input/ template.mif -voxel_size 1.25` |
| `fixelreorient` | 用变形场重定向 fixel 方向 | `fixelreorient fixel_dir warp.mif out_fixel_dir` |
| `warp2metric` | 从变形场提取 FC/Jacobian | `warp2metric warp.mif -fc fixel_mask fc.mif` |
| `warpinit` | 创建恒等变形场 | `warpinit template.mif identity_warp.mif` |
| `warpinvert` | 变形场求逆 | `warpinvert warp.mif inv_warp.mif` |
| `warpconvert` | 变形场格式互转 | `warpconvert warp.mif -type mrtrix2itk out.mif` |
| `warpcorrect` | 变形场修正 | `warpcorrect warp.mif out.mif` |
| `transformcalc` | 变换矩阵运算 | `transformcalc mat1.txt mat2.txt multiply out.txt` |
| `transformcompose` | 变换组合 | `transformcompose linear.txt warp.mif combined_warp.mif` |

### 方向集
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `dirgen` | 生成均匀方向 | `dirgen 300 directions.txt` |
| `dirflip` | 方向翻转优化 | `dirflip directions.txt flipped.txt` |
| `dirorder` | 方向排序 | `dirorder directions.txt ordered.txt` |
| `dirsplit` | 方向分割 | `dirsplit directions.txt -n 2` |
| `dirmerge` | 方向合并 | `dirmerge set1.txt set2.txt merged.txt` |
| `dirstat` | 方向统计 | `dirstat directions.txt` |

### 连接组
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `tck2connectome` | 连接组矩阵 | `tck2connectome tracks.tck atlas.mif connectome.csv -scale_invnodevol` |
| `connectome2tck` | 矩阵→纤维 | `connectome2tck tracks.tck connectome.csv out/ -nodes 1,2` |
| `connectomeedit` | 矩阵编辑 | `connectomeedit in.csv out.csv -symmetrise` |
| `connectomestats` | 连接组统计 | `connectomestats connectome.csv design.txt contrast.txt stats/ -options edge` |

### 其他
| 命令 | 用途 | 典型用法 |
|------|------|---------|
| `for_each` | 批量循环 | `for_each sub-*/ : mrconvert IN/dwi.mif IN/dwi.nii.gz` |
| `mrtrix_cleanup` | 清理临时文件 | `mrtrix_cleanup` |
| `peaks2amp` | 峰值→幅值 | `peaks2amp peaks.mif amp.mif` |
| `peaks2fixel` | 峰值→fixel 目录 | `peaks2fixel peaks.mif fixel_dir/` |
| `vectorstats` | 向量统计检验 | `vectorstats data.csv design.txt contrast.txt stats/` |
| `mesh2voxel` | 网格→体素图 | `mesh2voxel mesh.obj template.mif pve.mif` |
| `meshconvert` | 网格格式转换 | `meshconvert in.obj out.vtk` |
| `meshfilter` | 网格滤波 | `meshfilter in.obj smooth out.obj` |
| `voxel2fixel` | 体素值→fixel | (将体素标量值映射到所有 fixel) |
| `voxel2mesh` | 体素→网格 | `voxel2mesh label.mif mesh.obj` |
| `maskfilter` | 掩膜滤波 | `maskfilter mask.mif dilate out.mif -npass 1` |
| `maskdump` | 掩膜坐标导出 | `maskdump mask.mif > coords.txt` |
| `tsfinfo` | TSF 信息 | `tsfinfo file.tsf` |
| `tsfmult` | TSF 相乘 | `tsfmult a.tsf b.tsf out.tsf` |
| `tsfdivide` | TSF 相除 | `tsfdivide a.tsf b.tsf out.tsf` |
| `tsfsmooth` | TSF 平滑 | `tsfsmooth in.tsf out.tsf -stdev 10` |
| `tsfthreshold` | TSF 阈值化 | `tsfthreshold in.tsf -abs 0.5 out.tsf` |
| `tsfvalidate` | TSF 验证 | `tsfvalidate file.tsf tracks.tck` |
| `dcminfo` | DICOM 信息 | `dcminfo dicom_dir/` |
| `dcmedit` | DICOM 编辑 | `dcmedit dicom_file -tag 0010,0010 "value"` |
| `afdconnectivity` | AFD 连接性 | `afdconnectivity fod.mif tracks.tck roi1.mif roi2.mif` |

## 交互工作流

### 第一阶段：需求收集

向用户依次了解：

1. **数据情况**
   - 数据格式（DICOM/IMA / NIfTI / .mif）
   - 被试数量和命名方式
   - 是否有 T1 结构像、AP/PA b0
   - 数据是否已整理归类

2. **分析目标**
   - **DTI 指标** — FA/AD/RD/MD 等
   - **FOD/CSD 反卷积** — 响应函数 + CSD
   - **纤维追踪** — 全脑/ROI 追踪 + SIFT
   - **FBA 纤维分析** — 完整 21 步流程
   - **连接组矩阵** — 基于图谱的连接组
   - **统计分析** — 群组比较 / 置换检验
   - **仅整理/转换格式**

3. **已处理情况** — 哪些步骤已完成？是否有中间结果？

### 第二阶段：流程规划

根据目标推荐管线，并制定命令执行计划。管线依赖关系：

```
预处理（降噪/Gibbs/掩膜/eddy/B1）→ FOD → 纤维追踪 → 连接组
                                    ↓
                                   FBA（21步）
                         DTI（依赖预处理输出）
```

### 第三阶段：逐步执行

对每个步骤：
1. 向用户说明将要执行的命令和预期输出
2. 执行 bash 命令（使用 Bash 工具）
3. 验证输出文件是否存在
4. 检查命令是否成功（exit code = 0）
5. 如有错误，分析错误信息并提供修复方案
6. 完成后确认，询问是否继续下一步

### 第四阶段：结果交付

- 告知用户所有输出文件的位置
- 提供可视化建议（`mrview` 或导出 PNG）
- 如有后续分析建议，一并提出

## 耗时参考

| 步骤 | 耗时 |
|------|------|
| 格式转换 | 1-3 min/被试 |
| dwidenoise | 5-10 min/被试 |
| mrdegibbs | 2-5 min/被试 |
| dwifslpreproc (eddy) | 30-120 min/被试（GPU 可加速） |
| dwibiascorrect | 5-10 min/被试 |
| dwi2tensor + tensor2metric | 5-15 min/被试 |
| dwi2response | 5-10 min/被试 |
| dwi2fod + mtnormalise | 10-30 min/被试 |
| tckgen（百万纤维）| 10-60 min/被试 |
| population_template | 30-120 min |
| fixelcfestats | 60-300 min（5000 置换） |

## 常见错误排查

| 错误 | 原因 | 解决 |
|------|------|------|
| `command not found: mrconvert` | MRtrix3 未在 PATH | `export PATH=/path/to/mrtrix3/bin:$PATH` |
| `mrconvert: error while loading` | 动态库缺失 | 检查 MRtrix3 安装完整性 |
| `[ERROR] image "xxx" does not exist` | 路径或文件名错误 | 确认文件路径和扩展名 |
| `[ERROR] matrix does not fit the data` | bvecs/bvals 与 DWI 不匹配 | 检查梯度表行列数 |
| `eddy: command not found` | FSL 未安装或未配置 | `export FSLDIR=...` 或安装 FSL |
| `[ERROR] failed to open shell script` | dwifslpreproc 需要 Python | `pip install mrtrix3` |
| `[ERROR] multiple matching shells` | b 值多壳层但用了单组织 | 改用 msmt_csd 算法 |
