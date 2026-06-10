# sort — 原始数据整理模块

## 用途

将原始采集的 IMA/DICOM 或 NIFTI 格式数据按被试进行整理归类，确保后续处理模块能正确找到每个被试的数据。

## 文件清单

### sortimg.m（App Designer 主界面）

**入口函数**。运行后在 GUI 中选择：
- **模式**：IMA 格式整理 / NIFTI 格式整理
- **工作目录**：原始数据所在的根目录
- **样例会前缀**：用于匹配被试 ID 的前缀模式

### IMAsort.m

**功能**：将 IMA/DICOM 格式的原始数据按被试整理归类。
- 根据文件夹前缀匹配被试
- 复制指定关键字文件到目标目录
- 输出结构：`工作目录/sub-xxx/DWI/`、`工作目录/sub-xxx/T1/`

### NIFTIsort.m

**功能**：将 NIFTI 格式的原始 T1 和 DWI 数据按被试整理归类。
- 自动识别 T1 和 DWI 文件
- 输出结构同 IMAsort

## 输出结构

```
工作目录/
├── sub-001/
│   ├── DWI/
│   │   ├── sub-001_dwi.nii.gz
│   │   └── sub-001_dwi.bvec
│   │   └── sub-001_dwi.bval
│   └── T1/
│       └── sub-001_T1.nii.gz
├── sub-002/
│   └── ...
```

## 典型对话

```
用户: 我有 DICOM 数据要整理
Agent: sortimg 支持两种模式：
  1. IMA 格式整理 — 用于 DICOM/IMA 格式
  2. NIFTI 格式整理 — 用于 .nii.gz 格式
  请问您的数据是哪种格式？
```
