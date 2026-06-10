# map — 纤维网络矩阵构建模块

## 用途

从纤维追踪结果构建脑连接组矩阵（connectome），支持多种分配策略和边权重方案。

## 文件清单

### build_map.m（App Designer 主界面）

**入口函数**。配置网络构建参数。

### buildmap.m

**功能**：执行连接矩阵构建。

**核心命令**：`tck2connectome`

## 参数

### 纤维分配策略（assign）

| 选项 | 说明 |
|------|------|
| `voxels` | 基于体素的分配（默认） |
| `radial` | 径向搜索分配 |
| `forward` | 正向分配 |
| `reverse` | 反向分配 |

### 边权重（scale）

| 选项 | 说明 |
|------|------|
| `length` | 纤维长度作为权重 |
| `invlength` | 纤维长度的倒数 |
| `invnodevol` | 节点体积的倒数 |

### 其他选项

| 参数 | 说明 |
|------|------|
| 对称化 | 矩阵是否对称 |
| 对角线置零 | 是否去除自连接 |
| 加权 | 是否加权（binary 或 weighted） |
| 输出 txt | 是否额外输出文本格式 |

## 图谱路径

- AAL: `Templates/aal.nii`
- Brainnetome: `Templates/BrainnetomeAtlas_BNA_MPM_thr25_1.25mm.nii.gz`
- MNI152: `Templates/MNI152.nii.gz`

## 输出

连接矩阵保存到 `brainnet/` 目录，格式为 .txt 或 .csv。

## 前置条件

- 已完成纤维追踪（fiber 模块），有 .tck 文件
- 已有脑图谱文件（AAL/Brainnetome）
