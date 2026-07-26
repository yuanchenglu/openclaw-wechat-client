# 经验教训

> 本文档记录项目中遇到的问题、原因分析和解决方案，供团队成员和 AI 助手参考。

---

## Gateway Token 配置

**问题**：客户端调用 Gateway API 返回 401 错误

**场景**：微信频道插件无法正常工作，用户收到"认证配置需要更新"的错误提示

**原因分析**：
- 客户端未配置 `OPENCLAW_API_KEY` 环境变量
- Gateway 配置中未启用 token 认证模式
- 客户端无法从 `~/.openclaw/openclaw.json` 读取到有效的 token

**解决方案**：

### 方法一：配置环境变量（推荐）

在 systemd service 文件中添加：
```bash
Environment="OPENCLAW_API_KEY=<your-token>"
```

然后重载服务：
```bash
systemctl --user daemon-reload
systemctl --user restart openclaw-wechat-channel
```

### 方法二：检查 Gateway 配置

1. 检查 Gateway 认证配置：
   ```bash
   cat ~/.openclaw/openclaw.json | jq '.gateway.auth'
   ```

2. 确认 `auth.mode` 和 `auth.token` 是否正确配置：
   ```json
   {
     "gateway": {
       "auth": {
         "mode": "token",
         "token": "your-secret-token-here"
       }
     }
   }
   ```

### 方法三：使用 .env 文件（需要代码支持 dotenv 加载）

创建 `.env` 文件：
```bash
OPENCLAW_API_KEY=<your-token>
```

**检查方法**：
```bash
# 1. 检查环境变量
echo $OPENCLAW_API_KEY

# 2. 检查 Gateway 配置
cat ~/.openclaw/openclaw.json | jq '.gateway.auth'

# 3. 检查服务环境变量
systemctl --user show openclaw-wechat-channel | grep Environment

# 4. 测试 Token 是否有效
curl -H "Authorization: Bearer $OPENCLAW_API_KEY" http://127.0.0.1:18789/v1/models
```

**相关文件**：
- Gateway 配置：`~/.openclaw/openclaw.json`
- 服务配置：`~/.config/systemd/user/openclaw-wechat-channel.service`

**预防措施**：
- 在部署脚本中自动配置 `OPENCLAW_API_KEY`
- 在文档中明确说明 Token 配置要求
- 提供诊断脚本帮助用户快速定位问题

**经验总结**：
- Token 认证是 Gateway 的安全机制，必须正确配置
- 错误信息需要明确区分"Token 缺失"和"Token 无效"两种情况
- 提供具体的诊断步骤和解决方案，降低用户排查难度

---

## Gateway 服务不可用（502/503）

**问题**：客户端收到 502/503 错误，无法正常调用 API

**原因分析**：
- Gateway 服务未启动
- Gateway 服务崩溃或正在重启
- 后端服务（如 AI 模型）不可用

**解决方案**：

```bash
# 1. 检查服务状态
openclaw gateway status

# 2. 如果未运行，启动服务
openclaw gateway start

# 3. 或通过 systemd 管理
systemctl --user status openclaw-gateway
systemctl --user start openclaw-gateway

# 4. 查看错误日志
journalctl --user -u openclaw-gateway -n 50 --no-pager
```

**预防措施**：
- 配置 systemd 自动重启策略
- 添加健康检查脚本
- 监控 Gateway 服务状态

---

## Chat Completions API 未启用（404）

**问题**：客户端收到 404 错误，Chat Completions API 未启用

**原因分析**：
- Gateway 的 `/v1/chat/completions` 端点默认禁用
- OpenClaw 升级或配置重置后恢复默认设置

**解决方案**：

启用 Chat Completions API：
```json
{
  "gateway": {
    "http": {
      "endpoints": {
        "chatCompletions": {
          "enabled": true
        }
      }
    }
  }
}
```

或通过 OpenClaw 对话框发送：
```
请帮我启用 Gateway 的 Chat Completions API：
在 gateway 配置中添加：
"http": {"endpoints": {"chatCompletions": {"enabled": true}}}
```

---

## 权限不足（403）

**问题**：客户端收到 403 错误，请求被拒绝

**原因分析**：
- Gateway 配置了 IP 白名单，当前请求来源 IP 不在允许列表中
- 其他权限配置问题

**解决方案**：

```bash
# 检查 IP 白名单配置
cat ~/.openclaw/openclaw.json | jq '.gateway.security.ipWhitelist'

# 暂时关闭 IP 限制（仅测试用）
# 修改 Gateway 配置，清空或关闭 IP 白名单
```

---

## 连接失败

**问题**：无法连接到 Gateway 服务

**原因分析**：
- Gateway 服务未启动
- 端口配置错误
- 防火墙阻止连接

**诊断步骤**：

```bash
# 1. 测试端口连通性
curl -v http://127.0.0.1:18789/v1/models

# 2. 检查端口是否被监听
netstat -tlnp | grep 18789
# 或
ss -tlnp | grep 18789

# 3. 检查 Gateway 配置的端口
cat ~/.openclaw/openclaw.json | jq '.gateway.http.port'

# 4. 检查服务进程
ps aux | grep openclaw

# 5. 检查防火墙
sudo ufw status
```

---

## 错误信息设计原则

### 问题
早期版本的错误信息过于模糊，例如：
```
⚠️ 认证配置需要更新
【问题描述】
OpenClaw Gateway 的认证配置不正确，导致请求被拒绝。
```

用户和 AI 都难以快速诊断问题：
- 是 Token 缺失？Token 错误？Token 过期？
- 应该如何解决？

### 改进方案
错误信息应包含：
1. **具体原因**：明确说明是什么问题
2. **诊断步骤**：提供可执行的命令
3. **解决方案**：多种可选的解决方法
4. **相关文件**：涉及的配置文件路径

### 示例
改进后的错误信息：
```
⚠️ Gateway Token 未配置

【问题描述】
请求未携带 Gateway Token，导致认证失败（HTTP 401）。

**具体原因**：客户端未找到有效的认证 Token
- 环境变量 OPENCLAW_API_KEY 未设置
- Gateway 配置文件中未启用 token 认证模式

【解决方法】

**方法一：配置环境变量（推荐）**
在 systemd service 文件中添加：
```bash
Environment="OPENCLAW_API_KEY=<your-token>"
```
...
```

---

## 版本历史

| 日期 | 版本 | 变更内容 |
|------|------|----------|
| 2025-03-30 | 1.0 | 初始版本，记录 Gateway Token 配置问题 |
