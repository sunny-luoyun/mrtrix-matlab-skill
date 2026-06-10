# fiber — 纤维重建模块

## 用途

从 FOD 数据进行全脑或 ROI 纤维追踪，支持 SIFT/SIFT2 滤波和 Track Density Imaging (TDI) 图生成。

## 文件清单

### fiber.m（App Designer 主界面）

**入口函数**。配置追踪参数、ROI、滤波选项。

### 核心处理函数

| 函数 | 对应命令 | 说明 |
|------|---------|------|
| `fiberbuild.m` | `tckgen` | 全脑或 ROI 纤维追踪，自动选择最近 FOD 文件 |
| `sift.m` | `tcksift` | SIFT 滤波，减少纤维数量但保持密度分布 |
| `weightc.m` | `tcksift2` | SIFT2 权重计算，为每条纤维分配权重 |
| `tck2nii.m` | `tckmap` | TDI 图生成，支持平滑和权重 |

### ROI 工具

| 函数 | 说明 |
|------|------|
| `ROIListDialog.m` | ROI 定义对话框，支持：球形 ROI、MASK 文件 ROI、AAL 模板 ROI、Brainnetome 模板 ROI |
| `utils_tal2icbm_spm.m` | Talairach → ICBM SPM 坐标转换 |

## 追踪参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| 角度 | 45° | 最大转弯角度 |
| 最小长度 | 10 mm | 纤维最小长度 |
| 最大长度 | 200 mm | 纤维最大长度 |
| FOD 阈值 | 0.1 | FOD 幅值阈值，低于此值停止追踪 |
| 步数 | 1000 | 追踪迭代步数 |
| 纤维数量 | 500000 | 输出纤维总数 |

## SIFT 参数

| 参数 | 说明 |
|------|------|
| 纤维数量 | SIFT 滤波后保留的纤维目标数 |
| 迭代次数 | 默认 20 |

## TDI 参数

| 参数 | 说明 |
|------|------|
| 体素大小 | 输出 TDI 图的分辨率（mm） |
| 平滑 | 高斯平滑核大小（mm） |
| 使用权重 | 是否使用 SIFT2 权重 |

## 处理流程

1. 选择追踪模式：全脑追踪 / ROI 追踪
2. 配置追踪参数（角度/长度/阈值/纤维数）
3. 执行追踪 → `fiberbuild.m`
4. 可选：SIFT 滤波 → `sift.m`
5. 可选：SIFT2 权重 → `weightc.m`
6. 可选：TDI 图 → `tck2nii.m`
