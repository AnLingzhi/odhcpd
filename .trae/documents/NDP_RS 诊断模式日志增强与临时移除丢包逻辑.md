## 目标
- 在 NS 与 RS 入口无条件打印诊断日志，显示收到包的源 MAC 与本机接口 MAC，便于确认反射包的源 MAC 实际值。
- 暂时移除“自发包丢弃”行为，仅保留后续流程，避免因过滤判断错误导致看不到日志。

## 变更范围
- 文件：`src/ndp.c`（函数 `handle_solicit`），`src/router.c`（函数 `handle_icmpv6` 针对 `ND_ROUTER_SOLICIT`）。
- 使用现有日志宏（`notice` 或 `warn`），保证日志在系统中可见。

## 具体修改
### ndp.c:handle_solicit
- 保留既有有效性检查与目标地址校验。
- 获取接口 MAC：`odhcpd_get_mac(iface, mymac)`。
- 获取包源 MAC：优先 `sockaddr_ll` 的 `ll->sll_addr`；`ll->sll_halen == ETH_ALEN` 时格式化为字符串，否则标记为 `unknown`（可选保留之前从 `ND_OPT_SOURCE_LINKADDR` 解析的增强，不触发丢弃）。
- 无条件打印诊断日志：
  - 格式：`"DIAG_NS: Iface=%s, Pkt_SrcMAC=%s, MyMAC=%s"`。
  - 级别：`notice(...)`（或 `warn(...)`）。
- 移除之前的 `is_self_sent` 导致的早期 `return`，但保留 `is_self_sent` 变量以维持后续 NA 应答逻辑中 `!is_self_sent` 的条件（不破坏既有行为）。

示例补丁锚点（参考行号）：
- 在 `src/ndp.c:390` 之后插入：
```
char pkt_mac[18];
char my_mac_str[18];
uint8_t mymac[6];
odhcpd_get_mac(iface, mymac);
if (ll->sll_halen == ETH_ALEN)
    snprintf(pkt_mac, sizeof(pkt_mac), "%02x:%02x:%02x:%02x:%02x:%02x",
             ll->sll_addr[0], ll->sll_addr[1], ll->sll_addr[2], ll->sll_addr[3], ll->sll_addr[4], ll->sll_addr[5]);
else
    snprintf(pkt_mac, sizeof(pkt_mac), "unknown");
snprintf(my_mac_str, sizeof(my_mac_str), "%02x:%02x:%02x:%02x:%02x:%02x",
         mymac[0], mymac[1], mymac[2], mymac[3], mymac[4], mymac[5]);
notice("DIAG_NS: Iface=%s, Pkt_SrcMAC=%s, MyMAC=%s", iface->name, pkt_mac, my_mac_str);
```
- 删除或注释掉（实际修改为删除）此前的 `if (is_self_sent) return;`，以确保诊断模式不丢弃。
- 保留并计算 `is_self_sent`，用于后续 `send_na` 的判断，不改变后续逻辑（如 `!is_self_sent`）。

### router.c:handle_icmpv6（ND_ROUTER_SOLICIT）
- 在 `hdr->icmp6_type == ND_ROUTER_SOLICIT` 分支内：
  - 遍历选项（使用 `icmpv6_for_each_option`）查找 `ND_OPT_SOURCE_LINKADDR`；若存在，取 `opt->data` 作为源 MAC；否则记为 `unknown`。
  - 获取接口 MAC：`odhcpd_get_mac(iface, mymac)`，格式化字符串。
  - 无条件打印诊断日志：
    - 格式：`"DIAG_RS: Iface=%s, Pkt_SrcMAC=%s, MyMAC=%s"`。
    - 级别：`notice(...)`（或 `warn(...)`）。
- 不改变既有转发/应答逻辑，仅增加日志与移除任何早退丢弃（我们不会新增丢弃）。

示例补丁锚点（参考行号）：
- 在 `src/router.c:1149–1154` 区块进入 RS 分支后插入：
```
struct nd_router_solicit *rs = (struct nd_router_solicit *)data;
struct icmpv6_opt *opt; uint8_t *end = (uint8_t *)data + len; uint8_t *mac_ptr = NULL;
icmpv6_for_each_option(opt, &rs[1], end) {
    if (opt->type == ND_OPT_SOURCE_LINKADDR && opt->len >= 1) { mac_ptr = opt->data; break; }
}
char pkt_mac[18]; if (mac_ptr)
    snprintf(pkt_mac, sizeof(pkt_mac), "%02x:%02x:%02x:%02x:%02x:%02x",
             mac_ptr[0], mac_ptr[1], mac_ptr[2], mac_ptr[3], mac_ptr[4], mac_ptr[5]);
else
    snprintf(pkt_mac, sizeof(pkt_mac), "unknown");
uint8_t mymac[6]; char my_mac_str[18];
odhcpd_get_mac(iface, mymac);
snprintf(my_mac_str, sizeof(my_mac_str), "%02x:%02x:%02x:%02x:%02x:%02x",
         mymac[0], mymac[1], mymac[2], mymac[3], mymac[4], mymac[5]);
notice("DIAG_RS: Iface=%s, Pkt_SrcMAC=%s, MyMAC=%s", iface->name, pkt_mac, my_mac_str);
```

## 代码风格与兼容性
- 复用现有包含与宏，无新增依赖。
- 字符串缓冲区长度 `18` 足以容纳 `xx:xx:xx:xx:xx:xx` 与终止符。
- 不添加注释，遵循工程约定。

## 验证与回滚
- 确认日志在设备上出现，观察实际 `Pkt_SrcMAC` 值。
- 验证后再决定恢复或改进过滤逻辑（基于真实 MAC 值做精确匹配）。
- 本次只增加日志与移除早退丢弃，不更改既有转发与 NA 应答语义。