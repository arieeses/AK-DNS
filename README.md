# AK-DNS

AKDNS 一键测速与系统 DNS 接管脚本（基于 [akile-network/aktools](https://github.com/akile-network/aktools) 的 `akdns.sh` 定制增强）。

自动测速全部 AKDNS 节点，选出**最快的前 2 个节点**并在其后追加 **`1.1.1.1`** 作为兜底，统一写入 `/etc/resolv.conf`；支持**定期自动测速**（systemd timer / cron），保持解锁 DNS 始终指向最优节点。

> An enhanced fork of AKDNS: speed-tests all nodes, applies the **top-2 fastest** plus **`1.1.1.1`** as a fallback, and can **re-test on a schedule** (systemd timer / cron).

---

## 一键安装 / One-liner

```bash
bash <(curl -fsSL https://raw.githubusercontent.com/arieeses/AK-DNS/main/akdns.sh)
```

> 若无 `curl`，可用 `wget`：
>
> ```bash
> bash <(wget -qO- https://raw.githubusercontent.com/arieeses/AK-DNS/main/akdns.sh)
> ```

运行后进入交互菜单。接管系统 DNS 需要 root，请按提示使用 `sudo` 或以 root 运行。

---

## 主要特性

- **前 2 名 + 兜底**：测速排序后取延迟最低的 2 个 AKDNS 节点，第 3 行写入 `1.1.1.1`，正好符合 glibc `MAXNS=3` 上限。
- **定期自动测速**：可安装 systemd timer（优先）或 cron 定时任务，按 `hourly` / `daily` / `weekly` / `Nh` 间隔自动重测并接管。测速全部失败时保持现有 DNS 不变，绝不断网。
- **统一接管**：不论原后端是 systemd-resolved / NetworkManager / netplan / resolvconf，一律收敛为静态 `/etc/resolv.conf`，可选 `chattr +i` 锁定防覆盖。
- **完整备份 / 还原**：接管前自动备份，菜单一键回滚。
- **UDP→TCP 回退**：UDP/53 被封锁时自动改用 TCP（写入 `options use-vc`）。
- **中英双语界面**。

## 命令行用法

```bash
# 交互菜单
sudo bash akdns.sh

# 非交互式测速并接管（供定时任务调用，也可手动跑一次）
sudo bash akdns.sh --auto

# 安装定期自动测速（间隔: hourly / daily / weekly / Nh，默认 daily）
sudo bash akdns.sh --install-cron=daily
sudo bash akdns.sh --install-cron=12h

# 卸载定时任务
sudo bash akdns.sh --uninstall-cron

# 其他
bash akdns.sh --help
bash akdns.sh --version
bash akdns.sh --lang en      # 强制英文界面
```

## 可调参数

编辑脚本顶部：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `TOP_DNS_COUNT` | `2` | 取测速排名前几名的 AKDNS 节点 |
| `FALLBACK_DNS` | `1.1.1.1` | 追加的公共兜底 DNS（可空格分隔多个） |
| `AKDNS_DEFAULT_INTERVAL` | `daily` | 定时任务默认间隔 |
| `LOCK_RESOLV_CONF` | `true` | 是否对 `/etc/resolv.conf` 加 immutable 锁 |

## 定时任务日志

- **systemd**：`journalctl -u akdns-auto`
- **cron**：`/var/log/akdns-auto.log`

## 还原

菜单选 `7) 还原 DNS 配置`，或删除定时任务后手动还原备份（位于 `/var/lib/akdns/backup`）。

---

## 免责声明

本项目为个人定制增强版，DNS 节点与解锁能力归属 [Akile](https://dns.akile.ai)。请在了解「接管系统 DNS」影响的前提下使用，生产环境建议先在测试机验证。
