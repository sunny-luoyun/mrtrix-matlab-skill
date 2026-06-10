# 连接组矩阵 CLI 参考

## 流程

纤维追踪结果 + 脑图谱 → `tck2connectome` → 连接组矩阵

## 命令

```bash
# 基本连接组（计数）
tck2connectome tracks_sift.tck atlas.mif connectome.csv

# 带权重（SIFT2 权重或 FA 等标量）
tck2connectome tracks_sift.tck atlas.mif connectome_weighted.csv \
  -tck_weights_in weights.csv

# 按体素体积归一化（invnodevol）
tck2connectome tracks_sift.tck atlas.mif connectome_norm.csv \
  -scale_invnodevol \
  -tck_weights_in weights.csv

# 对称化连接组
connectomeedit connectome.csv connectome_sym.csv -symmetrise
```

## 图谱选项

| 图谱 | 体数 | 说明 |
|------|------|------|
| aal.mif | 116 | AAL 模板 |
| Brainnetome | 246 | Brainnetome 图谱 |
| Desikan-Killiany | 84 | 基于 FreeSurfer |

图谱需要先通过 `labelconvert` 转为 MRtrix 兼容格式：

```bash
labelconvert atlas.nii.gz LUT_in.txt LUT_out.txt atlas.mif
```

## 统计

```bash
# 连接组级别统计
connectomestats connectome.csv design.txt contrast.txt stats/ \
  -options edge
```
