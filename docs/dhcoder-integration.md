# OpenChamber + DHCoder 改造与打包指南

> 将 OpenCode 二次封装为 `dhcoder` 后，让 OpenChamber 桌面端（Electron）正确识别并启动 dhcoder 服务的改造记录。

---

## 1. 改造背景

- 原始命令行工具：`opencode`
- 二次封装后的命令行工具：`dhcoder`
- 兼容方式：通过软链把 `opencode` 链接到 `dhcoder`，使 OpenChamber 仍然调用 `opencode` 命令时实际执行 `dhcoder`

```bash
# 示例：把 dhcoder 链接为 opencode（确保 /usr/local/bin/opencode 实际指向 dhcoder）
ln -sf /path/to/dhcoder /usr/local/bin/opencode
```

OpenChamber 启动服务时大致流程：

1. spawn `opencode serve --hostname 127.0.0.1 --port <随机端口>`
2. 监听子进程 stdout，等待服务就绪日志
3. 拿到 URL 后请求 `GET /global/health`，验证 `{healthy: true}`

---

## 2. 遇到的问题

### 2.1 服务启动日志前缀不匹配

OpenChamber 原代码等待的日志格式：

```text
opencode server listening on http://127.0.0.1:<port>
```

而 `dhcoder` 实际输出：

```text
dhcoder server listening on http://127.0.0.1:<port>
```

因为前缀不一致，OpenChamber 在 30 秒超时后报：

```text
Timeout waiting for OpenCode to start after 30000ms
```

### 2.2 密码环境变量名差异（可选）

OpenChamber 传给子进程的环境变量是：

```text
OPENCODE_SERVER_PASSWORD
```

而 dhcoder 可能读取的是：

```text
DHCODER_SERVER_PASSWORD
```

如果 dhcoder 不兼容 `OPENCODE_SERVER_PASSWORD`，启动时会提示未授权模式：

```text
Warning: DHCODER_SERVER_PASSWORD is not set; server is unsecured.
```

当前 OpenChamber 的健康检查在无密码模式下可以通过，因此不影响基本使用。后续如需启用密码保护，需要让 dhcoder 兼容 `OPENCODE_SERVER_PASSWORD`，或在 OpenChamber 中同时注入 `DHCODER_SERVER_PASSWORD`。

---

## 3. 代码改造

### 3.1 修改服务就绪日志匹配逻辑

需要放宽匹配条件，从 `startsWith('opencode server listening')` 改为 `includes('server listening')`，以兼容任意前缀。

#### 文件 1：VS Code 扩展

```text
packages/vscode/src/opencode.ts:646
```

修改前：

```typescript
if (!line.startsWith('opencode server listening')) continue;
```

修改后：

```typescript
if (!line.includes('server listening')) continue;
```

#### 文件 2：Web / Desktop 服务端运行时

```text
packages/web/server/lib/opencode/lifecycle.js:304
```

修改前：

```javascript
if (!line.startsWith('opencode server listening')) continue;
```

修改后：

```javascript
if (!line.includes('server listening')) continue;
```

#### 文件 3：单元测试

```text
packages/web/server/lib/opencode/lifecycle.test.js
```

将测试中模拟的 stdout 输出从：

```javascript
child.stdout.emit('data', 'opencode server listening on http://127.0.0.1:45678\n');
```

改为：

```javascript
child.stdout.emit('data', 'dhcoder server listening on http://127.0.0.1:45678\n');
```

涉及行号（以当前版本为准）：113、136、159、227。

---

## 4. 打包命令

### 4.1 前置条件

- 安装 Bun（项目要求 `bun@1.3.14`，当前可用 `1.3.13`）
- 安装 Node.js >= 22
- 安装项目依赖：

```bash
bun install
```

> 如果在中国大陆网络遇到 `IntegrityCheckFailed` 错误，可尝试切换 registry：
>
> ```bash
> bun install --registry https://registry.npmjs.org
> ```

### 4.2 macOS 打包（DMG）

#### 未签名 DMG（测试/内部使用）

```bash
bun run electron:build -- --mac dmg
```

产物路径：

```text
packages/electron/dist/OpenChamber-1.13.8-mac-arm64.dmg
```

`.app` 包路径：

```text
packages/electron/dist/mac-arm64/OpenChamber.app
```

#### 直接运行 .app 测试（可查看终端日志）

```bash
/Applications/OpenChamber.app/Contents/MacOS/OpenChamber
```

或从 dmg 挂载目录运行：

```bash
/Volumes/OpenChamber/OpenChamber.app/Contents/MacOS/OpenChamber
```

#### 实时查看系统日志

```bash
log stream --predicate 'process == "OpenChamber"' --info --debug
```

### 4.3 Windows 打包

#### 未签名 EXE/NSIS 安装包

```bash
bun run electron:build -- --win nsis
```

产物路径（示例）：

```text
packages/electron/dist/OpenChamber Setup 1.13.8.exe
```

#### 仅打便携版（portable）

```bash
bun run electron:build -- --win portable
```

产物路径（示例）：

```text
packages/electron/dist/OpenChamber 1.13.8.exe
```

#### Windows 注意事项

- 项目 `package.json` 中 `packageManager` 字段为 `bun`，但 Windows 上 Bun 的 workspace 安装可能不稳定。如果遇到问题，可尝试：
  - 使用 WSL2 环境下的 Linux 构建
  - 或在 Windows 上安装 Bun for Windows 后执行 `bun install`
- 原生模块（`better-sqlite3`、`node-pty`、`bun-pty`）会在构建时自动 rebuild
- 未签名安装包在 Windows 上会触发 SmartScreen，需要用户点击"更多信息" → "仍要运行"

### 4.4 同时打包多平台

```bash
# macOS + Windows
bun run electron:build -- --mac dmg --win nsis

# 全部平台（mac / win / linux）
bun run electron:build
```

---

## 5. 验证 dhcoder 是否被正确调用

### 5.1 检查软链

```bash
which opencode
ls -la $(which opencode)
```

应指向 dhcoder：

```text
/usr/local/bin/opencode -> /path/to/dhcoder
```

### 5.2 手动测试 dhcoder 启动输出

```bash
/usr/local/bin/opencode serve --hostname 127.0.0.1 --port 4096
```

期望输出（30 秒内）：

```text
server listening on http://127.0.0.1:4096
```

或带 dhcoder 前缀：

```text
dhcoder server listening on http://127.0.0.1:4096
```

### 5.3 检查健康检查端点

```bash
curl http://127.0.0.1:4096/global/health
```

期望返回：

```json
{"healthy": true}
```

---

## 6. 故障排查

### 问题 1：`Timeout waiting for OpenCode to start`

**原因**：dhcoder 的 stdout 没有输出 OpenChamber 能识别的 `server listening on ...` 格式。

**解决**：确保 dhcoder 启动成功后打印包含 `server listening on http://...` 的行。

### 问题 2：`Server started but health check failed`

**原因**：dhcoder 没有实现 `GET /global/health` 端点，或返回字段不是 `{healthy: true}`。

**解决**：在 dhcoder 中实现该端点并返回正确格式。

### 问题 3：健康检查 401 Unauthorized

**原因**：dhcoder 只认 `DHCODER_SERVER_PASSWORD`，而 OpenChamber 传的是 `OPENCODE_SERVER_PASSWORD`。

**解决（二选一）**：

- 让 dhcoder 兼容 `OPENCODE_SERVER_PASSWORD` 环境变量
- 修改 OpenChamber，在 spawn dhcoder 时同时注入 `DHCODER_SERVER_PASSWORD`

---

## 7. 相关文件速查

| 文件 | 作用 |
|---|---|
| `packages/vscode/src/opencode.ts` | VS Code 扩展启动 opencode/dhcoder |
| `packages/web/server/lib/opencode/lifecycle.js` | Web / Desktop 端启动 opencode/dhcoder |
| `packages/web/server/lib/opencode/lifecycle.test.js` | 启动逻辑单元测试 |
| `packages/electron/package.json` | Electron 打包配置、`build` 字段 |
| `packages/electron/scripts/build-web-assets.mjs` | 构建 web UI 资源 |
| `packages/electron/scripts/rebuild-native.mjs` | 重建原生模块 |
| `packages/electron/scripts/package.mjs` | 调用 electron-builder 打包 |

---

## 8. 版本信息

- OpenChamber: `1.13.8`
- Electron: `41.9.2`
- electron-builder: `26.15.3`
- Bun: `1.3.13`（项目声明 `1.3.14`）
- Node.js: `>= 22`

---

## 9. 快速命令备忘

```bash
# 安装依赖
bun install

# macOS DMG
bun run electron:build -- --mac dmg

# Windows NSIS
bun run electron:build -- --win nsis

# 多平台
bun run electron:build -- --mac dmg --win nsis

# 终端启动 macOS App 查看日志
/Applications/OpenChamber.app/Contents/MacOS/OpenChamber

# macOS 实时日志
log stream --predicate 'process == "OpenChamber"' --info --debug
```
