# wechat-export

导出微信好友的聊天记录，生成 HTML（图片内嵌）+ TXT + JSON。

## 用法

```
/wechat-export <微信好友备注或昵称>
```

或在对话中说：「导出微信上和xxx的聊天记录」

## 输出

```
D:\微信聊天记录\导出的聊天记录\<联系人名>\
├── <联系人名>.html    # 仿微信气泡UI，图片base64内嵌
├── <联系人名>.txt     # 纯聊天文本
├── <联系人名>.json    # 完整结构化数据
└── images/            # 解密后的图片文件
```

## 依赖工具

执行前自动检查并下载：

1. **[WeChatExport](https://github.com/Ray0612/WeChat-Export-Tool)** — 提取数据库密钥
   - 需要 `get_key.js` + `wx_key.dll` + Node.js
   - 不存在则从 GitHub Releases 下载

2. **[wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt)** — 解密数据库 + 图片 + 导出
   - `git clone` + `pip install pycryptodome zstandard`
   - 图片解密必须用官方 `v2_decrypt_file`，绝对不能自己实现

## 执行流程

### 0. 前置检查
- 确认微信已登录且数据库近期有更新
- 读取微信配置找到数据目录（`%APPDATA%\Tencent\xwechat\config\*.ini`）

### 1. 安装依赖
- 检查 WeChatExport 和 wechat-decrypt 是否存在
- 不存在则自动下载
- 检查 Python 依赖

### 2. 提取密钥（需管理员权限）
- 关闭微信 → 启动 `get_key.js`（DLL hook）→ 启动微信 → 捕获 passphrase
- 微信 4.1+ 不能用内存模式扫描（`find_all_keys_windows.py` 扫不到），必须用 DLL hook 或 Frida

### 3. 解密数据库
- Passphrase 通过 PBKDF2-SHA512(salt, 256000 iterations) 派生每库 AES-256 密钥
- 只解必需的：`message_*.db`、`contact/contact.db`、`session/session.db`

### 4. 查找联系人
- 在 `Contact` 表中通过 `remark` 或 `nick_name` 匹配
- 消息表名 = `Msg_` + MD5(wxid)
- zstd 解压消息内容，通过 `real_sender_id` 区分发送者

### 5. 图片解密
- **必须用 `decode_image.v2_decrypt_file`**（V2 格式是 `[AES][RAW][XOR]` 三段式）
- 图片 AES key 需通过 UIN 爆破获取
- 通过 `packed_info_data` 提取本地 MD5 匹配 `.dat` 文件
- base64 内嵌到 HTML

### 6. 生成文件
- HTML：仿微信气泡 UI，图片 base64 内嵌，日期分组，点击放大
- TXT：纯文本，不含 XML/代码，格式 `[时间] 我/对方: 内容`
- JSON：完整结构化数据

## 常见坑

1. **密钥扫不到**：微信 4.1+ 内存中不再存 `x'<hex>'` 格式密钥。必须用 DLL hook 或 Frida hook `sqlite3_key`
2. **图片灰屏**：自己写 V2 解密漏了 RAW/XOR 段。必须用官方 `v2_decrypt_file`
3. **taskkill 杀不掉微信**：需要管理员权限
4. **消息内容为空**：文本消息 `WCDB_CT` 可能为 0，不要只处理压缩情况
5. **图片 AES key**：是 16 字节 ASCII 字符串（MD5 前 16 字符），不是 hex decode
6. **消息表名**：`Msg_` + MD5(wxid)，注意群聊 wxid 后缀 `@chatroom`
