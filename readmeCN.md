# GCP Debian 实例出站流量限制与断网保护指南 (智能自适应完全体)

本指南提供了一套在 **Debian**（及其他主流 Linux）系统下运行的 VPS 出站流量监控与断网保护方案。

由于 Google Cloud (GCP) 的免费流量政策极其严格且超额费用高昂，本方案利用 Linux 内核级的 `iptables` 流量计数器精确统计流出的总数据量。当 30 天内的累积出站流量超过设定阈值（默认 **190 GB**）时，系统会自动触发断网保护，**切断除 SSH 远程连接以外的所有出站流量**，彻底杜绝信用卡被爆刷的风险。

## 🌟 完全体核心优势

1. **智能网卡自适应**：脚本利用 `ip route` 动态识别当前承载外网流量的真实网卡名称（如 `ens4` 或 `eth0`），**无需人工硬编码，绝不会选错网卡**。
2. **绝对路径适配**：核心命令全部改用 `/sbin/iptables` 绝对路径，彻底解决了 Debian 系统下后台 `crontab` 定时任务找不到命令（`command not found`）的经典报错。
3. **多行健壮性过滤**：数据提取部分加入了精确定位与 `tail -n 1` 限制，完美解决了因多条重复规则导致脚本报整数表达式错误（`integer expression expected`）的问题。
4. **极简查询别名**：在终端任何目录下输入 `liuliang` 即可秒级查看当前识别到的活动网卡以及精确到小数点后 4 位的已用流量。

---

## 🛠️ 从零配置步骤

请以 **`root`** 用户身份登录您的 GCP Linux 实例，依次执行以下四个步骤：

### 第一步：初始化系统语言环境（消除警告）

为了防止 Debian 系统频繁报出烦人的 `setlocale: LC_ALL` 警告，先一键修复系统语言包：

```bash
apt update && apt install -y locales
sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
export LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

```

### 第二步：一键写入智能监控脚本

直接复制并执行以下整块命令，脚本将自动写入到全局可执行路径中，具有网卡自动识别和规则自动初始化功能：

```bash
cat << 'EOF' > /usr/local/bin/traffic_quota.sh
#!/bin/bash

# ==========================================
# 配置参数
# ==========================================
QUOTA_GB=190  # 最大允许出站流量阈值（单位：GB）

# ==========================================
# ⚡ 智能网卡自动识别
# ==========================================
# 自动获取当前连接外网的活动网卡名称
INTERFACE=$(/sbin/ip route show | grep default | awk '{print $5}' | tail -n 1)

if [ -z "$INTERFACE" ]; then
    echo "无法自动识别外网网卡，请检查网络连接。"
    exit 1
fi

# ==========================================
# 自动动态初始化防火墙计数链
# ==========================================
/sbin/iptables -N TRAFFIC_MONITOR 2>/dev/null
if ! /sbin/iptables -C OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR 2>/dev/null; then
    /sbin/iptables -A OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR
fi

# ==========================================
# 核心流量统计与判断
# ==========================================
# 精准提取 TRAFFIC_MONITOR 计数器的字节数，确保只取最后一行纯数字
CURRENT_BYTES=$(/sbin/iptables -L OUTPUT -v -n -x | grep "TRAFFIC_MONITOR" | awk '{print $2}' | tail -n 1)

if [ -z "$CURRENT_BYTES" ]; then
    echo "未能成功读取流量计数器。"
    exit 1
fi

# 计算当前已用 GB 数（保留4位小数）
CURRENT_GB=$(awk -v bytes="$CURRENT_BYTES" 'BEGIN {printf "%.4f", bytes / 1024 / 1024 / 1024}')

# 打印当前日志
echo "网卡: ${INTERFACE} | 当前已用出站流量: ${CURRENT_GB} GB / 限额: ${QUOTA_GB} GB"

# 使用 Bash 内置安全整数运算计算阈值字节数
QUOTA_BYTES=$(( QUOTA_GB * 1024 * 1024 * 1024 ))

# 核心断网判断逻辑
if [ "$CURRENT_BYTES" -gt "$QUOTA_BYTES" ]; then
    echo "警告：流量已超过 ${QUOTA_GB}GB！正在实施断网保护..."
    
    # 检查是否已经实施过保护，避免重复添加规则
    if ! /sbin/iptables -C TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT 2>/dev/null; then
        # 1. 紧急放行 SSH 22 端口（允许出站和入站，保障管理员绝对不断连）
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --sport 22 -j ACCEPT
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT
        
        # 2. 拒绝其他所有通过活动网卡流出的外部网络流量
        /sbin/iptables -A TRAFFIC_MONITOR -j DROP
        echo "断网保护已生效。仅保留 SSH 连接。"
    fi
fi
EOF

# 赋予脚本可执行权限
chmod +x /usr/local/bin/traffic_quota.sh

```

### 第三步：配置快捷命令别名 `liuliang`

将快捷指令写入系统全局 Bash 配置文件中并刷新，从此查流量只需打 6 个字母：

```bash
echo "alias liuliang='/usr/local/bin/traffic_quota.sh'" >> /etc/bash.bashrc
source /etc/bash.bashrc

```

### 第四步：部署后台 Cron 定时任务

1. 打开计划任务编辑器：
```bash
crontab -e

```


2. 将以下两行标准配置粘贴至文件的**最底部**（注意：必须独立成行，确保结尾没有多余符号）：
```text
# 每 5 分钟自动检测一次出站流量是否超标
*/5 * * * * /usr/local/bin/traffic_quota.sh >> /var/log/traffic_quota.log 2>&1

# 每 30 天的午夜 0 点，自动重置（清零）流量计数器开始新计费周期
0 0 */30 * * /sbin/iptables -Z TRAFFIC_MONITOR >> /var/log/traffic_quota.log 2>&1

```


3. 保存并退出。当系统提示 `crontab: installing new crontab` 时，说明全自动保护罩已正式开启。

---

## 📊 日常管理指令

### 1. 实时查看流量消耗

在终端任意路径下，直接输入：

```bash
liuliang

```

**输出示例：**

> 网卡: ens4 | 当前已用出站流量: 0.0695 GB / 限额: 190 GB

### 2. 查看后台定时任务运行日志

检查脚本在后台是否有每 5 分钟准时“打卡”记录：

```bash
tail -n 10 /var/log/traffic_quota.log

```

### 3. 流量超标断网后，如何手动恢复网络？

当进入了新的计费周期，或者您需要临时扩容解除断网限制时，直接执行以下两条命令，**清空断网规则并重置计数器**，网络将瞬间全开恢复：

```bash
# 1. 清空 TRAFFIC_MONITOR 链中产生的所有 DROP 封禁规则
/sbin/iptables -F TRAFFIC_MONITOR

# 2. 将流量计数器归零
/sbin/iptables -Z TRAFFIC_MONITOR

```
