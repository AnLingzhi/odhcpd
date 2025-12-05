I will implement the **LAN Priority Logic** in `src/ndp.c` to prevent reflection issues where `odhcpd` incorrectly learns downstream neighbors on the upstream (Master) interface.

### Plan
1.  **Modify `src/ndp.c`**:
    *   Locate the `ndp_netevent_cb` function, specifically the `NETEV_NEIGH6_ADD` case.
    *   Insert a new logic block that executes when a neighbor is added on a **Master** interface (`iface->master`).
    *   Iterate through all configured interfaces using `avl_for_each_element`.
    *   For each interface, check if it is a **Slave** interface (`!c->master`) and has NDP enabled (or just has addresses).
    *   Iterate through the IPv6 addresses (`addr6`) assigned to the Slave interface.
    *   Use `odhcpd_bmemcmp` to check if the new neighbor's IP matches the prefix of any Slave interface address.
    *   If a match is found, **return early** from the function to ignore the event, preventing route updates and logging.

### Proposed Code Changes (`src/ndp.c`)

```c
static void ndp_netevent_cb(unsigned long event, struct netevent_handler_info *info)
{
	struct interface *iface = info->iface;
	bool add = true;

	if (!iface || iface->ndp == MODE_DISABLED)
		return;

	switch (event) {
	case NETEV_ADDR6_DEL:
		add = false;
		netlink_dump_neigh_table(false);
		_o_fallthrough;
	case NETEV_ADDR6_ADD:
		setup_addr_for_relaying(&info->addr.in6, iface, add);
		break;
	case NETEV_NEIGH6_DEL:
		add = false;
		_o_fallthrough;
	case NETEV_NEIGH6_ADD:
		if (info->neigh.flags & NTF_PROXY) {
			if (add) {
				netlink_setup_proxy_neigh(&info->neigh.dst.in6, iface->ifindex, false);
				setup_route(&info->neigh.dst.in6, iface, false);
				netlink_dump_neigh_table(false);
			}
			break;
		}

		/* LAN Priority Logic: Ignore neighbors on Master interface if they belong to a Slave subnet */
		if (add && iface->master) {
			struct interface *c;
			avl_for_each_element(&interfaces, c, avl) {
				if (!c->master && c->ndp != MODE_DISABLED) {
					size_t i;
					for (i = 0; i < c->addr6_len; ++i) {
						struct odhcpd_ipaddr *addr = &c->addr6[i];
						if (addr->prefix_len > 128) continue;
						
						if (!odhcpd_bmemcmp(&addr->addr.in6, &info->neigh.dst.in6, addr->prefix_len)) {
							/* Found match on slave interface, ignoring reflection */
							return;
						}
					}
				}
			}
		}

		if (add &&
		    !(info->neigh.state &
		      (NUD_REACHABLE|NUD_STALE|NUD_DELAY|NUD_PROBE|NUD_PERMANENT|NUD_NOARP)))
			break;

		setup_addr_for_relaying(&info->neigh.dst.in6, iface, add);
		setup_route(&info->neigh.dst.in6, iface, add);

		if (!add)
			netlink_dump_neigh_table(false);
		break;
	default:
		break;
	}
}
```
