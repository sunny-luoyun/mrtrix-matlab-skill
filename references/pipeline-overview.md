# 整体流程总览

## 快速决策树

```
用户输入: "我有一批弥散数据要处理"

问: 数据是否已按被试整理？
├─ 否 → 使用 sort 模块（IMAsort / NIFTIsort）
└─ 是 → 问: 处理目标是什么？
        ├─ DTI 指标 → sort → pre → dti
        ├─ FOD/CSD  → sort → pre → fod
        ├─ 纤维追踪 → sort → pre → fod → fiber
        ├─ FBA 分析 → sort → pre → fod → fba（自身21步）
        ├─ 网络矩阵 → sort → pre → fod → fiber → map
        ├─ 统计分析 → 确认已有前序结果 → stats
        └─ 仅整理数据 → sort（独立完成）
```

## 模块依赖矩阵

| 模块 | 前置依赖 | 输入格式 | 输出格式 |
|------|---------|---------|---------|
| sort | 无 | IMA/DICOM/NIFTI | .nii.gz / .mif |
| pre | sort | .nii.gz / .mif | .mif + .nii.gz |
| dti | pre | .mif（预处理后） | .nii（MNI空间） |
| fod | pre | .mif（预处理后） | .mif（FOD） |
| fiber | fod | .mif（FOD） | .tck |
| fba | fod | .mif（FOD） | 多种 .mif/.nii |
| map | fiber | .tck + 图谱 | .txt / .csv |
| stats | 任意前序 | .mif/.nii/.tck | 数据/图表 |

## 各模块预处理步骤可选项

pre 模块中，功能可单独勾选，典型组合：

| 场景 | 推荐勾选 |
|------|---------|
| 仅有 DWI（无 AP/PA） | 格式转换 + 去噪 + Gibbs + B1 校正 + 掩膜 |
| 有 AP/PA b0 | 上项 + 头动校正（含 eddy） |
| 有 T1 | 上项 + T1 配准到 MNI + T1 组织分割 |
| 需 DWI 到 MNI | 上项 + DWI 配准到 MNI |

## 耗时预估

| 步骤 | 耗时（典型） | 说明 |
|------|------------|------|
| 格式转换 | 1-3 min/被试 | 取决于数据量 |
| 去噪 | 5-10 min/被试 | |
| Gibbs 校正 | 2-5 min/被试 | |
| 头动校正(eddy) | 30-120 min/被试 | GPU 可加速 |
| B1 校正 | 5-10 min/被试 | |
| DTI 指标计算 | 5-15 min/被试 | 含配准到 MNI |
| FOD 计算 | 10-30 min/被试 | |
| 纤维追踪 | 10-60 min/被试 | 取决于纤维数量 |
| FBA 模版构建 | 30-120 min | 取决于被试数 |
| FBA 统计 | 60-300 min | 置换检验 |
| 簇统计 | 30-120 min | 置换检验 |

## 常用错误排查

| 错误 | 可能原因 | 解决 |
|------|---------|------|
| `command not found: mrconvert` | 环境变量未设置 | 运行 `env.m` |
| `Undefined function 'prepro'` | 路径未添加 | 运行 `setup_path` |
| `mrconvert: error while loading` | MRtrix3 未安装或路径不对 | 检查 `env.m` 中的路径 |
| eddy 报错 | FSL 未配置 | 检查 `FSLDIR` 环境变量 |
