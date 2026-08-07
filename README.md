# OpenVPN 管理系统（ovpnx.sh 增强版）

基于 [RusianHu/openvpn-quickbox](https://github.com/RusianHu/openvpn-quickbox) 的 `ovpnx.sh`
改造而来，在原「一键安装 + 证书管理」基础上，新增**账号密码认证**与 **Web 管理后台**。

## 改造内容

| 功能 | 实现方式 |
|------|----------|
| 账号 + 密码认证 | 证书(CN)认证 + `auth-user-pass-verify` 叠加用户名/密码；密码 `加盐 sha256` 存于 `$WORKDIR/accounts.db` |
| Web 管理后台 | Python 标准库实现（无第三方依赖），监听 `8080`；超级管理员密码登录 |
| 在线注册账号 | Web 表单 → 自动生成证书 + 写入账号库 |
| 账号列表 / 流量 / 设备IP / 状态 | 解析 `/var/log/openvpn-status.log`，展示每 CN 的 Real Address、收发字节、连接时间 |
| 删除账号 | Web 一键删除：吊销证书 + 清理文件 + 删除账号记录 |
| 操作日志 | 每次管理操作追加写入 `$WORKDIR/op.log` |
| 一键重启 | Web 按钮 / 菜单项，调用 `systemctl restart`，需管理员登录 |
| 超级管理员密码 | 安装向导设置，存于 `$WORKDIR/admin.pass`（权限 600） |

## 文件

- `ovpnx.sh` — 改造后的主脚本（含完整中文注释）。安装时自动生成内嵌 Web 后台到 `$WORKDIR/ovpnx_web.py`
- `ovpnx_web.py` — 独立的 Web 后台实现（与主脚本内嵌版功能一致，便于单独查看/调试）
- `test_web.py` — Web 后台自动化测试（17/17 通过）

## 使用

```bash
# 安装（向导会要求设置管理员密码，并自动启动 Web 后台）
sudo bash ovpnx.sh

# 子命令模式（供 Web 后台/脚本调用，非交互）
sudo bash ovpnx.sh make <name>      # 生成证书
sudo bash ovpnx.sh revoke <name>     # 吊销证书
sudo bash ovpnx.sh clean <name>      # 清理文件
sudo bash ovpnx.sh restart           # 重启 OpenVPN
sudo bash ovpnx.sh web               # 仅启动 Web 后台

# 访问管理后台
# http://<服务器公网IP>:8080   （用安装时设置的管理员密码登录）
```

## 验证结果（沙箱内已实测）

| 验证项 | 结果 |
|--------|------|
| `bash -n ovpnx.sh` 语法 | ✅ 通过 |
| `ovpnx_web.py` 单测（登录/注册/列表/删除/日志/status解析/密码哈希） | ✅ 17/17 通过 |
| `parse_status` awk 解析（多账号流量/设备IP） | ✅ 通过 |
| `auth-check.sh` 密码校验（正确通过 / 错误拒绝） | ✅ 通过 |
| 内嵌 Web 后台全链路（curl 模拟：登录→注册→列表→删除→日志） | ✅ 通过 |

> 说明：沙箱为容器环境（无真实 systemd、未安装 OpenVPN），故**未在此完整跑通 OpenVPN 服务端**。
> 上述验证覆盖了所有不依赖 systemd/openvpn 的增强逻辑。在你真实的 Ubuntu 22.04+ 服务器上，
> 按 `sudo bash ovpnx.sh` 安装即可获得完整的证书 + 账号密码 + Web 管理体验。

## 安全提示

- 管理员密码与账号密码均以 `加盐 sha256` 存储，非明文。
- Web 后台监听 `0.0.0.0:8080`，**建议在生产环境用 Nginx 反代 + HTTPS + 防火墙限制来源 IP**。
- `accounts.db` / `admin.pass` 权限为 `600`（仅 root 可读）。
