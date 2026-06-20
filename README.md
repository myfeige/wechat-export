# Claude Code Skills

我的 Claude Code 技能集合。每个技能是一个 Markdown 文件，放在 `~/.claude/skills/` 目录下即可被 Claude Code 识别。

---

## wechat-export：微信聊天记录一键导出

### 这是什么

一个 Claude Code 技能，让你在对话中直接导出微信好友的完整聊天记录。**不需要打开任何 GUI 工具**，告诉 Claude Code 好友名字就行，全自动完成：

1. 从微信进程提取数据库密钥（DLL hook）
2. 解密 SQLCipher 加密的消息数据库
3. 找到目标联系人、导出全部消息
4. 解密聊天中的图片（WeChat V2 加密格式）
5. 生成 HTML（图片内嵌、仿微信气泡 UI）+ 纯文本 TXT + 结构化 JSON

### 安装方法

```bash
# 1. 下载技能文件，放到 Claude Code 的 skills 目录
cp skills/wechat-export.md ~/.claude/skills/

# 2. 安装 Python 依赖（一次性）
pip install pycryptodome zstandard

# 3. 确保已安装 Node.js（运行密钥提取脚本用）
```

首次使用时会自动检查并下载两个开源工具：
- [WeChatExport](https://github.com/Ray0612/WeChat-Export-Tool) — 密钥提取
- [wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt) — 数据库解密 + 图片解密

### 使用方法

在 Claude Code 对话中说：

```
导出微信上和张三的聊天记录
```

或者用斜杠命令：

```
/wechat-export 张三
```

Claude Code 会自动执行全部流程，过程中可能需要你：
- **扫码登录微信**（如果微信未登录）
- **点一次 UAC 确认**（密钥提取需要管理员权限）

### 输出结果

```
D:\微信聊天记录\导出的聊天记录\张三\
├── 张三.html    # 双击浏览器打开，仿微信气泡UI，图片内嵌
├── 张三.txt     # 纯文本聊天内容，不含代码
├── 张三.json    # 完整结构化数据
└── images/        # 解密后的图片文件
```

### 原理简述

微信 PC 版的聊天记录存储在本地加密的 SQLite 数据库中（SQLCipher 4，AES-256-CBC）。密钥仅在微信启动时出现在进程内存中。流程是：

```
关闭微信 → 启动 DLL hook → 启动微信 → 捕获 passphrase
    → PBKDF2 派生每库密钥 → 解密 SQLite → 查询消息
    → 解密 V2 图片 → base64 内嵌 → 生成 HTML
```

### 兼容性

- Windows 10/11
- 微信 PC 版 4.1.x
- 理论上支持 macOS/Linux（需替换密钥提取方式）

### 注意事项

- 所有操作在本地完成，聊天数据不会上传到任何服务器
- 仅供个人数据备份使用
