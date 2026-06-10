# dti — 弥散指标计算模块

## 用途

从预处理后的 DWI 数据计算弥散张量和峰度张量指标，并将结果配准到 MNI 空间。

## 文件清单

### dti.m（App Designer 主界面）

**入口函数**。运行后打开参数界面，勾选需要计算的指标，支持批量处理。

**可选指标**：

| 指标 | 函数 | 对应命令 | 说明 |
|------|------|---------|------|
| DT | `dt.m` | `dwi2tensor` | 扩散张量 |
| DKI | `dkt.m` | `dwi2tensor -dkt` | 扩散峰度张量 |
| FA | `fa.m` | `tensor2metric -fa` | 各向异性分数 |
| AD | `ad.m` | `tensor2metric -ad` | 轴向扩散率 |
| RD | `rd.m` | `tensor2metric -rd` | 径向扩散率 |
| ADC | `adc.m` | `tensor2metric -adc` | 表观扩散系数 |
| CL | `cl.m` | `tensor2metric -cl` | 线性各向异性 |
| CP | `cp.m` | `tensor2metric -cp` | 平面各向异性 |
| CS | `cs.m` | `tensor2metric -cs` | 球形各向异性 |
| AK | `ak.m` | `tensor2metric -ak` | 轴向峰度（需 DKI） |
| MK | `mk.m` | `tensor2metric -mk` | 平均峰度（需 DKI） |
| RK | `rk.m` | `tensor2metric -rk` | 径向峰度（需 DKI） |

### 其他函数

| 函数 | 说明 |
|------|------|
| `tensor2T1.m` | 将张量图像配准到 T1 空间 |

## 处理流程

1. `dt.m` 先拟合扩散张量 → `dkt.m` 再拟合峰度张量（若勾选 DKI）
2. 逐个计算勾选的指标（FA/AD/RD 等），自动生成 MNI 空间结果
3. 所有指标输出为 .nii 格式，在 MNI 空间

## 参数建议

- FA 阈值（用于 FA 算法选择体素）：默认 0.7
- MNI 模板路径：`Templates/MNI152.nii.gz`
- 如果不需要组分析，可跳过 MNI 配准
