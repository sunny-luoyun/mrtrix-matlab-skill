# 统计分析 CLI 参考

## 图像统计

```bash
# ROI 统计
mrstats image.mif -mask roi.mif -output mean,std,min,max,count

# 多图像均值
mrmath sub-*/fa.mif mean group_mean_fa.mif -axis 3
mrmath sub-*/fa.mif std group_std_fa.mif -axis 3
```

## 簇统计（Voxel 级别）

```bash
mrclusterstats input_images.txt design.txt contrast.txt mask.mif stats/ \
  -nshifts 5000 \
  -threshold 2.3
```

## Fixel 统计

见 FBA 参考（Step 20），核心命令：

```bash
fixelcfestats subject_files.txt design.txt contrast.txt fixel_mask/ connectivity/ stats/ \
  -cfe_h 2.0 -cfe_e 0.5 -cfe_c 0.5 \
  -nshifts 5000
```

## 连接组统计

```bash
connectomestats connectome_list.txt design.txt contrast.txt stats/ \
  -options edge \
  -nshifts 5000
```

## 纤维统计

```bash
tckstats tracks.tck -output length,weighted_length
```

## 设计矩阵与对比矩阵说明

`design.txt` 和 `contrast.txt` 是所有统计命令的输入文件。

### 格式规则

- **design.txt**: 每行一个被试，每列一个变量（多数为组别/协变量）
- **contrast.txt**: 每行一个对比，每列对应 design.txt 中的变量
- 用空格或 tab 分隔
- 行数 = 被试数，列数 = 变量数

### 常见实验设计模板

#### 两组独立样本（每组 3 个被试）

```
# design.txt: [截距, 组A, 组B]
1 1 0    # sub-001 (A组)
1 1 0    # sub-002 (A组)
1 1 0    # sub-003 (A组)
1 0 1    # sub-004 (B组)
1 0 1    # sub-005 (B组)
1 0 1    # sub-006 (B组)

# contrast.txt: A组 > B组
0 1 -1
```

#### 两组配对样本（前后对比）

```
# design.txt: [截距, 后测-前测]
1 0    # sub-001 (前测)
1 0    # sub-002 (前测)
1 0    # sub-003 (前测)
1 1    # sub-001 (后测)
1 1    # sub-002 (后测)
1 1    # sub-003 (后测)

# contrast.txt: 后测 > 前测
0 1
```

#### 单因素三组方差分析（每组 2 个被试）

```
# design.txt: [截距, 组B, 组C] (A为基线)
1 0 0  # sub-001 (A组)
1 0 0  # sub-002 (A组)
1 1 0  # sub-003 (B组)
1 1 0  # sub-004 (B组)
1 0 1  # sub-005 (C组)
1 0 1  # sub-006 (C组)

# contrast.txt:
# 主效应 F 检验（B vs A, C vs A 联合检验）
0 1 0
0 0 1
```

#### 含协变量（年龄作为协变量）

```
# design.txt: [截距, 组别, 年龄]
1 0 25  # sub-001 (对照, 25岁)
1 0 30  # sub-002 (对照, 30岁)
1 0 28  # sub-003 (对照, 28岁)
1 1 27  # sub-004 (患者, 27岁)
1 1 32  # sub-005 (患者, 32岁)
1 1 29  # sub-006 (患者, 29岁)

# contrast.txt: 患者 vs 对照（控制年龄）
0 1 0
```

### 生成建议

与用户确认以下信息后，AI 自动生成 design.txt 和 contrast.txt：

1. **分组信息**：有几组？每组多少被试？
2. **对比方式**：哪两组比较？是否配对？
3. **协变量**：是否有年龄、性别等协变量需要控制？
4. **检验方向**：单侧（A > B）还是双侧（A ≠ B）？
