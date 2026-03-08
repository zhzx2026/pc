# GitHub Actions Windows Cloud PC (MSTSC + Encrypted Backup)

这个仓库提供一个 GitHub Actions 工作流：启动 `windows-latest` Runner，开启远程桌面（RDP），通过 `ngrok tcp` 暴露公网地址，并在最后 1 小时执行“加密分卷 ZIP 备份”。

## 1. 配置 Secrets
在仓库 `Settings -> Secrets and variables -> Actions` 里添加：

- `NGROK_AUTHTOKEN`: 你的 ngrok token
- `RDP_PASSWORD`: 远程桌面密码（用于用户 `runneradmin`）
- `ZIP_PASSWORD`: 备份 zip 加密密码

> 密码建议至少 12 位，包含大小写、数字和符号。

## 2. 启动工作流
1. 打开 `Actions`。
2. 选择 `Windows Cloud PC (RDP + Encrypted Backup)`。
3. 点击 `Run workflow`。
4. 默认总时长 `300` 分钟（5 小时），默认备份窗口 `60` 分钟（最后 1 小时）。

## 3. 获取连接信息
工作流日志中会打印：

- Host
- Port
- User: `runneradmin`
- 示例命令：`mstsc /v:host:port`

## 4. 运行与保存策略
- 前 4 小时：RDP 在线可连接（默认）。
- 最后 1 小时：自动执行备份，把文件打包为 **ZIP + AES256 密码加密 + 分卷**。
- 备份源默认包含：
  - `C:\Users\runneradmin`
  - `D:\a`
- 备份通过 `actions/upload-artifact` 上传到当前 workflow 的 Artifacts（默认保留 30 天）。

## 5. “全天候”说明（重要）
GitHub Hosted Runner 不是永久主机，单次作业有时长上限，结束后环境会销毁。

本仓库已加：
- 每 5 小时自动触发（`schedule`）
- 手动触发（`workflow_dispatch`）

这可以做“近似连续在线”，但不保证真正 24/7 零中断。若你需要稳定 24/7，请改成自托管 Windows 机器作为 `self-hosted` Runner。

## 6. 安全建议
- 不要复用常用密码。
- 不使用时停用工作流或删除 ngrok token。
- 建议在 ngrok 控制台限制来源 IP（如可用）。
