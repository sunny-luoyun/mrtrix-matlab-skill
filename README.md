# mrtrix-matlab-skill

MRtrix3 扩散磁共振图像处理的skill。

## 使用前提

1. 安装 [MRtrix3](https://www.mrtrix.org/) 等依赖工具
2. 克隆 [mrtrix-matlab](https://github.com/sunny-luoyun/mrtrix-matlab) 项目到本地

## 安装

```bash
# 克隆本技能仓库
git clone git@github.com:sunny-luoyun/mrtrix-matlab-skill.git

# 将 skill 目录链接到 (以opencode为例）
ln -s $(pwd)/mrtrix-matlab-skill ~/.config/opencode/skills/mrtrix-matlab-skill
```
或直接将 `SKILL.md` 和 `references/` 目录复制到 `~/.config/opencode/skills/` 下。

## 首次使用配置

**重要**：安装后，用文本编辑器打开 `SKILL.md`，找到文件顶部的「项目路径配置」段落，
将 `MRTRIX_MATLAB_HOME` 改为你电脑上 `mrtrix-matlab` 项目的实际路径：

```text
MRTRIX_MATLAB_HOME = /Users/你的用户名/code/mrtrix-matlab
```

AI 每次读取本文件时会自动使用你填的路径，**一次配置永久有效**。
如果你换了电脑或移动了项目目录，记得更新这个路径。

