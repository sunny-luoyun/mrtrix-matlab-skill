# mrtrix3-cli-skill

MRtrix3 扩散磁共振图像处理 CLI 技能。AI 可直接在终端中执行 MRtrix3 命令对数据进行处理。

## 前提

系统已安装 [MRtrix3](https://www.mrtrix.org/download/)，且 `mrconvert` 等命令在 PATH 中可用。

## 安装

```bash
# 将 skill 链接到配置目录(opencode为例)
ln -s mrtrix-skill ~/.config/opencode/skills/mrtrix3-cli
```

## 使用

在对话中提及需要处理弥散/扩散磁共振数据即可自动触发。
