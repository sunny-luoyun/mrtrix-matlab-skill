# 数据整理 CLI 参考

## DICOM → .mif

```bash
# mrconvert 可以直接读取 DICOM 目录
mrconvert dicom_dir/ dwi.mif

# 查看 DICOM 信息
dcminfo dicom_dir/
```

## DICOM → NIfTI

```bash
# 按系列分组转换
mrconvert dicom_dir/ dwi.nii.gz -export_series all

# 指定系列
mrconvert dicom_dir/ dwi.nii.gz -export_series 4
```

## NIfTI → .mif

```bash
mrconvert dwi.nii.gz dwi.mif -fslgrad bvecs bvals
```

## .mif → NIfTI

```bash
mrconvert dwi.mif dwi.nii.gz -export_grad_fsl bvecs bvals
```

## 批量处理建议

```bash
# 遍历所有子目录转换
for dir in /path/to/raw/sub-*/; do
  sub=$(basename $dir)
  mrconvert $dir/dicom/ ${sub}/dwi.mif
done
```
