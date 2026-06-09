---
title: Obsidian + Firefly + GitHub 自动发布系统（macOS LaunchAgent）
published: 2026-06-10
update: 2026-06-10
description: 这是一套实现「Obsidian 写作 → 自动同步 Firefly 博客 → GitHub 自动部署」的完整方案，核心目标是：**写完 Markdown 后自动发布，无需手动 git 操作**
tags:
  - 博客
  - Firefly
  - "#Git"
category: 教程
draft: false
author: rain
password: "851150"
---


# 1. 系统架构
整体数据流：
```

Obsidian Vault  
↓（软链接）  
5-Publish/Publish  
↓  
Firefly/src/content/posts  
↓  
Git commit + push（自动化）  
↓  
GitHub  
↓  
自动部署（Cloudflare / Vercel / GitHub Pages）

````
---

# 2. 关键设计：软链接

在 Obsidian 中创建指向 Firefly 博客目录的软链接：

```bash
ln -s /Users/woo/Firefly/src/content/posts \
"/Users/woo/Library/Mobile Documents/iCloud~md~obsidian/Documents/rain/5-Publish/Publish"
````

验证：

```bash
ls -l 5-Publish/Publish
```

正确结果应为：

```
Publish -> /Users/woo/Firefly/src/content/posts
```

---

# **3. Git 仓库设计原则**

⚠️ 不建议在 Obsidian Vault 初始化 Git

如果误初始化，可以取消：

```bash
rm -rf .git
```

推荐原则：

- Obsidian：只写作，不管理 Git
- Firefly：唯一 Git 发布源

---

# **4. 自动化方案选择**

## **❌ 不推荐（实时监听）**

如 fswatch：

- 每次保存都触发 commit/push
- Git 历史极度混乱

---

## **✅ 推荐（LaunchAgent 定时同步）**

- 每 5~10 分钟检查一次
- 有变更才提交
- 无变更直接退出

---

# **5. 同步脚本（核心逻辑）**

创建脚本：

```bash
~/scripts/firefly-sync.sh
```

内容：

```bash
#!/bin/bash

cd ~/Firefly || exit 1

# 只检查博客内容是否变化
if [[ -z $(git status --porcelain src/content/posts) ]]; then
    exit 0
fi

git add src/content/posts

git commit -m "auto sync $(date '+%Y-%m-%d %H:%M:%S')" || exit 0

git push origin master
```

赋予执行权限：

```bash
chmod +x ~/scripts/firefly-sync.sh
```

---

# **6. LaunchAgent 定时任务**

创建文件：

```bash
~/Library/LaunchAgents/com.firefly.autosync.plist
```

内容：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
"http://www.apple.com/DTDs/PropertyList-1.0.dtd">

<plist version="1.0">
<dict>

    <key>Label</key>
    <string>com.firefly.autosync</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/woo/scripts/firefly-sync.sh</string>
    </array>

    <key>StartInterval</key>
    <integer>600</integer>

    <key>RunAtLoad</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/firefly-sync.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/firefly-sync.err</string>

</dict>
</plist>
```

说明：

- `600` = 每 10 分钟执行一次
- `RunAtLoad` = 开机立即执行一次

---

# **7. 加载 LaunchAgent（macOS 15+）**

加载任务：

```bash
launchctl bootstrap gui/$(id -u) \
~/Library/LaunchAgents/com.firefly.autosync.plist
```

查看状态：

```bash
launchctl list | grep firefly
```

正常输出示例：

```
-   0   com.firefly.autosync
```

---

# **8. 手动测试与验证**

手动触发：

```bash
launchctl kickstart gui/$(id -u)/com.firefly.autosync
```

查看日志：

```bash
cat /tmp/firefly-sync.log
```

示例输出：

```
[master 445061a] auto sync 2026-06-10 00:32:31
 1 file changed, 13 insertions(+)
 create mode 100644 xxx.md
```

---

# **9. 工作效果**

最终实现：

- Obsidian 写 Markdown
- 自动同步到 Firefly
- 自动 commit
- 自动 push GitHub
- 自动触发部署

---

# **10. 使用体验总结**

最终效果是：

```
写完文章
    ↓
最多 10 分钟内自动上线博客
```

---

# **11. 设计原则总结**

- Obsidian 只负责写作
- Firefly 是唯一 Git 源
- LaunchAgent 负责调度
- 脚本负责判断变更
- Git 只记录“真正的内容变化”

核心思想：

不监听文件变化，而是周期性检测 + 条件提交

这是 macOS 上最稳定的轻量自动发布方案。