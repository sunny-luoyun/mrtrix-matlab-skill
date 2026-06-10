# FOD/CSD CLI 参考

## 流程

预处理 DWI → `dwi2response` → `dwi2fod` → `mtnormalise`

## 响应函数算法选择

| 算法 | 适用场景 | 原理 |
|------|---------|------|
| `dhollander` | 多壳层，无 T1 | 全自动，利用多壳层信息区分 WM/GM/CSF |
| `msmt_5tt` + `tournier` | 有 T1，T1 质量好 | 利用 5tt 分割约束组织 mask，tournier 迭代选 WM |
| `msmt_5tt` + `tax` | 有 T1，T1 质量好 | 同上，用 tax 阈值法选 WM |
| `msmt_5tt` + `fa` | 有 T1，T1 质量好 | 同上，用 FA 阈值选 WM（需调 FA 阈值） |
| `tournier` | 单壳层，无 T1 | 迭代选择单纤维体素 |
| `tax` | 单壳层/临床 | 快速阈值法 |

## dhollander（推荐，多壳层无 T1）

```bash
dwi2response dhollander dwi_prepro.mif wm.txt gm.txt csf.txt -mask mask.mif
```

## msmt_5tt（5TT 引导，需 T1）

先由 5tt 分割提供组织先验，再用指定算法估计 WM 响应函数：

```bash
# 1. 生成 5tt
5ttgen fsl T1.nii.gz 5tt.mif

# 2. 5TT 引导响应函数估计（三种 WM 算法选一）
# B1: tournier 迭代法
dwi2response msmt_5tt dwi_prepro.mif wm.txt gm.txt csf.txt \
  -mask mask.mif -5tt 5tt.mif -wm_algo tournier
# B2: tax 阈值法
# dwi2response msmt_5tt dwi_prepro.mif wm.txt gm.txt csf.txt \
#   -mask mask.mif -5tt 5tt.mif -wm_algo tax
# B3: FA 阈值法
# dwi2response msmt_5tt dwi_prepro.mif wm.txt gm.txt csf.txt \
#   -mask mask.mif -5tt 5tt.mif -wm_algo fa
```

## tournier / tax（单壳层，无 T1）

```bash
dwi2response tournier dwi_prepro.mif response.txt -mask mask.mif
# dwi2response tax dwi_prepro.mif response.txt -mask mask.mif
```

## FOD 计算与标准化

### 多组织 MSMT-CSD（与 dhollander / msmt_5tt 搭配）

```bash
dwi2fod msmt_csd dwi_prepro.mif \
  wm.txt wm_fod.mif \
  gm.txt gm.mif \
  csf.txt csf.mif \
  -mask mask.mif
mtnormalise wm_fod.mif wm_fod_norm.mif \
  gm.mif gm_norm.mif \
  csf.mif csf_norm.mif \
  -mask mask.mif
```

### 单组织 CSD（与 tournier / tax 搭配）

```bash
dwi2fod csd dwi_prepro.mif response.txt fod.mif -mask mask.mif
mtnormalise fod.mif fod_norm.mif -mask mask.mif
```

## 质量检查

```bash
# 查看 FOD
mrview wm_fod_norm.mif -mode 2
shview wm_fod_norm.mif

# 检查峰值
sh2peaks wm_fod_norm.mif peaks.mif -num 3
```
