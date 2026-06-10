# mrtrix-matlab-skill

MRtrix3 扩散磁共振图像处理的skill。

## 使用前提

1. 安装 [MRtrix3](https://www.mrtrix.org/)、[FSL](https://fsl.fmrib.ox.ac.uk/)、[ANTs](http://stnava.github.io/ANTs/) 等依赖工具
2. 克隆 [mrtrix-matlab](https://github.com/sunny-luoyun/mrtrix-matlab) 项目到本地
3. 按项目 README 配置好环境（运行 `setup_path` 添加路径，配置 `env.m` 中的工具路径）

## 安装

```bash
# 克隆本技能仓库
git clone git@github.com:sunny-luoyun/mrtrix-matlab-skill.git

# 将 skill 目录链接到 opencode
ln -s $(pwd)/mrtrix-matlab-skill ~/.config/opencode/skills/mrtrix-matlab-skill
```

或直接将 `SKILL.md` 和 `references/` 目录复制到 `~/.config/opencode/skills/` 下。

