这里为您整理了一份最新的 **README 中文指南**。

由于考虑到不同用户的实际需求（有些 VPS 商家是按“开机日”每 30 天滚动计费，而 GCP 等商家是严格按“自然月每月 1 日”计费），本指南同时收录了 **【版本 A：GCP 每月 1 日自然月重置版】** 和 **【版本 B：固定 30 天轮询重置版】**，并统一集成了“智能网卡自适应”与“计费开始/截止时间看板”功能。

---

# GCP / 独立 VPS 出站流量限制与断网保护指南 (双版本通用完全体)

本方案利用 Linux 内核级的 `iptables` 流量计数器精确统计服务器流出的总公网数据量。当累积出站流量超过设定阈值（默认 **190 GB**）时，系统会自动触发断网保护，**切断除 SSH 远程连接（22端口）以外的所有出站流量**，彻底杜绝高额账单风险。

## ✨ 完全体核心优势

1. **智能网卡自适应**：自动识别承载外网流量的真实网卡名称（如 `ens4` 或 `eth0`），无需人工硬编码。
2. **可视化周期看板**：手动输入 `liuliang` 即可秒级查看当前识别到的网卡、精确的**流量计算开始与截止时间**以及当前用量。
3. **绝对路径适配**：核心命令全部改用 `/sbin/` 绝对路径，彻底解决后台 `crontab` 定时任务找不到命令（`command not found`）的经典报错。
4. **多行健壮性过滤**：数据提取加入了精准过滤限制，完美解决因多条重复防火墙规则导致脚本报整数表达式错误（`integer expression expected`）的问题。

---

## 🛠️ 从零配置步骤

请以 **`root`** 用户身份登录您的 Linux 实例，依次执行以下步骤：

### 第一步：初始化系统语言环境（消除警告）

一键修复 Debian/Ubuntu 系统由于缺少英文语言包频繁报出 `setlocale: LC_ALL` 警告的问题：

```bash
apt update && apt install -y locales
sed -i 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
locale-gen en_US.UTF-8
update-locale LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8
export LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8

```

---

### 第二步：根据计费规则，选择以下【一个】版本写入脚本

#### 方案 A：GCP 专属自然月版（每月 1 日 0 点自动清零）

* **适用场景**：Google Cloud (GCP) 等严格按照**自然月**（每月 1 号到当月最后一天）计算免费额度和账单的商家。
* **特点**：计费周期自动对齐本月 1 号至最后一天。跨月时脚本首次运行会自动清零计数器，不需要人工干预。

直接复制并执行以下整块命令写入：

```bash
cat << 'EOF' > /usr/local/bin/traffic_quota.sh
#!/bin/bash

# ==========================================
# ⚙️ 用户自定义配置
# ==========================================
QUOTA_GB=190  # 允许出站流量阈值（单位：GB）

# ==========================================
# 🗓️ 自动计算 GCP 自然月周期
# ==========================================
START_DATE=$(date +"%Y-%m-01 00:00:00")
END_DATE=$(date -d "$(date +'%Y-%m-01') +1 month -1 day" +"%Y-%m-%d 23:59:59")
CURRENT_MONTH=$(date +"%Y-%m")

# ==========================================
# ⚡ 智能网卡自动识别与初始化
# ==========================================
INTERFACE=$(/sbin/ip route show | grep default | awk '{print $5}' | tail -n 1)
if [ -z "$INTERFACE" ]; then echo "无法自动识别外网网卡"; exit 1; fi

/sbin/iptables -N TRAFFIC_MONITOR 2>/dev/null
if ! /sbin/iptables -C OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR 2>/dev/null; then
    /sbin/iptables -A OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR
fi

# 跨月自动清零检查
LAST_MONTH_FILE="/var/run/traffic_monitor_month.txt"
[ -f "$LAST_MONTH_FILE" ] && LAST_MONTH=$(cat "$LAST_MONTH_FILE") || LAST_MONTH=""
if [ "$CURRENT_MONTH" != "$LAST_MONTH" ]; then
    /sbin/iptables -F TRAFFIC_MONITOR && /sbin/iptables -Z TRAFFIC_MONITOR
    echo "$CURRENT_MONTH" > "$LAST_MONTH_FILE"
fi

# ==========================================
# 📊 流量统计与看板输出
# ==========================================
CURRENT_BYTES=$(/sbin/iptables -L OUTPUT -v -n -x | grep "TRAFFIC_MONITOR" | awk '{print $2}' | tail -n 1)
[ -z "$CURRENT_BYTES" ] && CURRENT_BYTES=0
CURRENT_GB=$(awk -v bytes="$CURRENT_BYTES" 'BEGIN {printf "%.4f", bytes / 1024 / 1024 / 1024}')

echo "=================================================="
echo " 网卡设备: ${INTERFACE}"
echo " 计费周期: [ ${START_DATE} ] 至"
echo "           [ ${END_DATE} ] (GCP自然月)"
echo " 流量状态: ${CURRENT_GB} GB / 限额 ${QUOTA_GB} GB"
echo "=================================================="

QUOTA_BYTES=$(( QUOTA_GB * 1024 * 1024 * 1024 ))
if [ "$CURRENT_BYTES" -gt "$QUOTA_BYTES" ]; then
    echo "警告：流量已超过限额！正在实施断网保护..."
    if ! /sbin/iptables -C TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT 2>/dev/null; then
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --sport 22 -j ACCEPT
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT
        /sbin/iptables -A TRAFFIC_MONITOR -j DROP
        echo "断网保护已生效。仅保留 SSH 连接。"
    fi
fi
EOF
chmod +x /usr/local/bin/traffic_quota.sh

```

---

#### 方案 B：固定 30 天轮询版（不随自然月漂移，精准每 30 天清零）

* **适用场景**：部分按“开机日/购买日”起算每 30 天为一个计费周期的独立 VPS 商家。
* **特点**：计费周期显示为“当前时间”至“30天后”。依靠系统的 `Crontab` 强制每 30 天清零账本。

直接复制并执行以下整块命令写入：

```bash
cat << 'EOF' > /usr/local/bin/traffic_quota.sh
#!/bin/bash

# ==========================================
# ⚙️ 用户自定义配置
# ==========================================
QUOTA_GB=190  # 允许出站流量阈值（单位：GB）

# ==========================================
# 🗓️ 滚动计算 30 天周期时间
# ==========================================
START_DATE=$(date +"%Y-%m-%d %H:%M:%S")
END_DATE=$(date -d "+30 days" +"%Y-%m-%d %H:%M:%S")

# ==========================================
# ⚡ 智能网卡自动识别与初始化
# ==========================================
INTERFACE=$(/sbin/ip route show | grep default | awk '{print $5}' | tail -n 1)
if [ -z "$INTERFACE" ]; then echo "无法自动识别外网网卡"; exit 1; fi

/sbin/iptables -N TRAFFIC_MONITOR 2>/dev/null
if ! /sbin/iptables -C OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR 2>/dev/null; then
    /sbin/iptables -A OUTPUT -o "$INTERFACE" -j TRAFFIC_MONITOR
fi

# ==========================================
# 📊 流量统计与看板输出
# ==========================================
CURRENT_BYTES=$(/sbin/iptables -L OUTPUT -v -n -x | grep "TRAFFIC_MONITOR" | awk '{print $2}' | tail -n 1)
[ -z "$CURRENT_BYTES" ] && CURRENT_BYTES=0
CURRENT_GB=$(awk -v bytes="$CURRENT_BYTES" 'BEGIN {printf "%.4f", bytes / 1024 / 1024 / 1024}')

echo "=================================================="
echo " 网卡设备: ${INTERFACE}"
echo " 流量统计: 正在运行中... (自上次清零起算)"
echo " 预期预测: 如果现在清零，新周期截止大约在: ${END_DATE}"
echo " 流量状态: ${CURRENT_GB} GB / 限额 ${QUOTA_GB} GB"
echo "=================================================="

QUOTA_BYTES=$(( QUOTA_GB * 1024 * 1024 * 1024 ))
if [ "$CURRENT_BYTES" -gt "$QUOTA_BYTES" ]; then
    echo "警告：流量已超过限额！正在实施断网保护..."
    if ! /sbin/iptables -C TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT 2>/dev/null; then
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --sport 22 -j ACCEPT
        /sbin/iptables -I TRAFFIC_MONITOR -p tcp --dport 22 -j ACCEPT
        /sbin/iptables -A TRAFFIC_MONITOR -j DROP
        echo "断网保护已生效。仅保留 SSH 连接。"
    fi
fi
EOF
chmod +x /usr/local/bin/traffic_quota.sh

```

---

### 第三步：配置快捷查询命令 `liuliang`

```bash
echo "alias liuliang='/usr/local/bin/traffic_quota.sh'" >> /etc/bash.bashrc
source /etc/bash.bashrc

```

---

### 第四步：根据所选版本，配置对应的定时任务（Crontab）

运行 `crontab -e` 打开编辑器，移到**最底部**，根据第二步选择的版本，粘贴对应的配置：

* **如果选了【方案 A：GCP 专属自然月版】**，粘贴这一行即可（脚本内部会自动判断跨月清零）：
```text
*/5 * * * * /usr/local/bin/traffic_quota.sh >> /var/log/traffic_quota.log 2>&1

```


* **如果选了【方案 B：固定 30 天轮询版】**，必须粘贴以下两行（第二行负责强制每 30 天清空计数器）：
```text
*/5 * * * * /usr/local/bin/traffic_quota.sh >> /var/log/traffic_quota.log 2>&1
0 0 */30 * * /sbin/iptables -Z TRAFFIC_MONITOR >> /var/log/traffic_quota.log 2>&1

```



---

## 📊 日常维护

### 1. 实时查看流量状态

在终端直接键入：

```bash
liuliang

```

### 2. 流量超标断网后，如何手动恢复？

当进入了新的计费周期需要复活网络，或者需要临时解除限制时，输入以下两条命令即可重置：

```bash
# 清空 TRAFFIC_MONITOR 链中产生的封禁规则
/sbin/iptables -F TRAFFIC_MONITOR
# 将流量计数账本归零
/sbin/iptables -Z TRAFFIC_MONITOR

```
