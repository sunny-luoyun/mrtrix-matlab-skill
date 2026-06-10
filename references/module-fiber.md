# 纤维追踪 CLI 参考

## 流程

FOD → 5TT 生成 → `tckgen` → SIFT → TDI

## 命令序列

### 1. 5TT 和 GMWMI 生成

```bash
5ttgen fsl T1.nii.gz 5tt.mif
5tt2gmwmi 5tt.mif gmwmi.mif
```

### 2. 全脑纤维追踪

```bash
tckgen wm_fod_norm.mif tracks.tck \
  -algorithm iFOD2 \
  -act 5tt.mif \
  -backtrack \
  -seed_gmwmi gmwmi.mif \
  -select 1000000 \
  -maxlength 250 \
  -minlength 10 \
  -step 0.5 \
  -angle 45 \
  -power 0.33 \
  -crop_at_gmwmi
```

#### 追踪算法选择（-algorithm）

| 算法 | 全名 | 适用场景 |
|------|------|---------|
| `iFOD2` | 2nd-order Integration over FODs | **默认推荐**，基于 FOD 的概率追踪 |
| `SD_Stream` | Streamline from SD | 基于 FOD 的确定性追踪 |
| `Tensor_Det` | Deterministic tractography | 基于张量的确定性追踪（需 dt.mif） |
| `Tensor_Prob` | Probabilistic tractography | 基于张量的概率追踪（需 dt.mif） |
| `FACT` | Fiber Assignment by Continuous Tracking | 确定性纤维赋值追踪（需 dt.mif） |

> **注意**：Tensor_Det / Tensor_Prob / FACT 需要 `dwi2tensor` 生成的张量图像作为输入，
> 不能直接用 FOD。如用这些算法，输入改为 `dt.mif` 并配合 `-grad` 选项。

关键参数说明：

| 参数 | 推荐值 | 说明 |
|------|--------|------|
| `-algorithm` | iFOD2 | 追踪算法（iFOD2 / SD_Stream / Tensor_Det / Tensor_Prob / FACT）|
| `-select` | 1M-10M | 输出纤维数，越多越密，文件越大 |
| `-maxlength` | 250-300 | 最长纤维长度 (mm) |
| `-minlength` | 10-20 | 最短纤维长度 (mm) |
| `-step` | 0.5-1.0 | 追踪步长 (mm) |
| `-angle` | 30-60 | 最大转向角 (度) |
| `-seed_gmwmi` | gmwmi.mif | 从灰质-白质边界播种 |
| `-seed_dynamic` | fod.mif | 动态播种（按 FOD 幅值） |

### 3. SIFT 滤波

```bash
tcksift tracks.tck wm_fod_norm.mif tracks_sift.tck \
  -act 5tt.mif \
  -term_number 200000
```

### 4. SIFT2（替代 SIFT，输出权重而非子集）

```bash
tcksift2 tracks.tck wm_fod_norm.mif weights.csv \
  -act 5tt.mif
```

### 5. TDI 图（5 种对比度）

```bash
# 对比度 1: tdi（默认）—— 纤维密度图，方向编码彩色（-dec）
tckmap tracks_sift.tck tdi.mif -vox 1.0 -dec -template wm_fod_norm.mif

# 对比度 2: length —— 纤维长度图，每个体素内纤维的平均长度
tckmap tracks_sift.tck length.mif -contrast length -vox 1.0 -template wm_fod_norm.mif

# 对比度 3: invlength —— 纤维长度倒数图
tckmap tracks_sift.tck invlength.mif -contrast invlength -vox 1.0 -template wm_fod_norm.mif

# 对比度 4: fod_amp —— FOD 幅值图，沿纤维方向的 FOD 幅值
tckmap tracks_sift.tck fod_amp.mif -contrast fod_amp -vox 1.0 -template wm_fod_norm.mif

# 对比度 5: curvature —— 纤维曲率图
tckmap tracks_sift.tck curvature.mif -contrast curvature -vox 1.0 -template wm_fod_norm.mif

# 可选: 高斯平滑
# tckmap tracks_sift.tck tdi_smooth.mif -vox 1.0 -dec -template wm_fod_norm.mif -smooth 3
```

## ROI 追踪（需要在特定 ROI 内/间追踪）

```bash
# 从 ROI 播种
tckgen wm_fod_norm.mif tracks_roi.tck \
  -act 5tt.mif \
  -seed_image roi.mif \
  -select 50000

# ROI 间纤维（包含两个 ROI）
tckgen wm_fod_norm.mif tracks_between.tck \
  -act 5tt.mif \
  -seed_image roi1.mif \
  -include roi2.mif \
  -select 50000

# 排除特定区域
tckgen wm_fod_norm.mif tracks_exclude.tck \
  -act 5tt.mif \
  -seed_gmwmi gmwmi.mif \
  -exclude exclude_mask.mif \
  -select 1000000
```

## 质量检查

```bash
tckinfo tracks.tck
tckstats tracks.tck
mrview wm_fod_norm.mif -tractography.load tracks_sift.tck
```
