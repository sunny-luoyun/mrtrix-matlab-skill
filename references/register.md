# 配准 CLI 参考

## 配准策略概览

本 skill 采用两阶段配准策略，与 `/Users/langqin/software/mrtrix-matlab` 项目一致：

```
阶段一：个体空间 → MNI 标准空间（线性）
  T1  → flirt (12 DOF) → transformconvert → T1_to_MNI_mrtrix.txt
  DWI → dwiextract mean_b0 → flirt (6 DOF) → transformconvert → dwi_to_MNI_mrtrix.txt
  └──→ DTI 指标 / FOD 均复用此矩阵

阶段二：个体 FBA → 群体模板（非线性，仅 FBA）
  population_template → mrregister (-nl_warp) → 变形场
  └──→ mask 变换 / FOD 变换 / fixel 重定向 / FC 提取
```

---

## 阶段一：线性配准到 MNI

### 1. T1 → MNI (12 DOF)

使用 FSL `flirt` 进行 12 自由度仿射配准，再用 `transformconvert` 转为 MRtrix3 格式：

```bash
# 输入: T1.nii.gz 或 T1.mif
# 模板: Templates/MNI152.nii.gz

# 1.0 如果输入是 .mif，先转 .nii.gz（flirt 需要）
mrconvert T1.mif T1.nii.gz -force

# 1.1 FSL flirt 线性配准 T1 → MNI152 (dof=12)
flirt \
  -in T1.nii.gz \
  -ref Templates/MNI152.nii.gz \
  -dof 12 \
  -out T1_coreg.nii.gz \
  -omat T1_to_MNI_fsl.mat

# 1.2 转换 FSL 矩阵为 MRtrix3 格式
transformconvert \
  T1_to_MNI_fsl.mat \
  T1.nii.gz \
  Templates/MNI152.nii.gz \
  flirt_import \
  T1_to_MNI_mrtrix.txt \
  -force

# 1.3 应用变换（可选，看配准效果）
mrtransform T1.nii.gz T1_MNI.nii.gz \
  -linear T1_to_MNI_mrtrix.txt \
  -template Templates/MNI152.nii.gz
```

> **为什么用 `flirt` 而非 `mrregister`？** `flirt` 对 T1→MNI 这类跨模态配准更成熟，且与 FSL 生态兼容。12 DOF 允许各向异性缩放（适应不同分辨率）。

### 2. DWI → MNI (6 DOF)

用 DWI 的 mean b0 做刚性配准（6 DOF，只旋转+平移，不缩放）：

```bash
# 输入: dwi.mif (预处理后)
# 模板: Templates/MNI152.nii.gz

# 2.1 提取 b0 并取均值
dwiextract dwi.mif - -bzero | mrmath - mean mean_b0.mif -axis 3 -force

# 2.2 b0 → .nii.gz（flirt 需要）
mrconvert mean_b0.mif mean_b0.nii.gz -force

# 2.3 FSL flirt 刚性配准 b0 → MNI152 (dof=6)
flirt \
  -in mean_b0.nii.gz \
  -ref Templates/MNI152.nii.gz \
  -dof 6 \
  -out dwi_coreg.nii.gz \
  -omat dwi_to_MNI_fsl.mat

# 2.4 转换 FSL 矩阵为 MRtrix3 格式
transformconvert \
  dwi_to_MNI_fsl.mat \
  mean_b0.nii.gz \
  Templates/MNI152.nii.gz \
  flirt_import \
  dwi_to_MNI_mrtrix.txt \
  -force
```

> **为什么 b0 用 6 DOF（刚性）而非 12？** b0 像已经是弥散像的空间，只需要和 MNI 对齐方向/位置，不允许缩放扭曲，确保 DTI 指标的空间完整性。

### 3. DTI 指标 → MNI

所有 DTI 指标（FA/AD/RD/ADC/CL/CP/CS/AK/MK/RK）均复用 `dwi_to_MNI_mrtrix.txt`：

```bash
# 复用预计算的 dwi_to_MNI 矩阵，批量变换所有指标
for metric in fa ad rd adc cl cp cs ak mk rk; do
  mrtransform ${metric}.mif ${metric}_MNI.nii.gz \
    -linear dwi_to_MNI_mrtrix.txt \
    -template Templates/MNI152.nii.gz \
    -force
done
```

> **核心原则**：只配准一次 `dwi→MNI`，所有 DTI 指标共用同一矩阵，保证空间一致性。

### 4. FOD → MNI

同样复用 `dwi_to_MNI_mrtrix.txt`，但需要 `-reorient_fod yes`：

```bash
# 多组织 MSMT-CSD
mrtransform wmfod_norm.mif wmfod_norm_MNI.mif \
  -linear dwi_to_MNI_mrtrix.txt \
  -template Templates/MNI152.nii.gz \
  -reorient_fod yes \
  -force

# 单组织 CSD
mrtransform fod_norm.mif fod_norm_MNI.mif \
  -linear dwi_to_MNI_mrtrix.txt \
  -template Templates/MNI152.nii.gz \
  -reorient_fod yes \
  -force
```

> **`-reorient_fod yes` 的作用**：FOD 是方向函数，空间变换后需要根据变形重新定向，否则方向信息错误。DTI 标量（FA/RD 等）是各向同性标量，不需要重定向。

---

## 阶段二：非线性配准到群体模板（FBA）

### 5. 构建群体模板（population_template）

```bash
# 建立软链接目录（避免移动大量数据）
mkdir -p fod_input mask_input
for sub in sub-001 sub-002 sub-003; do
  ln -sf ${PWD}/FBA/subjects/${sub}/wmfod_norm.mif fod_input/${sub}.mif
  ln -sf ${PWD}/FBA/subjects/${sub}/dwi_mask_upsampled.mif mask_input/${sub}.mif
done

# 构建群体 FOD 模板
population_template \
  fod_input/ \
  -mask_dir mask_input/ \
  FBA/template/wmfod_template.mif \
  -voxel_size 1.25
```

关键参数：

| 参数 | 说明 |
|------|------|
| `-mask_dir` | 个体掩膜目录，用于配准加速和掩膜约束 |
| `-voxel_size` | 模板体素大小，须与上采样一致（默认 1.25mm） |
| `-linear` | 仅线性配准（较快） |
| `-nl` | 使用非线性配准（更精确，FBA 推荐） |
| `-nl_scale` | 非线性配准缩放，默认 `0.25,0.5,0.75,1.0` |
| `-nl_niter` | 非线性配准迭代次数，默认 `5,5,10,20` |

### 6. 个体 FOD 非线性配准到模板（mrregister）

```bash
mrregister \
  FBA/subjects/sub-001/wmfod_norm.mif \
  -mask1 FBA/subjects/sub-001/dwi_mask_upsampled.mif \
  FBA/template/wmfod_template.mif \
  -nl_warp FBA/subjects/sub-001/subject2template_warp.mif \
            FBA/subjects/sub-001/template2subject_warp.mif \
  -nl_scale 0.5,0.75,1.0 \
  -nl_niter 5,5,15 \
  -force
```

**`mrregister` 关键参数详解：**

| 参数 | 含义 | 推荐值 |
|------|------|--------|
| `-mask1` | 源图像掩膜 | 个体上采样 mask |
| `-mask2` | 目标图像掩膜（可选） | 模板 mask |
| `-nl_warp` | 输出变形场：正向源→目标，反向目标→源 | - |
| `-nl_scale` | 多分辨率缩放层级 | `0.5,0.75,1.0`（3 层）或 `0.25,0.5,0.75,1.0`（4 层） |
| `-nl_niter` | 每层迭代次数 | `5,5,15` 或 `5,5,10,20`（对应 4 层） |
| `-type` | 配准类型：`rigid` / `affine` / `rigid_nonlinear` / `nonlinear` | 与 FOD 配常用 `nonlinear` |
| `-init` | 初始变换（先线性对齐再非线性） | `linear.mat` |
| `-noreorientation` | 配准过程中不重定向 FOD | 默认重定向，FBA 场景设为 `-noreorientation` |

**多分辨率策略说明：**
- `-nl_scale 0.5,0.75,1.0`：先在 50% 分辨率粗配，再到 75%，最后 100% 精配
- `-nl_niter 5,5,15`：每层的迭代次数，最后层更多迭代做精细调整
- 这种策略加速收敛、避免局部最优

### 7. 变形场应用

`subject2template_warp.mif` 是后续所有 Fixel 分析的基础，它用于：

#### 7a. Mask 变换到模板空间（取交集）

```bash
# 个体 mask → 模板空间
mrtransform dwi_mask_upsampled.mif \
  -warp subject2template_warp.mif \
  -interp nearest \
  -datatype bit \
  dwi_mask_in_template_space.mif \
  -force

# 所有被试 mask 取交集
mrmath sub-*/dwi_mask_in_template_space.mif min \
  template/template_mask.mif \
  -datatype bit \
  -force
```

#### 7b. FOD 变换到模板空间（不重定向）

```bash
# 说明：此处暂不重定向（-reorient_fod no）
# 因为后续 fixelreorient 会独立处理方向重定向
mrtransform wmfod_norm.mif \
  -warp subject2template_warp.mif \
  -reorient_fod no \
  fod_in_template_space_NOT_REORIENTED.mif \
  -force
```

#### 7c. Fixel 重定向

```bash
# 用 warp 场重定向 fixel 方向
fixelreorient \
  fixel_in_template_space_NOT_REORIENTED \
  subject2template_warp.mif \
  fixel_in_template_space \
  -force
```

#### 7d. 从变形场提取 FC（Fiber Cross-section）

```bash
# warp2metric 从变形场计算纤维横截面积变化
warp2metric \
  subject2template_warp.mif \
  -fc template/fixel_mask \
  fc.mif \
  -force
```

---

## `transformconvert` 详解

FSL 与 MRtrix3 使用不同格式的变换矩阵，`transformconvert` 负责互转。

```bash
# FSL → MRtrix3
transformconvert fsl.mat src.nii ref.nii flirt_import mrtrix.txt -force

# MRtrix3 → FSL
transformconvert mrtrix.txt src.nii ref.nii mrtrix2flirt fsl.mat -force

# ITK 格式转换
transformconvert itk.mat src.nii ref.nii itk_import mrtrix.txt -force
```

参数说明：

| 参数 | 说明 |
|------|------|
| `flirt_import` | FSL 仿射矩阵 → MRtrix3 |
| `mrtrix2flirt` | MRtrix3 → FSL 仿射矩阵 |
| `itk_import` | ITK/ANTs 格式 → MRtrix3 |

> **文件格式区别**：FSL 矩阵是 4×4 的 text 文件（前 3 行有效，第 4 行 `0 0 0 1`），MRtrix3 矩阵也是 4×4 但使用不同的坐标映射约定（voxel-to-RAS vs FSL 的 voxel-to-scanner）。

---

## `flirt` vs `mrregister` 选择

| 场景 | 推荐工具 | 原因 |
|------|---------|------|
| T1 → MNI | `flirt` | 跨模态、FSL 成熟、T1 与 MNI 模板同模态 |
| DWI b0 → MNI | `flirt` | 兼容 FSL 生态、6 DOF 控制精确 |
| DTI 指标 → MNI | `mrtransform -linear` | 复用预计算矩阵，批量处理 |
| FOD → MNI | `mrtransform -linear` | 可同时 `-reorient_fod yes` |
| FOD → 群体模板 | `mrregister` | 原生 .mif 支持、FOD 配准优化、直接输出 warp |
| 图像 → 标准空间 | `mrtransform` | 使用已有矩阵或 warp 变换 |

---

## 配准质量检查

```bash
# 1. mrview 叠加检查
mrview Templates/MNI152.nii.gz -overlay.load T1_MNI.nii.gz

# 2. 棋盘格模式检查
mrview Templates/MNI152.nii.gz -overlay.load T1_MNI.nii.gz -mode 2

# 3. 计算相似度指标
mrmetric T1_MNI.nii.gz Templates/MNI152.nii.gz -metric mi,ncc

# 4. 检查变形场是否合理（无折叠）
warp2metric subject2template_warp.mif -jac template/fixel_mask jac.mif
mrstats jac.mif -output mean,std

# 5. FOD 配准后检查
mrview wmfod_template.mif -overlay.load fod_in_template_space_NOT_REORIENTED.mif
```

---

## 常见问题排查

| 问题 | 可能原因 | 解决 |
|------|---------|------|
| `flirt` 配准结果明显偏移 | 初始对齐太差 | 先用 `-searchrx -180 180 -searchry -180 180 -searchrz -180 180` 扩大搜索范围 |
| T1→MNI 结果扭曲过大 | 12 DOF 过拟合 | 改为 9 DOF 或 7 DOF（`-dof 9` / `-dof 7`） |
| `mrregister` 配准失败 | 未初始化的非线性配准 | 加 `-type affine` 先做线性预配，再 `-init` 初始化非线性 |
| 变形场折叠（Jacobian ≤ 0） | 正则化太弱 / 图像质量差 | 增加 `-nl_scale` 层数或增加 `-nl_niter` 迭代 |
| FOD 重定向后异常 | 重定向顺序/方向错误 | 确认 FOD 配准用 `-noreorientation`，独立用 `fixelreorient` |
| `transformconvert` 报错 | 源/参考图像路径错误 | 确认 src 和 ref 图像路径与配准时使用的一致 |
| 不同被试配准到同一模板 | 模板构建时未包含所有被试 | 确认 `population_template` 输入包含了所有组别被试 |
