# pre — 预处理模块

## 用途

对整理后的 DWI 和 T1 数据进行标准化预处理，包括格式转换、去噪、校正、配准和组织分割。

## 文件清单

### prepro.m（App Designer 主界面）

**入口函数**。运行后打开预处理参数配置界面，可勾选多个处理步骤，配置参数，批量处理多个被试。

**可勾选的步骤**（可按需组合）：
1. 格式转换（DICOM/NIFTI → .mif）
2. 去噪（dwidenoise）
3. Gibbs 校正（mrdegibbs）
4. 头动校正（含 eddy）
5. B1 偏差场校正
6. T1 配准到 MNI
7. DWI 配准到 MNI
8. T1 组织分割

### 单步函数

| 函数 | 命令行 | 说明 |
|------|--------|------|
| `change_format` | `mrconvert` | DICOM/NIFTI → .mif |
| `denoise` | `dwidenoise` | 扩散加权图像去噪 |
| `gibbs` | `mrdegibbs` | Gibbs ringing 校正 |
| `headmove` | `dwifslpreproc` | 头动 + 涡流校正（需 AP/PA b0） |
| `bias` | `dwibiascorrect ants` | B1 偏差场校正 |
| `mask` | `dwi2mask` + `maskfilter` | 脑掩膜生成 + 膨胀 |
| `dwitoMNI` | `flirt` + `mrtransform` | DWI 线性配准到 MNI152 |
| `T1toMNI` | `flirt` | T1 配准到 MNI152 |
| `T1corg` | `5ttgen fsl` + `5tt2gmwmi` | T1 五组织分割 |

## 关键参数

### headmove（头动校正）

- **AP 相位编码 b0**：需要提供 AP 方向的 b0 文件
- **PA 相位编码 b0**：需要提供 PA 方向的 b0 文件
- **无 AP/PA**：如果无 AP/PA b0 配对，此步骤不可用

### dwitoMNI（DWI 配准到 MNI）

- **T1 配准方式**：基于 T1 到 MNI 的配准矩阵，变换 DWI 到 MNI
- **插值方式**：线性/样条

### T1corg（组织分割）

- **输出**：5tt.mif（5 组织类型）+ gmwmi.mif（白质-灰质边界）

## 处理建议

1. 如果数据量大，建议先在单个被试上测试完整流程
2. eddy（头动校正）是最耗时的步骤，建议 GPU 加速
3. T1 组织分割和 DWI 配准可并行运行
4. 所有预处理步骤输出均为 .mif 格式，保留原始数据不变
