# Gemtek XG2010G (PonWrt) 配置与运维说明

本分支（`xg2010g`）基于上游 `pbs05/ponwrt`（Linux 6.18），针对 **Gemtek XG2010G** 进行了启动稳定性加固、硬件端口修复及面向国内运营商（如 10G-EPON / XG-PON PPPoE 拨号）的开箱优化。

---

## 1. 分支相对上游 `pbs05/ponwrt` 的核心改动

1. **无串口环境启动安全加固**（`target/linux/airoha/an7581/config-6.18`）
   - 关闭 `CONFIG_CPU_FREQ` / `CONFIG_ARM_AIROHA_SOC_CPUFREQ` / `CONFIG_AIROHA_CPU_PM_DOMAIN`
   - 关闭 `CONFIG_MTD_UBI_FASTMAP`（避免 Linux 6.18 与 U-Boot fastmap 不兼容）
   - 关闭 `CONFIG_RANDOMIZE_BASE`（无硬件 TRNG `kaslr-seed` 时规避 KASLR 重定位问题）
2. **XG2010G 硬件与网络栈补丁**
   - DTS：修正 `&gdm1` MAC 偏移为 `<&lan_mac 4>`、`&gsw_port4`（`lan4`）为 `<&lan_mac 3>`；两颗 RTL8261N 万兆 PHY 启用 `reset-before-id-read;`
   - AN7581 PCS / USXGMII：合入速率自适应（`625`）、SDK 交叉搜索 RX 校准（`628`）及可选 TX FIR 覆盖（`629`）
   - PPE 硬件卸载：合入 PPPoE 8 字节会话头出口 MTU 计算修复（`920-17`，防大包丢包）与网桥 FDB 漫游流表自动失效（`990-01`）
   - 用户态修复：合入 `ppp 2.5.4` 上游 PR #638 修复（`501`）、`luci-app-upnp` nft 计数器解析修复（`100`），并过滤 `distfeeds.list` 中的本地编译 feed 防止 `apk update` 404
3. **开箱默认项**
   - 默认主机名：`XG2010G`
   - 默认语言 / 时区：`zh_cn`（简体中文）、`Asia/Shanghai (CST-8)`
   - 默认 WAN 接口：出厂即为 `pppoe`（`ipv6='1'`、`keepalive='5 5'`）+ `wan6`（`device='@wan'`、`reqaddress='try'`、`reqprefix='auto'`），改 VLAN 时 `wan6` 自动跟随无需重配，且不会触发双 `odhcp6c`
   - 默认开启防火墙软件与 PPE 硬件流量卸载（`flow_offloading='1'`、`flow_offloading_hw='1'`）
   - 内置 `luci-app-pon`、`airoha-pon-debug`、AN7581 EIP93 硬件加密（`kmod-crypto-hw-eip93` + `cryptodev`）、FullCone NAT、WireGuard、CAKE、BBR 及常用网络诊断工具（已剔除冗余 IPTV 组播代理组件）

---

## 2. PPPoE 拨号 + VLAN + IPv6 标准配置（必读）

PON 数据口固定为 `pon0`（**切勿手动改动或桥接控制口 `omci0` / `oam0`**）。

### 2.1 创建运营商宽带 VLAN（以 VLAN `3961` 为例）
```sh
uci set network.pon0_3961='device'
uci set network.pon0_3961.type='8021q'
uci set network.pon0_3961.ifname='pon0'
uci set network.pon0_3961.vid='3961'
uci set network.pon0_3961.name='pon0.3961'
```
> 若局端通过 OMCI/OAM 自动剥离了 VLAN（Untagged），则无需创建 `8021q` 设备，保持 `wan.device='pon0'` 即可。可用 `pppoe-discovery -I pon0.3961` 验证能否收到 PADO。

### 2.2 填写宽带账号与绑定 VLAN（出厂已预置 `wan` + `@wan` 的 `wan6`）
新固件首次启动已由 `config_generate` 自动生成标准 PPPoE 双栈结构（`wan.ipv6='1'` + `wan6.device='@wan'`），**无需改动 `wan6`**，只需修改 `wan` 的设备与账号密码：

```sh
uci set network.wan.device='pon0.3961'
uci set network.wan.username='宽带账号'
uci set network.wan.password='宽带密码'
uci commit network
/etc/init.d/network reload
```
> **注意**：不要把 `wan.ipv6` 改成 `'auto'` 同时又保留静态 `wan6`（`device='@wan'`），否则 `netifd` 派生的 `wan_6` 会与静态 `wan6` 启动两个 `odhcp6c`。若改用纯桥接模式，直接停用或删除 `wan` / `wan6` 即可。

---

## 3. EPON / 10G-EPON 与 GPON / XG(S)-PON 状态排查区分

通过 `ponctl status` 排查线路时，不同制式关注的字段不同：

| 制式 | 控制口 | 关键同步与注册指标 | 说明 |
| --- | --- | --- | --- |
| **GPON / XG-PON / XGS-PON** | `omci0` | `xgtc_sync`、`onu_state=O5`、`onu_id_valid=1`、`service_ready=1` | 走 ITU-T OMCI 协议栈 |
| **EPON / 10G-EPON** (`epon-1g` / `epon-10g-1g` / `epon-10g-10g`) | `oam0` | `pcs_sync`、`mpcp_state`、`llid_valid=1`、`service_ready=1`、`data_path_configured=1` | `onu_state` 与 `xgtc_sync` 显示 `not-applicable` 属正常现象 |

### 国内联通 / 电信 EPON 的 OAM 模式（`operator`）说明
`/etc/config/pon` 中 `pon.line0_oam.operator` 支持两个值：
- `'ieee'`：标准 IEEE 802.3ah OAM（上游默认）。
- `'ctc'`：IEEE 802.3ah + 中国电信/国内运营商 CTC 扩展 OAM（带 `ctc_oui='111111'`，支持 LOID/密码或纯 MAC 认证下的扩展属性协商）。
- **排障建议**：在北京联通等纯 MAC 认证 10G-EPON 场景下，首先修改 `pon0` MAC 为入网 MAC；若插纤后 `llid_valid=1` 但无法完成 OAM 业务下发或 `pppoe-discovery` 无响应，可将 `pon.line0_oam.operator` 切换为 `'ctc'`（无 LOID 时将 `loid` / `loid_password` 留空）后重启 `airoha-pond`。

---

## 4. 关于 EN7572 `calibration: missing` / `empty BOB table` 告警说明

切换到 `epon-10g-1g` 或 `epon-10g-10g` 时，内核日志与 `ponctl status` 会出现：
```text
airoha-en7572 0-0050: preparing epon-10g-1g-a0 with an empty BOB table using stock-compatible register codes
```
- **原因**：美版 Gemtek XG2010G 出厂仅在 Flash 校准区写入了面向 XG(S)-PON 的 **BOB A2 表**，面向 10G-EPON 的 **BOB A0 表**出厂即为全 `0xFF` 空表。
- **安全性与兼容性**：`kmod-airoha-en7572` 已严格按照原厂 SDK `AdaptivePon()` 行为回退到兼容默认寄存器码，突发发光门控（BEN）仍由 PON MAC 硬件严格接管，不会长发光或损坏模块。
- **上线核验建议**：插上光纤完成注册后，执行 `ponctl status` 检查 `[frontend]` 下的 `rx_power_dbm`（通常 `-8 ~ -24 dBm`）、`tx_power_dbm`（10G-EPON 通常 `+2 ~ +6 dBm`）及 `tx_bias_ma` 处于合理区间即可。

---

## 5. 上线验收命令速查

```sh
# 1. 查看光模块收发光功率与 PON 注册状态
ponctl status

# 2. 检查拨号 IPv4 与自动生成的 IPv6 (wan_6)
ifstatus wan
ifstatus wan_6

# 3. 验证 PPPoE 1492 MTU + PPE 硬件卸载下 1464 字节不分片大包不丢包
ping -c 4 -M do -s 1464 223.5.5.5
```
