# FBA CLI 参考（21 步完整流程）

## 流程

```
个体处理（Step 1-6）→ 模版构建（Step 7）→ 配准（Step 8-9）
→ Fixel分析（Step 10-16）→ 纤维追踪/连接（Step 17-18）
→ 平滑/统计（Step 19-20）→ 可视化（Step 21）
```

数据目录结构建议：

```
FBA/
├── sub-001/          # 每个被试独立目录
├── sub-002/
├── templates/        # 模版和群体文件
└── stats/            # 统计结果
```

每个被试目录下应有：DWI (.mif/.nii)、bvecs、bvals、mask。

---

## Step 1: 估计响应函数

对每个被试：

```bash
# dhollander 算法（多组织，推荐）
dwi2response dhollander dwi.mif wm.txt gm.txt csf.txt -mask mask.mif

# 或 tournier 算法（单组织）
dwi2response tournier dwi.mif response.txt -mask mask.mif
```

## Step 2: 计算平均响应函数

把所有被试的响应函数文件收集到一起，取平均：

```bash
responsemean sub-*/wm.txt group_average_wm.txt
responsemean sub-*/gm.txt group_average_gm.txt
responsemean sub-*/csf.txt group_average_csf.txt
```

## Step 3: DWI 上采样

```bash
mrgrid dwi.mif regrid -vox 1.25 dwi_upsampled.mif
# -vox 1.25 与 Brainnetome 图谱分辨率匹配
# 可根据需要调整，如 1.0 或 1.5
```

## Step 4: 上采样后掩膜

```bash
dwi2mask dwi_upsampled.mif mask_upsampled.mif
```

## Step 5: CSD 计算

```bash
dwi2fod msmt_csd dwi_upsampled.mif \
  group_average_wm.txt wm_fod.mif \
  group_average_gm.txt gm.mif \
  group_average_csf.txt csf.mif \
  -mask mask_upsampled.mif
```

## Step 6: FOD 标准化

```bash
mtnormalise wm_fod.mif wm_fod_norm.mif \
  gm.mif gm_norm.mif \
  csf.mif csf_norm.mif \
  -mask mask_upsampled.mif
```

## Step 7: 群体模版构建

详见 `references/register.md`「阶段二：Step 7」。

推荐使用 symlink 方式（避免移动大量数据）：

```bash
# 建立 symlink 目录收集所有被试 FOD 和 mask
mkdir -p templates/fod_input templates/mask_input
for sub in sub-001 sub-002 sub-003; do
  ln -sf ${PWD}/sub-001/wmfod_norm.mif templates/fod_input/sub-001.mif
  ln -sf ${PWD}/sub-001/dwi_mask_upsampled.mif templates/mask_input/sub-001.mif
done

# 构建群体模板（带掩膜约束）
population_template \
  templates/fod_input/ \
  -mask_dir templates/mask_input/ \
  templates/wmfod_template.mif \
  -voxel_size 1.25
```

关键参数：

| 参数 | 说明 |
|------|------|
| `-mask_dir` | 个体掩膜目录，约束配准范围 |
| `-voxel_size` | 模板体素大小，与上采样一致（1.25mm） |
| `-nl` | 使用非线性配准（推荐） |
| `-nl_scale` | 非线性缩放层级，默认 `0.25,0.5,0.75,1.0` |
| `-nl_niter` | 每层迭代次数，默认 `5,5,10,20` |

## Step 8: 非线性配准到模版

详见 `references/register.md`「阶段二：Step 8」。

```bash
mrregister wmfod_norm.mif \
  -mask1 dwi_mask_upsampled.mif \
  templates/wmfod_template.mif \
  -nl_warp subject2template_warp.mif \
            template2subject_warp.mif \
  -nl_scale 0.5,0.75,1.0 \
  -nl_niter 5,5,15 \
  -force
```

参数详解：

| 参数 | 含义 | 推荐值 |
|------|------|--------|
| `-mask1` | 源图像掩膜（加速 + 约束） | 上采样 mask |
| `-nl_warp` | 输出变形场（正向 + 反向） | - |
| `-nl_scale` | 多分辨率缩放层级 | `0.5,0.75,1.0` |
| `-nl_niter` | 每层迭代次数 | `5,5,15`（最后层精调） |
| `-type` | 配准类型 | `nonlinear`（默认已包含） |

## Step 9: 掩膜交集

详见 `references/register.md`「阶段二：Step 9」。

```bash
# 个体 mask → 模版空间
mrtransform dwi_mask_upsampled.mif \
  -warp subject2template_warp.mif \
  -interp nearest -datatype bit \
  dwi_mask_in_template_space.mif -force

# 所有 mask 取交集
for sub in sub-*; do
  mrtransform $sub/dwi_mask_upsampled.mif \
    -warp $sub/subject2template_warp.mif \
    -interp nearest -datatype bit \
    $sub/dwi_mask_in_template_space.mif -force
done
mrmath sub-*/dwi_mask_in_template_space.mif min \
  templates/template_mask.mif -datatype bit -force
```

## Step 10: Fixel 掩膜

```bash
fod2fixel \
  -mask templates/template_mask.mif \
  -fmls_peak_value 0.1 \
  templates/wmfod_template.mif \
  templates/fixel_mask \
  -force
```

> `-fmls_peak_value`：FOD 峰值阈值，低于此值的 fixel 被剔除。默认 0.1，对于噪声大或分辨率低的数据可适当降低（如 0.05）。

## Step 11: FOD 变换到模版空间（不重定向）

```bash
# -reorient_fod no: 暂不重定向，后续 step13 fixelreorient 独立处理
mrtransform wmfod_norm.mif \
  -warp subject2template_warp.mif \
  -reorient_fod no \
  fod_in_template_space_NOT_REORIENTED.mif \
  -force
```

## Step 12: 计算 FD（Fiber Density）

从未重定向的 FOD 中提取 AFD（Apparent Fiber Density）：

```bash
fod2fixel -afd fod_in_template_space_NOT_REORIENTED.mif \
  templates/fixel_mask \
  fixel_in_template_space_NOT_REORIENTED \
  -force
```

输出目录 `fixel_in_template_space_NOT_REORIENTED/` 包含 `index.mif` 和 `fd.mif`。

## Step 13: Fixel 方向重定向

用 warp 场重定向 fixel 方向：

```bash
fixelreorient \
  fixel_in_template_space_NOT_REORIENTED \
  subject2template_warp.mif \
  fixel_in_template_space \
  -force
```

## Step 14: Fixel 对应关系

建立个体 fixel 到模板 fixel 的对应关系：

```bash
fixelcorrespondence \
  fixel_in_template_space/fd.mif \
  templates/fixel_mask \
  templates/fd \
  sub-001.mif \
  -force
```

## Step 15: 计算 FC（Fiber Cross-section）

用变形场提取纤维横截面积变化：

```bash
mkdir -p templates/fc
warp2metric \
  subject2template_warp.mif \
  -fc templates/fixel_mask \
  templates/fc \
  sub-001.mif \
  -force
```

## Step 16: 计算 log(FC) 和 FDC

```bash
mrcalc fc.mif -log log_fc.mif
mrcalc fd_corresp.mif fc.mif -multiply fdc.mif
```

## Step 17: 模版空间纤维追踪

在模版 FOD 上做追踪，用于构建 fixel-fixel 连接矩阵：

```bash
# 全脑追踪
tckgen templates/fod_template.mif templates/tracks.tck \
  -seed_dynamic templates/fod_template.mif \
  -select 20000000 \
  -maxlength 250 \
  -minlength 10 \
  -step 0.5

# SIFT 滤波
tcksift templates/tracks.tck templates/fod_template.mif \
  templates/tracks_sift.tck \
  -term_number 2000000
```

## Step 18: Fixel 连接矩阵

```bash
fixelconnectivity templates/fixel_mask templates/tracks_sift.tck templates/connectivity/
```

## Step 19: 平滑

```bash
fixelfilter fd/fd.mif smooth fd_smooth.mif \
  -connectivity templates/connectivity/ \
  -template templates/fixel_mask/ \
  -kernel 10mm

fixelfilter log_fc.mif smooth log_fc_smooth.mif \
  -connectivity templates/connectivity/ \
  -template templates/fixel_mask/ \
  -kernel 10mm

fixelfilter fdc.mif smooth fdc_smooth.mif \
  -connectivity templates/connectivity/ \
  -template templates/fixel_mask/ \
  -kernel 10mm
```

## Step 20: 统计检验

准备设计矩阵 `design.txt` 和对比矩阵 `contrast.txt`。

设计矩阵示例（两组比较，每组 3 个被试）：

```
1 1 0    # sub-001 group A
1 1 0    # sub-002 group A
1 1 0    # sub-003 group A
1 0 1    # sub-004 group B
1 0 1    # sub-005 group B
1 0 1    # sub-006 group B
```

对比矩阵示例（A vs B）：

```
0 1 -1
```

运行统计（CFE 参数可调）：

```bash
 fixelcfestats \
  sub-*/fd_smooth.mif \
  design.txt \
  contrast.txt \
  templates/fixel_mask/ \
  templates/connectivity/ \
  stats/fd/ \
  -cfe_h 2.0 \
  -cfe_e 0.5 \
  -cfe_c 0.5 \
  -nshifts 5000

fixelcfestats \
  sub-*/log_fc_smooth.mif \
  design.txt \
  contrast.txt \
  templates/fixel_mask/ \
  templates/connectivity/ \
  stats/log_fc/

fixelcfestats \
  sub-*/fdc_smooth.mif \
  design.txt \
  contrast.txt \
  templates/fixel_mask/ \
  templates/connectivity/ \
  stats/fdc/
```

关键参数：

| 参数 | 说明 |
|------|------|
| `-nshifts` | 置换次数，默认 5000 |
| `-connectivity` | 连接矩阵目录（Step 18 输出） |

## Step 21: 可视化

```bash
mrview templates/fod_template.mif \
  -fixel.load stats/fd/FD_smooth.mif
```

## 关键参数速查

| 参数 | FBA 推荐值 | 说明 |
|------|-----------|------|
| 上采样体素 | 1.25mm | 匹配 Brainnetome 图谱 |
| CSD 算法 | msmt_csd | 多组织多壳层 |
| 响应函数 | dhollander | 自动估计 WM/GM/CSF |
| 置换检验 | 5000 次 | 标准推荐 |
| 平滑核 | 10mm FWHM | FBA 标准 |
| 追踪纤维数 | 20M → SIFT 2M | 用于连接矩阵 |
