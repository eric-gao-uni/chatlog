# Chatlog 故障排除指南

## 数据库找不到问题 (Database Not Found Issue)

### 问题描述

在启动 Chatlog 服务后，API 返回空结果或找不到数据库文件的错误：

```
联系人API返回空结果
会话API找不到数据库文件
API响应: {"items":[],"total":0}
```

### 错误原因分析

#### 1. 平台版本检测错误

- **现象**：系统错误地检测为 v4 格式，但实际应该是 darwin v3 格式
- **影响**：导致数据库文件路径和文件名模式匹配错误
- **原因**：平台检测逻辑在某些情况下可能不准确

#### 2. 数据库文件加密

- **现象**：即使找到数据库文件，也无法读取数据
- **影响**：所有 API 返回空结果
- **原因**：WeChat 数据库文件默认是加密的，需要解密才能读取

#### 3. 文件锁定冲突

- **现象**：数据库文件被占用，无法访问
- **影响**：服务无法读取数据库内容
- **原因**：运行中的 WeChat 应用锁定了数据库文件

### 解决方案

#### 步骤 1: 关闭 WeChat 应用

```bash
# 完全退出 WeChat 应用
# 在 macOS 上，确保从菜单栏彻底退出微信，而不仅仅是关闭窗口
```

#### 步骤 2: 解密 WeChat 数据库

```bash
# 执行数据库解密（这是关键步骤）
./bin/chatlog decrypt --platform darwin --user-data-dir /Users/eric_gao/Library/Containers/com.tencent.xinWeChat/Data/Library/Application\ Support/com.tencent.xinWeChat
```

**注意**：这个解密步骤至关重要！如果跳过此步骤，即使服务启动成功，API 仍然会返回空结果。

#### 步骤 3: 重启 Chatlog 服务

```bash
# 停止当前服务（如果正在运行）
pkill -f "chatlog server"

# 等待进程完全停止
sleep 2

# 重新启动服务
./bin/chatlog server --host 0.0.0.0 --port 5030 --platform darwin --user-data-dir /Users/eric_gao/Library/Containers/com.tencent.xinWeChat/Data/Library/Application\ Support/com.tencent.xinWeChat
```

#### 步骤 4: 验证服务状态

```bash
# 检查健康状态
curl http://localhost:5030/health

# 测试 API 是否返回数据
curl "http://localhost:5030/api/v1/session?format=json" | head -10
curl "http://localhost:5030/api/v1/contact?format=json" | head -10
```

### 关键参数说明

- `--platform darwin`：明确指定平台为 darwin（macOS）
- `--user-data-dir`：指定 WeChat 数据目录的完整路径
- 路径中的空格需要用反斜杠转义

### 重要提醒

**⚠️ 解密步骤是必须的！**

- WeChat 数据库文件默认是加密的
- 必须先执行 `chatlog decrypt` 命令进行解密
- 解密后的数据库文件才能被服务正确读取
- 如果跳过解密步骤，所有 API 都会返回空结果

### 成功标志

当问题解决后，你应该看到：

- 健康检查返回：`{"status":"ok"}`
- Session API 返回包含聊天会话的 JSON 数据
- Contact API 返回包含联系人信息的 JSON 数据
- Web 界面可以正常访问：http://localhost:5030

### 预防措施

1. **启动服务前先关闭 WeChat**：避免文件锁定冲突
2. **执行解密步骤**：确保数据库文件已解密
3. **明确指定平台参数**：避免平台检测错误
4. **使用完整路径**：确保数据目录路径正确
5. **定期检查服务状态**：使用健康检查端点监控服务

### 常见错误信息

| 错误信息                  | 可能原因               | 解决方案                     |
| ------------------------- | ---------------------- | ---------------------------- |
| `{"items":[],"total":0}`  | 数据库未解密或路径错误 | **先执行解密**，然后重启服务 |
| `database file not found` | 平台检测错误或路径错误 | 明确指定 `--platform darwin` |
| `file locked`             | WeChat 正在运行        | 关闭 WeChat 应用             |
| `permission denied`       | 文件权限问题           | 检查数据目录访问权限         |

### 技术细节

#### 平台检测逻辑

Chatlog 会根据操作系统和 WeChat 版本自动检测数据库格式：

- **darwin v3**：较新的 macOS WeChat 版本
- **darwin v4**：更新的数据库格式
- 检测错误会导致文件名模式不匹配

#### 数据库解密过程

1. 服务启动时自动检测加密状态
2. 如果检测到加密数据库，会自动执行解密
3. 解密成功后，API 即可正常返回数据

#### 文件锁定机制

- WeChat 运行时会锁定数据库文件防止外部访问
- 必须完全退出 WeChat 才能释放文件锁
- 服务重启后会重新获取文件访问权限

---

_最后更新：2025 年 12 月 26 日_
