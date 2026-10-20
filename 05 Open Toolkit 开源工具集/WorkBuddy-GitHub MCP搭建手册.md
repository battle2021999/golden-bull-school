---
status: 可用
updated: 2026-10-20
---

# WorkBuddy-GitHub MCP 搭建手册

> 目标：让 WorkBuddy 通过 MCP 直接读写 GitHub 仓库（建文件、改文件、发 PR），不用来回切浏览器。
> 本手册的配置与排错表均经实机验证（Windows，2026-10）。

## 一、准备

- GitHub 账号 + 目标仓库（建议用细粒度 PAT，只授权单个仓库）
- Node.js 18+（保证 `npx` 可用）
- WorkBuddy 桌面端

## 二、创建令牌

1. GitHub → Settings → Developer settings → Personal access tokens → **Fine-grained tokens** → Generate new token
2. Repository access：Only select repositories → 选中目标仓库
3. Permissions → Repository permissions → **Contents: Read and write**（Metadata 自动为 Read-only）
4. 生成后立即复制，页面只显示一次

## 三、写配置

文件路径：`~/.workbuddy/mcp.json`（Windows：`C:\Users\<用户名>\.workbuddy\mcp.json`）

```json
{
  "mcpServers": {
    "golden-bull-github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "github_pat_xxx"
      }
    }
  }
}
```

## 四、关键坑：变量名必须是 GITHUB_PERSONAL_ACCESS_TOKEN

官方包 `@modelcontextprotocol/server-github` **只读 `GITHUB_PERSONAL_ACCESS_TOKEN`**。写成常见的 `GITHUB_TOKEN` 不会报配置错误，但请求根本不会带 Authorization 头。

症状很有欺骗性：

- 读取正常——因为公开仓库可以匿名读
- 写入报 `401 Authentication Failed: Requires authentication`
- 反复重启、重装、换令牌都无效

源码位置：`node_modules/@modelcontextprotocol/server-github/dist/common/utils.js`

```js
if (process.env.GITHUB_PERSONAL_ACCESS_TOKEN) {
  headers["Authorization"] = `Bearer ${process.env.GITHUB_PERSONAL_ACCESS_TOKEN}`;
}
```

**结论：给任何 MCP 配令牌前，先看包源码里读的是哪个变量名，不要凭习惯写。**

## 五、让配置生效

`env` 只在 MCP 进程启动时读取。改完 `mcp.json` 必须：

1. 完全退出 WorkBuddy（只关窗口不够）后重新打开；或
2. 在连接器管理页右上角的自定义连接器入口，对该服务重新点一次「信任」

## 六、连通性验收（三步缺一不可）

1. **读**：列出仓库根目录
2. **写**：新建一个测试 `.md`
3. **回读**：确认文件内容与提交一致

仅「读成功」不能证明令牌可用。

## 七、排错表

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 读正常、写报 401 Requires authentication | 用了 `GITHUB_TOKEN`，请求没带 Authorization | 改成 `GITHUB_PERSONAL_ACCESS_TOKEN` 并重启 |
| 读写都 404 | 令牌未授权该仓库，或 owner/repo 写错 | 检查令牌的 Repository access |
| 写报 403 Resource not accessible | Contents 权限是只读 | 改为 Read and write |
| 改完配置没反应 | 进程未重启，env 仍是旧的 | 完全退出客户端重开 |
| `npx` 长时间卡住 | 网络或 npm 源问题 | 检查代理，先手动跑一次 `npx` 预热缓存 |

## 八、安全要求

- 令牌只授单仓库、最小权限，能设过期就设
- `mcp.json` 含明文令牌，**不要提交进任何仓库**
- 怀疑泄露立即 revoke 并重新生成
- 团队协作时每人用各自令牌，不共用

## 九、常用能力速查

| 操作 | 工具 |
| --- | --- |
| 列出目录 / 读文件 | `get_file_contents` |
| 新建或覆盖单个文件 | `create_or_update_file`（更新时需传 sha） |
| 批量推多个文件（一个 commit） | `push_files` |
| 建 Issue / PR | `create_issue`、`create_pull_request` |
| 查提交历史 | `list_commits` |
