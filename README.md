# AK-DNS

AKDNS 一键测速与系统 DNS 接管脚本（基于 [akile-network/aktools](https://github.com/akile-network/aktools) 的 `akdns.sh` 定制增强）。

自动测速全部 AKDNS 节点，选出**最快的前 2 个节点**并在其后追加 **`1.1.1.1`** 作为兜底，按延迟从低到高写入 `/etc/resolv.conf`；支持**定期自动测速**（systemd timer / cron），保持解锁 DNS 始终指向最优节点。

> An enhanced fork of AKDNS: speed-tests all nodes, applies the **top-2 fastest** plus **`1.1.1.1`** as a fallback (ordered by latency), and can **re-test on a schedule** (systemd timer / cron).

> ⚠️ 接管系统 DNS 需要 **root**。若不是 root，先执行 `sudo -i` 切到 root 再运行。

---

## 快速开始 / Quick Start

> 以下命令均以 root 运行。`RAW` 地址为 `https://raw.githubusercontent.com/arieeses/AK-DNS/main/akdns.sh`。

### 1. 交互菜单（下载后运行，最稳）

```bash
curl -fsSL https://raw.githubusercontent.com/arieeses/AK-DNS/main/akdns.sh -o akdns.sh && bash akdns.sh
```

装过一次后 `/usr/local/bin/akdns` 已存在，之后直接运行 `akdns` 即可打开菜单。

### 2. 一键安装定期自动测速（**并立即接管一次**）

```bash
curl -fsSL https://raw.githubusercontent.com/arieeses/AK-DNS/main/akdns.sh | bash -s -- --install-cron=daily
```

安装定时任务的同时会**立刻测速并写入 DNS**，不用等到定时器首次触发。间隔可换成 `hourly` / `weekly` / `6h` / `12h`。

### 3. 手动立即测速并接管一次

```bash
akdns --auto
```

### 4. 查看结果 / 状态

```bash
cat /etc/resolv.conf                      # 看当前写入的 DNS（前 2 快 + 1.1.1.1）
systemctl list-timers akdns-auto.timer    # 看定时器下次触发时间（systemd）
journalctl -u akdns-auto --no-pager -n 30 # 看定时任务日志（systemd）
```

### 5. 卸载定时任务

```bash
akdns --uninstall-cron
```

---

## 主要特性

- **前 2 名 + 兜底**：测速排序后取延迟最低的 2 个 AKDNS 节点，第 3 行写入 `1.1.1.1`，正好符合 glibc `MAXNS=3` 上限。写入顺序 = 延迟从低到高，最快的作主用。
- **定期自动测速**：安装 systemd timer（优先）或 cron 定时任务，按 `hourly` / `daily` / `weekly` / `Nh` 间隔自动重测重排并接管。每次都重新排名，最快节点变化时自动提到第一行。
- **不断网保护**：测速全部超时时保持现有 DNS 不变；当前 DNS 已是最优时跳过写入。
- **统一接管**：不论原后端是 systemd-resolved / NetworkManager / netplan / resolvconf，一律收敛为静态 `/etc/resolv.conf`，可选 `chattr +i` 锁定防覆盖。
- **完整备份 / 还原**：接管前自动备份，菜单一键回滚。
- **UDP→TCP 回退**：UDP/53 被封锁时自动改用 TCP（写入 `options use-vc`）。
- **中英双语界面**。

## 命令行参数

| 命令 | 说明 |
| --- | --- |
| `akdns` | 打开交互菜单 |
| `akdns --auto` | 非交互式测速并接管一次（定时任务调用的就是它） |
| `akdns --install-cron[=INTERVAL]` | 安装定期自动测速并立即执行一次；间隔 `hourly`/`daily`/`weekly`/`Nh`，默认 `daily` |
| `akdns --uninstall-cron` | 卸载定时任务 |
| `akdns --lang <zh\|en>` | 强制界面语言 |
| `akdns --help` | 查看帮助 |
| `akdns --version` | 查看版本 |

> 首次通过管道 / 下载运行时，脚本会自动把自身复制到 `/usr/local/bin/akdns`，之后可直接用 `akdns` 调用。

### 交互菜单项

```
1) 流媒体解锁检测        2) 测速                3) 应用为系统 DNS
4) 一键测速并接管        5) 临时应用            6) 备份当前配置
7) 还原 DNS 配置         8) 状态                9) 定期自动测速
L) 语言                  0) 退出
```

## 可调参数

编辑脚本顶部（`/usr/local/bin/akdns` 或本地 `akdns.sh`）：

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `TOP_DNS_COUNT` | `2` | 取测速排名前几名的 AKDNS 节点 |
| `FALLBACK_DNS` | `1.1.1.1` | 追加的公共兜底 DNS（可空格分隔多个，如 `"1.1.1.1 8.8.8.8"`） |
| `COUNT` | `2` | 每个节点测速次数（取平均） |
| `TIMEOUT` | `1` | 单次 dig 超时秒数 |
| `AKDNS_DEFAULT_INTERVAL` | `daily` | 定时任务默认间隔 |
| `LOCK_RESOLV_CONF` | `true` | 是否对 `/etc/resolv.conf` 加 immutable 锁 |

> 注意：系统只认 `/etc/resolv.conf` 前 3 个 `nameserver`，`TOP_DNS_COUNT` + 兜底数量之和建议 ≤ 3。

## 定时任务日志

- **systemd**：`journalctl -u akdns-auto`
- **cron**：`/var/log/akdns-auto.log`

## 还原 DNS

- 菜单选 `7) 还原 DNS 配置` 一键回滚；
- 或先 `akdns --uninstall-cron` 卸载定时任务，再从备份目录 `/var/lib/akdns/backup` 手动还原。

## 常见问题

**Q：装完 `--install-cron` 后 `/etc/resolv.conf` 没变？**
早期版本只装定时器、不立即执行；现已改为安装后立即接管一次。若仍未变，多半是测速节点超时——脚本只写入实测**可达**的节点，全部超时则保持原样。跑 `akdns --auto` 看输出即可判断。

**Q：为什么不能用 `sudo bash <(curl ...)`？**
进程替换产生的 `/dev/fd/63` 在 `sudo` 起的新进程里不可见，会报 `/dev/fd/63: No such file or directory`。请以 root（或 `sudo -i` 后）直接用本文档给出的写法。

**Q：没有 `curl`？**
把 `curl -fsSL <url>` 换成 `wget -qO- <url>`，把 `curl -fsSL <url> -o akdns.sh` 换成 `wget -O akdns.sh <url>`。

---

## 免责声明

本项目为个人定制增强版，DNS 节点与解锁能力归属 [Akile](https://dns.akile.ai)。请在了解「接管系统 DNS」影响的前提下使用，生产环境建议先在测试机验证。
