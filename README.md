# GitHub Actions Windows Cloud PC (MSTSC + Cloudflare Tunnel)

这个仓库提供一个 GitHub Actions 工作流：启动 `windows-latest` Runner，先自动恢复最近一次备份，再开启远程桌面（RDP），通过 `cloudflared` 建立 Cloudflare Tunnel，并在最后 1 小时执行“加密分卷 ZIP 备份”。

## 1. 配置 Secrets
在仓库 `Settings -> Secrets and variables -> Actions` 里添加：

- `CLOUDFLARE_TUNNEL_TOKEN`: 你的 Cloudflare Tunnel token
- `CLOUDFLARE_RDP_HOSTNAME`: 绑定到该 Tunnel 的 RDP 主机名，例如 `rdp.example.com`
- `RDP_PASSWORD`: 远程桌面密码基值（用于用户 `runneradmin`）
- `ZIP_PASSWORD`: 备份 zip 加密密码

`RDP_PASSWORD` 如果本身不满足 Windows 密码复杂度，工作流会自动在末尾追加固定后缀 `-Aa1!`，日志里会提示你实际应该用哪种方式连接。

## 2. Cloudflare 侧前置配置
先在 Cloudflare Zero Trust 里准备一个远程管理 Tunnel：

- 创建一个 Tunnel，拿到 `Tunnel token`
- 给这个 Tunnel 配一个 Public Hostname，例如 `rdp.example.com`
- Hostname 对应的服务填 `rdp://localhost:3389`
- 建议再为该 Hostname 配置一个 Access 应用

## 3. 启动工作流
1. 打开 `Actions`。
2. 选择 `Windows Cloud PC (Cloudflare Tunnel + Encrypted Backup)`。
3. 点击 `Run workflow`。
4. 默认总时长 `300` 分钟（5 小时），默认备份窗口 `60` 分钟（最后 1 小时）。

## 4. 客户端连接方式
工作流日志中会打印：

- Cloudflare hostname
- User: `runneradmin`
- Password mode
- `cloudflared access rdp` 命令

在你的 Windows 客户端上：

1. 安装 `cloudflared`
2. 运行日志里给出的命令，例如：
   `cloudflared access rdp --hostname rdp.example.com --url rdp://localhost:13389`
3. 保持这个命令窗口不要关闭
4. 再运行：
   `mstsc /v:localhost:13389`

## 5. 运行与保存策略
- 启动时：自动查找并恢复最近一次成功上传的备份；第一次运行如果没有备份，会直接跳过恢复。
- 前 4 小时：RDP 在线可连接（默认）。
- 最后 1 小时：自动执行备份，把文件打包为 **ZIP + AES256 密码加密 + 分卷**。
- 备份源默认包含：
  - `C:\Users\runneradmin\Desktop`
  - `C:\Users\runneradmin\Documents`
  - `C:\Users\runneradmin\Downloads`
  - `C:\Users\runneradmin\Pictures`
  - `C:\Users\runneradmin\Videos`
  - `D:\CloudPC`
- 恢复时会把这些路径解压回原位置。
- 备份通过 `actions/upload-artifact` 上传到当前 workflow 的 Artifacts（默认保留 30 天）。

## 6. 使用建议
- 大文件和需要跨轮次保留的数据，优先放到 `D:\CloudPC`。
- 常用桌面文件放 `Desktop`、文档放 `Documents`，它们都会进入备份。

## 7. “全天候”说明（重要）
GitHub Hosted Runner 不是永久主机，单次作业有时长上限，结束后环境会销毁。

本仓库已加：
- 每 5 小时自动触发（`schedule`）
- 手动触发（`workflow_dispatch`）

这可以做“近似连续在线”，但不保证真正 24/7 零中断。若你需要稳定 24/7，请改成自托管 Windows 机器作为 `self-hosted` Runner。

## 8. 安全建议
- 不要复用常用密码。
- 不使用时停用工作流或撤销 Tunnel token。
- 建议为 RDP Hostname 配置 Cloudflare Access 策略，而不是裸露给公网。
