# OpenCode 远程 MCP 服务器配置指南

本文档介绍如何通过远程 MCP 服务器扩展 OpenCode 的功能，实现远程代码执行、文件操作等能力。

## 准备工作

- 一台运行 Linux 的远程服务器（需要能访问公网）
- 本地已安装 OpenCode

## 远程服务器配置

### 1. 安装 Node.js

```bash
# Ubuntu/Debian
curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
apt install -y nodejs

# 验证安装
node -v
npm -v
```

### 2. 安装并启动 MCP 服务器

使用官方 `server-everything` 作为示例，它提供丰富的工具集：

```bash
# 启动 MCP 服务（前台运行测试）
PORT=8765 npx -y @modelcontextprotocol/server-everything streamableHttp
```

看到以下输出表示启动成功：

```
Starting Streamable HTTP server...
MCP Streamable HTTP Server listening on port 8765
```

### 3. 后台持久化运行

```bash
# 后台运行
PORT=8765 nohup npx -y @modelcontextprotocol/server-everything streamableHttp > mcp.log 2>&1 &

# 查看日志
tail -f mcp.log
```

### 4. 开放防火墙端口（如需要）

```bash
# Ubuntu
ufw allow 8765/tcp
ufw reload

# CentOS
firewall-cmd --permanent --add-port=8765/tcp
firewall-cmd --reload
```

### 5. 开机自启（可选）

创建 systemd 服务 `/etc/systemd/system/mcp.service`：

```ini
[Unit]
Description=MCP Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/root
Environment=PORT=8765
ExecStart=/usr/bin/npx -y @modelcontextprotocol/server-everything streamableHttp
Restart=always

[Install]
WantedBy=multi-user.target
```

启动服务：

```bash
systemctl daemon-reload
systemctl enable mcp
systemctl start mcp
```

## 本地 OpenCode 配置

### 配置文件位置

- Windows: `%USERPROFILE%\.config\opencode\opencode.json`
- macOS/Linux: `~/.config/opencode/opencode.json`

### 添加 MCP 配置

编辑配置文件，添加以下内容：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "openai": {
      "options": {
        "baseURL": "https://your-api-endpoint.com/openai"
      }
    }
  },
  "mcp": {
    "remote-everything": {
      "type": "remote",
      "url": "http://你的服务器IP:8765/mcp",
      "enabled": true
    }
  }
}
```

**注意**：将 `你的服务器IP` 替换为实际服务器地址（如 `152.42.213.41`）

### 配置多个 MCP 服务器

```json
{
  "mcp": {
    "remote-everything": {
      "type": "remote",
      "url": "http://152.42.213.41:8765/mcp",
      "enabled": true
    },
    "another-mcp": {
      "type": "remote",
      "url": "http://another-server:8765/mcp",
      "enabled": true
    }
  }
}
```

### 配置认证（可选）

如果 MCP 服务器需要认证：

```json
{
  "mcp": {
    "my-mcp": {
      "type": "remote",
      "url": "http://152.42.213.41:8765/mcp",
      "enabled": true,
      "headers": {
        "Authorization": "Bearer YOUR_API_KEY"
      }
    }
  }
}
```

## 使用方法

1. **重启 OpenCode** 使配置生效

2. **在对话中使用 MCP 工具**，例如：
   - `使用 remote-everything 工具列出当前目录文件`
   - `用 remote-everything 执行 Python 代码 print("Hello")`
   - `通过 remote-everything 读取 /etc/hostname 文件`

3. **查看可用工具**
   
   MCP 服务器连接成功后，会自动加载其提供的工具。你可以直接询问：
   - `remote-everything 有哪些工具？`
   - `列出 remote-everything 可用的功能`

## 常用 MCP 服务器推荐

| 服务器 | 用途 | 启动命令 |
|--------|------|----------|
| server-everything | 综合工具集 | `PORT=8765 npx -y @modelcontextprotocol/server-everything streamableHttp` |
| server-filesystem | 文件系统操作 | `PORT=8765 npx -y @modelcontextprotocol/server-filesystem /path/to/dir streamableHttp` |
| server-github | GitHub 操作 | 需要配置 GITHUB_TOKEN 环境变量 |

## 故障排查

### 连接失败

1. 检查远程服务器 MCP 是否正在运行：
   ```bash
   tail -f mcp.log
   ```

2. 检查端口是否开放：
   ```bash
   # 服务器上
   netstat -tlnp | grep 8765
   
   # 本地测试
   curl http://你的服务器IP:8765/mcp
   ```

3. 检查防火墙设置

### 工具加载失败

1. 重启 OpenCode
2. 检查 MCP 配置 JSON 格式是否正确
3. 查看 OpenCode 日志中的错误信息

## 进阶配置

### 使用 code-executor-mcp 管理多个 MCP

```bash
# 安装
npm install -g code-executor-mcp

# 交互式配置
code-executor-mcp setup

# 启动
code-executor-mcp run --port 8765
```

### Docker 部署

```bash
docker run -d \
  --name mcp-server \
  -p 8765:8765 \
  -e PORT=8765 \
  npx -y @modelcontextprotocol/server-everything streamableHttp
```

## 注意事项

1. **安全风险**：远程 MCP 服务器可执行代码，请仅在可信网络中使用
2. **网络延迟**：远程调用会有网络延迟，实时性要求高的场景建议使用本地 MCP
3. **Token 消耗**：MCP 工具会增加上下文 token 消耗，注意 API 配额
4. **防火墙**：生产环境建议配置防火墙规则，限制访问来源

## 相关链接

- [OpenCode 官方文档](https://opencode.ai/docs/)
- [MCP 官方文档](https://modelcontextprotocol.io/)
- [modelcontextprotocol/servers](https://github.com/modelcontextprotocol/servers)
