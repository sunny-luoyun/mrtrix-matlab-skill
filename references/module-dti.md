# DTI CLI 参考

## 流程

预处理 DWI → `dwi2tensor` → `tensor2metric` → 复用 `dwi_to_MNI` 矩阵配准

## 命令序列

```bash
# 1. 张量拟合
dwi2tensor dwi_prepro.mif dt.mif -mask mask.mif

# 2. 计算弥散指标
tensor2metric dt.mif \
  -fa fa.mif \
  -ad ad.mif \
  -rd rd.mif \
  -adc adc.mif \
  -cl cl.mif \
  -cp cp.mif \
  -cs cs.mif \
  -mask mask.mif

# 3. (可选) DKI 指标
dwi2tensor dwi_prepro.mif dkt.mif -mask mask.mif -dkt
tensor2metric dkt.mif \
  -mk mk.mif \
  -ak ak.mif \
  -rk rk.mif \
  -mask mask.mif
```

## 配准到 MNI

详见 `references/register.md`「阶段一：线性配准到 MNI」。

**核心原则**：只配一次 `dwi→MNI`，所有 DTI 指标复用同一矩阵。

```bash
# 0. 先配准 DWI b0 → MNI（6 DOF，得到 dwi_to_MNI_mrtrix.txt）
# 详见预处理模块 Step 7b

# 1. 所有 DTI 指标批量变换（复用同一矩阵）
for metric in fa ad rd adc cl cp cs ak mk rk; do
  mrtransform ${metric}.mif ${metric}_MNI.nii.gz \
    -linear dwi_to_MNI_mrtrix.txt \
    -template Templates/MNI152.nii.gz \
    -force
done
```

## 方法对比（旧写法 vs 你的写法）

| 方法 | 命令 | 说明 |
|------|------|------|
| ❌ 旧写法（我之前的 skill） | `mrregister -type affine` 每次现配 | 每个指标各自配准，重复计算且可能不一致 |
| ✅ **你的写法** | `flirt` → `transformconvert` → `mrtransform -linear` | 一次配准，所有指标复用同一矩阵，空间完全对齐 |
