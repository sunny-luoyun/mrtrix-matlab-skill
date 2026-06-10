# 预处理 CLI 参考

## 流程概述

预处理是 DTI / FOD / 纤维追踪等所有下游分析的前置步骤。

## 标准命令序列

### 1. 格式转换（NIfTI → .mif）

```bash
# DWI 转换（需 bvecs/bvals 或 mrtrix 梯度格式）
mrconvert dwi.nii.gz dwi.mif -fslgrad bvecs bvals -datatype float32

# 从 DICOM 目录直接转（推荐！mrconvert 直接读取 DICOM）
mrconvert dicom_dir/ dwi.mif
```

### 2. 降噪（dwidenoise）

```bash
dwidenoise dwi.mif dwi_den.mif -noise noise.mif
# noise.mif: 估计的噪声图，可用于质量评估
```

### 3. Gibbs 伪影消除

```bash
mrdegibbs dwi_den.mif dwi_den_unr.mif
# 消除截断伪影（对高 b 值图像尤其重要）
```

### 4. 脑掩膜

```bash
dwi2mask dwi_den_unr.mif mask.mif

# 可选：掩膜膨胀（确保覆盖所有脑组织）
maskfilter mask.mif dilate mask_dilated.mif -npass 2
```

### 5. 头动 + 涡流校正（eddy）

需要 AP/PA 相位编码 b0 配对：

```bash
dwifslpreproc dwi_den_unr.mif dwi_den_unr_eddy.mif \
  -rpe_pair \
  -se_epi ap_b0.mif pa_b0.mif \
  -pe_dir ap \
  -eddy_options " --slm=linear --data_is_shelled"
```

无 AP/PA 对时跳过此步骤（eddy 无法运行）。

### 6. B1 偏差场校正

```bash
dwibiascorrect ants dwi_den_unr_eddy.mif dwi_prepro.mif -mask mask.mif
# 依赖 ANTs 中的 N4BiasFieldCorrection
```

### 7. T1 配准到 MNI（12 DOF）

详见 `references/register.md`「阶段一：T1→MNI」。

```bash
# 7a. T1→MNI via flirt（12 DOF 仿射）
mrconvert T1.mif T1.nii.gz -force
flirt -in T1.nii.gz -ref Templates/MNI152.nii.gz -dof 12 -out T1_coreg.nii.gz -omat T1_to_MNI_fsl.mat
transformconvert T1_to_MNI_fsl.mat T1.nii.gz Templates/MNI152.nii.gz flirt_import T1_to_MNI_mrtrix.txt -force
mrtransform T1.nii.gz T1_MNI.nii.gz -linear T1_to_MNI_mrtrix.txt -template Templates/MNI152.nii.gz

# 7b. DWI mean b0 → MNI via flirt（6 DOF 刚性）
dwiextract dwi.mif - -bzero | mrmath - mean mean_b0.mif -axis 3 -force
mrconvert mean_b0.mif mean_b0.nii.gz -force
flirt -in mean_b0.nii.gz -ref Templates/MNI152.nii.gz -dof 6 -out dwi_coreg.nii.gz -omat dwi_to_MNI_fsl.mat
transformconvert dwi_to_MNI_fsl.mat mean_b0.nii.gz Templates/MNI152.nii.gz flirt_import dwi_to_MNI_mrtrix.txt -force
```

### 8. 5TT 组织分割

```bash
5ttgen fsl T1.nii.gz 5tt.mif
5tt2gmwmi 5tt.mif gmwmi.mif
```

## 可选步骤组合

| 场景 | 必须步骤 | 可选步骤 |
|------|---------|---------|
| 仅 DTI | 格式转换 + 去噪 + Gibbs + 掩膜 | eddy、B1、flirt T1/DWI→MNI |
| FOD + 纤维追踪 | 同上 + B1 | eddy、5TT |
| FBA | 格式转换（不上采样前不做预处理） | —（FBA 有独立预处理）|

## 质量检查

```bash
# 检查各步骤输出
mrinfo dwi_prepro.mif
mrview dwi_prepro.mif -overlay.load mask.mif
```
