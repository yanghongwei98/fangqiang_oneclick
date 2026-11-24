# AI 安装手册

适用于在用户指定的 Linux VPS 上安装、验证或排查 `fangqiang_oneclick`。文中的下载和校验命令固定使用 [v2026.11.24.1](https://github.com/yanghongwei98/fangqiang_oneclick/releases/tag/v2026.11.24.1)。

## 1. 确认目标和版本

- 确认操作的是用户指定的 SSH 会话，以及该主机的公网 IP 或域名。安装会创建或更新本项目的 systemd 服务与监听端口。
- 支持 Debian 11/12/13 和 Ubuntu 22.04/24.04，架构为 amd64 或 arm64，需运行 systemd 并具备 root 或 sudo 权限。Ubuntu 26.10 会在系统检查阶段被拒绝，请和用户确认兼容的目标环境。
- 轻量版提供 10 个直连节点，订阅服务需要系统 Python 3。完整版提供 10 个直连节点和 10 个 WARP 节点，内置备用 Python 3。
- 已有部署时，检查服务归属与现有配置。安装器会复用本项目已有的密钥和订阅。卸载或删除旧配置属于单独操作。

```bash
cat /etc/os-release
uname -m
test -d /run/systemd/system && systemctl is-system-running
df -h /var/tmp /opt
systemctl show -p LoadState -p FragmentPath sing-box.service sbp-sub.service warp-svc.service
```

## 2. 下载安装包并校验

在有网络的机器上下载，按需求选择其中一个 `.sh`。可以把包和校验文件复制到离线 VPS 上再安装。

```bash
base=https://github.com/yanghongwei98/fangqiang_oneclick/releases/download/v2026.11.24.1
package=sing-box-plus-offline-warp-universal.sh
# 轻量版改用 package=sing-box-plus-offline-lite-universal.sh

mkdir -p fangqiang-install
cd fangqiang-install
curl -fL --retry 2 -o "$package" "$base/$package"
curl -fL --retry 2 -o SHA256SUMS "$base/SHA256SUMS"
awk -v name="$package" '$2 == name {print}' SHA256SUMS | sha256sum --check -
TMPDIR=/var/tmp bash "$package" --verify
```

两次校验都成功后再执行。包内校验会解压多架构文件。如果 `/tmp` 在小容量内存盘上，把 `TMPDIR` 指向空间充足的磁盘目录。上例采用 `/var/tmp`，先用 `df -h` 确认它所在的文件系统有空间。

## 3. 离线安装并验证服务

把 `SERVER_IP_OR_DOMAIN` 换成目标 VPS 的实际公网地址。在隔离环境试装时可以填写该环境的回环地址 `127.0.0.1`。

```bash
sudo env TMPDIR=/var/tmp bash "$package" --check
sudo env TMPDIR=/var/tmp bash "$package" --address SERVER_IP_OR_DOMAIN

sudo /opt/sing-box-offline/bin/sing-box check -c /opt/sing-box/config.json
systemctl is-active sing-box.service
systemctl is-active sbp-sub.service
sudo test -x /usr/local/bin/singbox-offline
```

预期 `sing-box` 为 `active`，并监听对应的 10 或 20 个节点。有可用的 Python 3 时，订阅服务也应为 `active`。试装若要求实证断网，使用虚拟机等隔离网络环境，在切断外网后执行第 3 步，保留访问生产主机所用的网卡和 SSH 会话。

## 4. WARP 联网激活

此步骤仅适用于完整版。激活前请由用户确认 [Cloudflare WARP 使用条款](https://www.cloudflare.com/application/terms/)。服务器需要能连接 Cloudflare。

```bash
sudo singbox-offline --activate-warp
```

交互提示输入 `yes` 后，程序会配置 MASQUE 和本地 SOCKS5 代理，并用 HTTPS 检查出站。自动化只有在用户已明确同意本次激活时才使用 `--accept-warp-tos`。

```bash
systemctl is-active warp-svc.service
sudo /opt/sing-box-offline/bin/warp-cli --accept-tos status
sudo ss -lnt 'sport = :40000'
sudo env SSL_CERT_FILE=/opt/sing-box-offline/warp/ca-certificates.crt \
  /opt/sing-box-offline/bin/python3 /opt/sing-box-offline/warp/verify-egress.py
```

成功结果包括 `Connected`、监听 `127.0.0.1:40000`，以及 `WARP_PROXY_OK warp=on` 或 `warp=plus`。如果测试实际节点链路，通过客户端分别访问 Cloudflare 的 `/cdn-cgi/trace`，对比直连的 `warp=off` 与 WARP 节点的 `warp=on` 或 `warp=plus`。端口在监听、WARP 已出站是两项独立结果。

## 5. 获取订阅与排错

- 在用户自己的终端运行 `sudo singbox-offline`，选 `2` 查看订阅。安装输出、订阅链接和管理菜单可能包含口令、访问令牌或节点凭据，转述结果时仅报告状态和经过脱敏的错误。
- 如果旧版 `v2026.11.24` 已安装，但 `singbox-offline` 报 `Permission denied`，可以用 `sudo bash /usr/local/bin/singbox-offline` 打开菜单，或用 `sudo bash /usr/local/bin/singbox-offline --activate-warp` 继续激活。本补丁版本已修复打包时的执行权限。
- 如果服务未启动，用 `systemctl status` 和 `journalctl -u 服务名 -n 60` 定位出错阶段。报告时分别列出下载校验、离线安装、监听和 WARP 出站的通过情况。
