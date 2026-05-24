---
name: docker-push
description: |
  将 Docker 镜像同步到阿里云容器镜像服务(ACR)。当用户提到拉取/下载/同步/推送 Docker 镜像、
  需要国内加速拉取镜像、或者提到具体的镜像名称(如 node:20、nginx:latest、redis:7 等)时，
  使用此技能。也适用于用户说"帮我拉个镜像"、"推送到阿里云"、"同步镜像到ACR"、
  "images.txt"等场景。即使用户只是提到了一个 Docker 镜像名称而没有明确说"推送"，
  也应该主动使用此技能询问是否需要同步。
---

# Docker 镜像推送到阿里云 ACR

## 概述

本项目通过 GitHub Actions 自动将 Docker Hub 镜像同步到阿里云容器镜像服务(ACR)，
解决国内拉取 Docker Hub 镜像慢或无法访问的问题。

## 工作原理

1. 将镜像名称添加到 `images.txt`
2. 提交并推送到 `main` 分支
3. GitHub Actions 自动触发，拉取镜像并推送到阿里云 ACR
4. 用户通过阿里云地址拉取镜像

## 阿里云镜像地址格式

```
crpi-36qq8mv72daqudw8.cn-guangzhou.personal.cr.aliyuncs.com/flycun/{镜像名:标签}
```

## 执行步骤

当用户给出镜像名称时，按以下步骤操作：

### 第一步：确认镜像名称

确认用户提供的镜像名称格式。支持的格式示例：
- `node:20` — 官方镜像
- `nginx:1.25.3` — 官方镜像带版本
- `kasmweb/nginx:1.25.3` — 带命名空间的镜像
- `--platform linux/arm64 redis:7` — 指定平台

如果用户只给了镜像名没有标签（如 `nginx`），提醒用户确认是否需要指定版本标签，
默认使用 `latest` 也可以。

### 第二步：检查去重

读取当前 `images.txt`，检查该镜像是否已存在。如果已存在：
- 告知用户该镜像已经在列表中
- 询问是否仍然需要推送（可能是重新同步）

### 第三步：添加到 images.txt

将镜像名称追加到 `images.txt` 文件末尾，每行一个镜像。

**重要规则：**
- 不要添加空行或注释，只添加镜像名称本身
- 如果用户指定了 `--platform`，保留在镜像名称前面，如：`--platform linux/arm64 redis:7`
- 每行格式为：`[选项] 镜像名:标签`

### 第四步：提交并推送

```bash
git add images.txt
git commit -m "添加镜像: {镜像名}"
git push origin main
```

推送后 GitHub Actions 会自动触发构建。

### 第五步：告知用户结果

推送成功后，向用户提供以下信息：

1. **已触发同步**：告知 GitHub Actions 已触发
2. **拉取命令**：

```bash
docker pull crpi-36qq8mv72daqudw8.cn-guangzhou.personal.cr.aliyuncs.com/flycun/{镜像名:标签}
```

3. **注意事项**：
   - 同步通常需要几分钟，取决于镜像大小
   - 可以在 GitHub Actions 页面查看进度
   - 对于指定了 `--platform` 的镜像，阿里云地址前会加上平台前缀

### 批量处理

如果用户一次提供多个镜像，全部添加到 `images.txt` 后一次性提交推送。
提交信息中列出所有添加的镜像。

## 特殊情况处理

### 重名镜像

如果 `images.txt` 中存在不同命名空间下的同名镜像（如 `a/nginx` 和 `b/nginx`），
工作流会自动在阿里云地址中添加命名空间前缀进行区分。
阿里云地址变为：`.../flycun/{命名空间}_{镜像名:标签}`

### 多平台镜像

带 `--platform` 的镜像在阿里云中会自动加平台前缀。
阿里云地址变为：`.../flycun/{平台前缀}{镜像名:标签}`
例如：`--platform linux/arm64 redis:7` → `.../flycun/linux_arm64_redis:7`

### 镜像带摘要

如果镜像名称包含 `@sha256:...`，工作流会自动去除摘要部分。
用户只需提供基本的镜像名:标签即可。
