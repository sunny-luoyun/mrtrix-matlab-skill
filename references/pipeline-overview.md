# 管线总览

## 快速决策树

```
用户: "我有弥散数据要处理"

问: 数据什么格式？
├─ DICOM (.dcm/.IMA) → mrconvert dicom_dir/ dwi.mif（一步到位，自动读梯度）
├─ NIfTI (.nii/.nii.gz) → mrconvert dwi.nii.gz dwi.mif -fslgrad bvecs bvals
└─ .mif（已有）→ 无需转换，直接使用

格式转换完成后 → 问: 处理目标？
    ├─ DTI 指标 → pre → dwi2tensor → tensor2metric
    ├─ FOD/CSD  → pre → dwi2response → dwi2fod → mtnormalise
    ├─ 纤维追踪 → pre → FOD → tckgen → tcksift
    ├─ FBA 分析 → FOD → 21步 fixel 流程
    ├─ 连接组   → FOD → tckgen → tck2connectome
    └─ 统计分析 → 确认已有结果 → 选对应统计命令
```

## 输出产物

| 管线 | 关键输出 | 格式 |
|------|---------|------|
| DTI | fa/ad/rd/adc/cl/cp/cs.nii.gz | NIfTI |
| FOD | wm_fod_norm.mif, gm_norm.mif, csf_norm.mif | .mif |
| 纤维追踪 | tracks.tck, tracks_sift.tck, tdi.mif | .tck / .mif |
| FBA | fd/log_fc/fdc_smooth.mif + stats 结果 | .mif |
| 连接组 | connectome.csv | .csv |
