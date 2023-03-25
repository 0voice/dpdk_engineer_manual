# dpdk hw-offload flows i

```
This solution tries to hw-offload flows in the flow classification cache. On success the NIC does the classification and hints the packets with a flow id, so that it can direct the packet directly to output target port, and update the flow cache statistics, either retrieved from NIC or sw-handled. If the hw-offload fails normal OVS SW handling applies.
The flow cache contains match and action structures that is used to generate a flow classification offload request to device, using the flow specification and the wildcard bitmask from the match structure (megaflow). This is done on add, modify and delete of flow cache elements in the dpif-netdev userspace module.
The actual flow offload request to the device is done by adding two netdev_class functions "hw_flow_offload" and "get_flow_stats".
In netdev-dpdk, these functions are then implemented using the DPDK RTE FLOW API.
```

![img](https://img2020.cnblogs.com/blog/665372/202105/665372-20210508104613308-1503180077.png)



```
How to enable hw offload:
Using OVSDB and add hw-port-id to interface options:

# ovs-vsctl set interface dpdk0 options:hw-port-id=0


Implementation details (hopefully not too messy - diff can be retried from github: https://github.com/napatech/ovs/commit/467f076835143a9d0c17ea514d4e5c0c33d72c98.diff):

1) OVSDB option hw-port-id is used to enable hw-offload and maps the port interface to the hw port id. If a dpdk<X> port, where the X does not match the hw-port-id, a in-port remap is requested using RTE FLOW ITEM PORT. This enables for virtual ports in NIC. This is needed to do south, east and west flow classification in NIC. Note: not supported on all NICs supporting RTE FLOW.

dpif-netdev.c
=============

static odp_port_t hw_local_port_map[MAX_HW_PORT_CNT];

This array keeps a mapping between all created ports in dpif-netdev which has associated a hw-port-id, and the dpif selected odp port id. This is used to map port id used in flows to hw-port-id used in NIC.

in do_add_port():

if (netdev_is_pmd(port->netdev)) {
    struct smap netdev_args;

    smap_init(&netdev_args);
    netdev_get_config(port->netdev, &netdev_args);
    port->hw_port_id = smap_get_int(&netdev_args, "hw-port-id", (int)(MAX_HW_PORT_CNT + 1));
    smap_destroy(&netdev_args);

    VLOG_DBG("ADD PORT (%s) has hw-port-id %i, odp_port_id %i\n", devname, port->hw_port_id, port_no);

    if (port->hw_port_id <= MAX_HW_PORT_CNT) {
        hw_local_port_map[port->hw_port_id] = port_no;
        /* Inform back to netdev driver the actual selected ODP port number */
        smap_init(&netdev_args);
        smap_add_format(&netdev_args, "odp_port_no", "%lu", (long unsigned int)port_no);
        netdev_set_config(port->netdev, &netdev_args, NULL);
        smap_destroy(&netdev_args);
    }
} else {
    port->hw_port_id = (int)(MAX_HW_PORT_CNT + 1); }

In do_add_port, the netdev_get_config() is called to retreive a hw-port-id if specified. When specified, the odp-port-id is then send back to netdev port.


static void
dp_netdev_try_hw_offload(struct dp_netdev_port *port, struct match *match,
						  const ovs_u128 *ufid, const struct nlattr *actions,
						  size_t actions_len)
{
	if (port->hw_port_id < MAX_HW_PORT_CNT) {
        VLOG_INFO("FLOW ADD (try match) -> Actions  in_port  %i (port name : %s, type : %s)\n", match->flow.in_port.ofp_port,
                   xstrdup(netdev_get_name(port->netdev)), xstrdup(port->type));
        netdev_try_hw_flow_offload(port->netdev, port->hw_port_id, match, ufid, actions, actions_len);
    }
}

This function is called to add a new flow to hw or modify an existing one. The netdev_hw_offload module holds a cache of all ufid's 
of flows that is offloaded successfully to hw.

if dp_netdev_pmd_remove_flow:

netdev_try_hw_flow_offload(port->netdev, -1, NULL, &flow->ufid, NULL, 0);

Is called to remove a hw-offloaded flow. The netdev_hw_offload module will delete the flow in hw if ufid found in cache.


In dp_netdev_process_rxq_port:

	if (port->hw_port_id < MAX_HW_PORT_CNT) {
		struct netdev_flow_stats *flow_stats = NULL;
		int i;
		time_t a = time_now();
		if (port->sec < a) {
	        if (!ovs_mutex_trylock(&stat_mutex)) {
				//printf("CHECK FOR STATS %08x for port %i\n", (long long unsigned)pmd, port->hw_port_id);
                port->sec = a;

                netdev_hw_get_stats_from_dev(rx, &flow_stats);

                if (flow_stats) {
                    static struct dp_netdev_flow *flow;
                    long long now = time_msec();
                    for (i = 0; i < flow_stats->num; i++) {
                        if (flow_stats->flow_stat[i].packets) {
                            /* Some statistics to update on from this flow */
                            VLOG_DBG("Update stats with pkts %lu, bytes %lu, errors %lu, flow-id %lu\n",
                                    flow_stats->flow_stat[i].packets, flow_stats->flow_stat[i].bytes,
                                    flow_stats->flow_stat[i].errors, (long unsigned int)flow_stats->flow_stat[i].flow_id);

                            flow = dp_netdev_pmd_find_flow(pmd, &flow_stats->flow_stat[i].ufid, NULL, 0);
                            if (flow) {
                                dp_netdev_flow_used(flow, flow_stats->flow_stat[i].packets, flow_stats->flow_stat[i].bytes, 0, now);
                            }
                        }
                    }
                    netdev_hw_free_stats_from_dev(rx, &flow_stats);
                }
                ovs_mutex_unlock(&stat_mutex);
			}
		}
	}
```

```
Added this section to poll for hw statistics each second, then update flow cache statistics. 
The netdev_hw_get_stats_from_dev is a function in the netdev_hw_offload module and retreives statistics from hw, or if not supported,
read from flow cache.
NOTE: or'ed tcp_flags are not supported yet.

	#define GET_ODP_OUT_PORT(id) (id & FLOW_ID_ODP_PORT_BIT)?\
		id & FLOW_ID_PORT_MASK:((id & FLOW_ID_PORT_MASK) < FLOW_ID_PORT_MASK)?\
				hw_local_port_map[id & HW_PORT_MASK]:OVS_BE32_MAX;

	if (port->hw_port_id < MAX_HW_PORT_CNT) {
		int i, ii;
		struct dp_packet_batch direct_batch;
		odp_port_t direct_odp_port = OVS_BE32_MAX;

		dp_packet_batch_init(&direct_batch);

		i = 0;
		while (i < batch.count) {
			uint32_t flow_id = dp_packet_get_pre_classified_flow_id(batch.packets[i]);
			if (flow_id != (uint32_t)-1) {
				odp_port_t odp_out_port = GET_ODP_OUT_PORT(flow_id);

				if (odp_out_port < OVS_BE32_MAX) {
					if (direct_batch.count && direct_odp_port != odp_out_port) {
						/* Check if SW flow statistics update in hw-offload is needed - only if hw cannot give flow stats */
						netdev_update_flow_stats(rx, &direct_batch);
						_send_pre_classified_batch(pmd, direct_odp_port, &direct_batch);
						direct_batch.count = 0;
					}
					direct_odp_port = odp_out_port;
					direct_batch.packets[direct_batch.count++] = batch.packets[i];

					for (ii = i+1; ii < batch.count; ii++) {
						batch.packets[ii-1] = batch.packets[ii];
					}
					batch.count--;
					continue;
				}
			}
			i++;
		}

		if (direct_batch.count) {
			VLOG_DBG("Tx directly from Port (odp) %i to %i, num %i, left %i\n", port_no, direct_odp_port, direct_batch.count, batch.count);
			/* Check if SW flow statistics update in hw-offload is needed - only if hw cannot give flow stats */
			netdev_update_flow_stats(rx, &direct_batch);
			_send_pre_classified_batch(pmd, direct_odp_port, &direct_batch);
		}

		if (!batch.count) return;
	}

And adding this section to check for pre-classification id from NIC. If pre-classified, send it directly to destination. 
netdev_update_flow_stats is called to count statistics from batch, if NIC does not support flow statistics directly.

 

netdev-dpdk.c
=============

Need to include match.h to be able to read flow in hw_flow_offload function.

#include "openvswitch/match.h"

Add FDIR_MODE_PERFECT mode for Intel Flow Director configuration.

    .fdir_conf = {
        .mode = RTE_FDIR_MODE_PERFECT,
    },


The struct netdev_dpdk structure adds a few fields to keep track of hw-offload:

/* added to handle hw offloading of
    * packet classification.
    */
bool hw_offload;
int hw_port_id;
uint16_t odp_port_no;

if hw-port-id is configured in OVSDB, that id is reported to dpif-netdev add_port through netdev_dpdk_get_config and odp_port_no is
set back through netdev_dpdk_set_config.

In netdev_dpdk_set_config:

    int hw_port_id = smap_get_int(args, "hw-port-id", -1);
    uint32_t odp_port_no = smap_get_int(args, "odp_port_no", -1);

    ovs_mutex_lock(&dpdk_mutex);
    ovs_mutex_lock(&dev->mutex);

    if (hw_port_id != -1)
        dev->hw_port_id = hw_port_id;

    if ((odp_port_no < (uint32_t)-1)) {
            dev->odp_port_no = (uint16_t)odp_port_no;

        if (dev->hw_port_id != -1) {
            dev->hw_offload = true;
            VLOG_INFO("HW OFFLOAD ready on device (%s) : hw-port-id <-> odp-port-id  %i <-> %i\n", netdev->name, dev->hw_port_id, dev->odp_port_no);
        }
        ovs_mutex_unlock(&dev->mutex);
        ovs_mutex_unlock(&dpdk_mutex);
        return 0;
    }

read hw-port-id from OVSDB and odp_port_no from dpif-netdev and enable hw_offload.

In netdev_dpdk_get_config:

    /* If configured, report hw-port-id */
    if (dev->hw_port_id != (uint32_t)-1)
         smap_add_format(args, "hw-port-id", "%d", dev->hw_port_id);

Add hw-port-id, if configured, to read by dpif-netdev module.



Main flow match to DPDK RTE FLOW mapping function:

static int
netdev_dpdk_hw_flow_offload(struct netdev *netdev,
        const struct match *match, int hw_port_id,
        const struct nlattr *nl_actions, size_t actions_len,
        int *flow_stat_support, uint16_t flow_id,
        uint64_t *flow_handle)
{
    static const struct flow_item {
       const char *descr;
    } flow_item_desc[] = {
       [RTE_FLOW_ITEM_TYPE_END] = {.descr = "RTE_FLOW_ITEM_TYPE_END"},
       [RTE_FLOW_ITEM_TYPE_VOID] = {.descr = "RTE_FLOW_ITEM_TYPE_VOID"},
       [RTE_FLOW_ITEM_TYPE_INVERT] = {.descr = "RTE_FLOW_ITEM_TYPE_INVERT"},
       [RTE_FLOW_ITEM_TYPE_ANY] = {.descr = "RTE_FLOW_ITEM_TYPE_ANY"},
       [RTE_FLOW_ITEM_TYPE_PF] = {.descr = "RTE_FLOW_ITEM_TYPE_PF"},
       [RTE_FLOW_ITEM_TYPE_VF] = {.descr = "RTE_FLOW_ITEM_TYPE_VF"},
       [RTE_FLOW_ITEM_TYPE_PORT] = {.descr = "RTE_FLOW_ITEM_TYPE_PORT"},
       [RTE_FLOW_ITEM_TYPE_RAW] = {.descr = "RTE_FLOW_ITEM_TYPE_RAW"},
       [RTE_FLOW_ITEM_TYPE_ETH] = {.descr = "RTE_FLOW_ITEM_TYPE_ETH"},
       [RTE_FLOW_ITEM_TYPE_VLAN] = {.descr = "RTE_FLOW_ITEM_TYPE_VLAN"},
       [RTE_FLOW_ITEM_TYPE_IPV4] = {.descr = "RTE_FLOW_ITEM_TYPE_IPV4"},
       [RTE_FLOW_ITEM_TYPE_IPV6] = {.descr = "RTE_FLOW_ITEM_TYPE_IPV6"},
       [RTE_FLOW_ITEM_TYPE_ICMP] = {.descr = "RTE_FLOW_ITEM_TYPE_ICMP"},
       [RTE_FLOW_ITEM_TYPE_UDP] = {.descr = "RTE_FLOW_ITEM_TYPE_UDP"},
       [RTE_FLOW_ITEM_TYPE_TCP] = {.descr = "RTE_FLOW_ITEM_TYPE_TCP"},
       [RTE_FLOW_ITEM_TYPE_SCTP] = {.descr = "RTE_FLOW_ITEM_TYPE_SCTP"},
       [RTE_FLOW_ITEM_TYPE_VXLAN] = {.descr = "RTE_FLOW_ITEM_TYPE_VXLAN"},
       [RTE_FLOW_ITEM_TYPE_E_TAG] = {.descr = "RTE_FLOW_ITEM_TYPE_E_TAG"},
       [RTE_FLOW_ITEM_TYPE_NVGRE] = {.descr = "RTE_FLOW_ITEM_TYPE_NVGRE"},
    };
    static const struct flow_action {
       const char *descr;
    } flow_action_desc[] = {
       [RTE_FLOW_ACTION_TYPE_END] = {.descr = "RTE_FLOW_ACTION_TYPE_END"},
       [RTE_FLOW_ACTION_TYPE_VOID] = {.descr = "RTE_FLOW_ACTION_TYPE_VOID"},
       [RTE_FLOW_ACTION_TYPE_PASSTHRU] = {.descr = "RTE_FLOW_ACTION_TYPE_PASSTHRU"},
       [RTE_FLOW_ACTION_TYPE_MARK] = {.descr = "RTE_FLOW_ACTION_TYPE_MARK"},
       [RTE_FLOW_ACTION_TYPE_FLAG] = {.descr = "RTE_FLOW_ACTION_TYPE_FLAG"},
       [RTE_FLOW_ACTION_TYPE_QUEUE] = {.descr = "RTE_FLOW_ACTION_TYPE_QUEUE"},
       [RTE_FLOW_ACTION_TYPE_DROP] = {.descr = "RTE_FLOW_ACTION_TYPE_DROP"},
       [RTE_FLOW_ACTION_TYPE_COUNT] = {.descr = "RTE_FLOW_ACTION_TYPE_COUNT"},
       [RTE_FLOW_ACTION_TYPE_DUP] = {.descr = "RTE_FLOW_ACTION_TYPE_DUP"},
       [RTE_FLOW_ACTION_TYPE_RSS] = {.descr = "RTE_FLOW_ACTION_TYPE_RSS"},
       [RTE_FLOW_ACTION_TYPE_PF] = {.descr = "RTE_FLOW_ACTION_TYPE_PF"},
       [RTE_FLOW_ACTION_TYPE_VF] = {.descr = "RTE_FLOW_ACTION_TYPE_VF"},
    };
    struct netdev_dpdk *dev = netdev_dpdk_cast(netdev);
    const struct rte_flow_attr flow_attr = {.group=0,.priority=0,.ingress=1,.egress=0};
    struct flow_pattern {
        uint32_t max;
        struct rte_flow_item items[];
    } *flow = NULL;

    /* Handle actions */
    struct flow_actions {
        uint32_t max;
        struct rte_flow_action actions[];
    } *rte_actions = NULL;
    uint8_t *ipv4_next_proto_mask = NULL;


    if (!dev->hw_offload) return 0;

    if (VLOG_IS_DBG_ENABLED()) {
        int x;
        match_print(match);
        uint8_t *bb = (uint8_t *)&match->flow;
        fprintf(stdout,"Flow:");
          for(x=0; x<sizeof(match->flow); x++) {
            if(!(x%16)) {
                fprintf(stdout, "\n>>%02d :", x);
            }
            fprintf(stdout, " %02X", *(bb+x));
          }
          fprintf(stdout, "\n----------------------\n");

          fprintf(stdout, "Flow mask:");
          bb = (uint8_t *)&match->wc.masks;
          for(x=0; x<sizeof(match->wc.masks); x++) {
            if(!(x%16)) {
                fprintf(stdout, "\n>>%02d :", x);
            }
            fprintf(stdout, " %02X", *(bb+x));
        }
        fprintf(stdout, "\n----------------------\n");
    }

#define CHECK_NONZERO(addr, size) if (!bitset) {\
uint8_t *padr = (uint8_t *)(addr); int it;\
for (it = 0; it < (size); it++) {\
    if (*(padr++) != 0) {\
        bitset = 1;break;\
    }\
}}
#define CHECK_NONZEROSIMPLE(var)  if (!bitset && (var) != 0) bitset = 1;

    /* Create a wc-zeroed version of flow */
    struct match match_zero_wc;
    match_init(&match_zero_wc, &match->flow, &match->wc);

    /*
     * Check all bits that we cannot yet handle
     */
    int bitset = 0;
    CHECK_NONZERO(&match_zero_wc.flow.tunnel, sizeof(match_zero_wc.flow.tunnel));
    CHECK_NONZEROSIMPLE(match->wc.masks.metadata);
    CHECK_NONZEROSIMPLE(match->wc.masks.skb_priority);
    CHECK_NONZEROSIMPLE(match->wc.masks.pkt_mark);
    CHECK_NONZEROSIMPLE(match->wc.masks.dp_hash);

    /* recirc id must be zero */
    CHECK_NONZEROSIMPLE(match_zero_wc.flow.recirc_id);

    CHECK_NONZEROSIMPLE(match->wc.masks.ct_state);
    CHECK_NONZEROSIMPLE(match->wc.masks.ct_zone);
    CHECK_NONZEROSIMPLE(match->wc.masks.ct_mark);
    CHECK_NONZEROSIMPLE(match->wc.masks.ct_label.u64.hi);
    CHECK_NONZEROSIMPLE(match->wc.masks.ct_label.u64.lo);
    CHECK_NONZEROSIMPLE(match->wc.masks.conj_id);
    CHECK_NONZEROSIMPLE(match->wc.masks.actset_output);
    /* L2 - unsupported */
    CHECK_NONZERO(&match->wc.masks.mpls_lse, sizeof(match_zero_wc.flow.mpls_lse)/sizeof(ovs_be32));
    /* L3 - unsupported */
    CHECK_NONZERO(&match->wc.masks.ipv6_src, sizeof(struct in6_addr));
    CHECK_NONZERO(&match->wc.masks.ipv6_dst, sizeof(struct in6_addr));
    CHECK_NONZEROSIMPLE(match->wc.masks.ipv6_label);
    CHECK_NONZERO(&match->wc.masks.nd_target, sizeof(struct in6_addr));
    CHECK_NONZERO(&match->wc.masks.arp_sha, sizeof(struct eth_addr));
    CHECK_NONZERO(&match->wc.masks.arp_tha, sizeof(struct eth_addr));

    /* If fragmented, then don't HW accelerate - for now */
    CHECK_NONZEROSIMPLE(match_zero_wc.flow.nw_frag);

    /* L4 - unsupported */
    CHECK_NONZEROSIMPLE(match->wc.masks.igmp_group_ip4);

    if (bitset) {
        VLOG_INFO("Cannot HW accelerate this flow");
        return 0;
    }

    /*
     * Now convert flow+masks to DPDK RTE FLOW and sent it to eth_dev filter control
     */

#define FLOW_MAX_ITEMS  100
#define INC_FLOW_ITEM_CNT(cnt) if (cnt < FLOW_MAX_ITEMS) cnt++; else \
{VLOG_ERR("Too many flow items for hw offload");goto error_exit;}

    flow = rte_zmalloc(NULL, sizeof(struct flow_pattern) + 100 * sizeof(struct rte_flow_item), 0);
    flow->max = 0;

    /* Check if any frame header fields must be matched */
    if (match) {

        /* If virtual port is requested as input port, then change input port id */
        if (hw_port_id >= 0 && hw_port_id < MAX_HW_PORT_CNT &&
            hw_port_id != dev->port_id) {
            struct rte_flow_item_port *port_item = rte_malloc(NULL, sizeof(struct rte_flow_item_port), 0);

            if (port_item) {
                port_item->index = (uint32_t)hw_port_id;
                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_PORT;
                flow->items[flow->max].spec = (void *)port_item;
                flow->items[flow->max].mask = (void *)NULL;
                INC_FLOW_ITEM_CNT(flow->max);
            }
        }

        /* Check if any eth hdr info must be specified */
        /* ETH */
        if (match->wc.masks.dl_src.be16[0] || match->wc.masks.dl_src.be16[1] || match->wc.masks.dl_src.be16[2] ||
            match->wc.masks.dl_dst.be16[0] || match->wc.masks.dl_dst.be16[1] || match->wc.masks.dl_dst.be16[2] ||
            (match->wc.masks.vlan_tci && match->flow.vlan_tci)) {

            int alloc_size = sizeof(struct rte_flow_item_eth) + (match->wc.masks.vlan_tci)?sizeof(uint32_t):0;
            struct rte_flow_item_eth *item_eth = rte_malloc(NULL, alloc_size, 0);
            struct rte_flow_item_eth *item_eth_masks = rte_zmalloc(NULL, alloc_size, 0);

            if (item_eth && item_eth_masks) {
                rte_memcpy(&item_eth->dst, &match->flow.dl_dst, sizeof(item_eth->dst));
                rte_memcpy(&item_eth->src, &match->flow.dl_src, sizeof(item_eth->src));
                item_eth->type = match->flow.dl_type;

                rte_memcpy(&item_eth_masks->dst, &match->wc.masks.dl_dst, sizeof(item_eth_masks->dst));
                rte_memcpy(&item_eth_masks->src, &match->wc.masks.dl_src, sizeof(item_eth_masks->src));

                item_eth_masks->type = match->wc.masks.dl_type;

                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_ETH;
                flow->items[flow->max].spec = (void *)item_eth;
                flow->items[flow->max].mask = (void *)item_eth_masks;
                INC_FLOW_ITEM_CNT(flow->max);

                /* VLAN TCI != 0 */
                if (match->wc.masks.vlan_tci && match->flow.vlan_tci) {
                    struct rte_flow_item_vlan *vlan = rte_malloc(NULL, sizeof(struct rte_flow_item_vlan), 0);
                    struct rte_flow_item_vlan *vlan_mask = rte_malloc(NULL, sizeof(struct rte_flow_item_vlan), 0);

                    if (vlan && vlan_mask) {
                        vlan->tpid = 0x8100;
                        vlan->tci = match->flow.vlan_tci;
                        vlan_mask->tpid = 0xffff;
                        vlan_mask->tci = match->wc.masks.vlan_tci;

                        flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_VLAN;
                        flow->items[flow->max].spec = (void *)vlan;
                        flow->items[flow->max].mask = (void *)vlan_mask;
                        INC_FLOW_ITEM_CNT(flow->max);
                    } else {
                        /* Failed allocation */
                        VLOG_ERR("failed to allocate RTE_FLOW vlan item");
                        goto error_exit;
                    }
                }

            } else {
                /* Failed allocation */
                VLOG_ERR("failed to allocate RTE_FLOW eth item");
                goto error_exit;
            }
        }

        /* Check if any ip_hdr info must be specified */
        /* IP v4 */
        uint8_t proto = 0;
        if ((match->flow.dl_type == ntohs(ETH_TYPE_IP)) &&
            (match->wc.masks.nw_src || match->wc.masks.nw_dst ||
             match->wc.masks.nw_tos || match->wc.masks.nw_ttl || match->wc.masks.nw_proto)) {
            struct rte_flow_item_ipv4 *item_ipv4 = rte_zmalloc(NULL, sizeof(struct rte_flow_item_ipv4), 0);
            struct rte_flow_item_ipv4 *item_ipv4_masks = rte_zmalloc(NULL, sizeof(struct rte_flow_item_ipv4), 0);

            if (item_ipv4 && item_ipv4_masks) {

                item_ipv4->hdr.type_of_service = match->flow.nw_tos;
                item_ipv4_masks->hdr.type_of_service = match->wc.masks.nw_tos;
                item_ipv4->hdr.time_to_live = match->flow.nw_tos;
                item_ipv4_masks->hdr.time_to_live = match->wc.masks.nw_tos;
                item_ipv4->hdr.next_proto_id = match->flow.nw_proto;
                item_ipv4_masks->hdr.next_proto_id = match->wc.masks.nw_proto;

                /* Save proto for L4 protocol setup */
                proto = item_ipv4->hdr.next_proto_id & item_ipv4_masks->hdr.next_proto_id;

                /* Remember proto mask address for later modification */
                ipv4_next_proto_mask = &item_ipv4_masks->hdr.next_proto_id;

                item_ipv4->hdr.src_addr = match->flow.nw_src;
                item_ipv4_masks->hdr.src_addr = match->wc.masks.nw_src;
                item_ipv4->hdr.dst_addr = match->flow.nw_dst;
                item_ipv4_masks->hdr.dst_addr = match->wc.masks.nw_dst;

                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_IPV4;
                flow->items[flow->max].spec = (void *)item_ipv4;
                flow->items[flow->max].mask = (void *)item_ipv4_masks;
                INC_FLOW_ITEM_CNT(flow->max);
            } else {
                /* Failed allocation */
                VLOG_ERR("failed to allocate RTE_FLOW ipv4 item");
                goto error_exit;
            }
        } /* End IPv4 */

        if (proto != IPPROTO_ICMP && proto != IPPROTO_UDP && proto != IPPROTO_SCTP && proto != IPPROTO_TCP &&
            (match->wc.masks.tp_src || match->wc.masks.tp_dst || match->wc.masks.tcp_flags)) {
            VLOG_INFO("L4 Protocol not supported");
            goto error_exit;
        }

        if ((proto == IPPROTO_UDP) && (match->wc.masks.tp_src || match->wc.masks.tp_dst)) {
            struct rte_flow_item_udp *item_udp = rte_zmalloc(NULL, sizeof(struct rte_flow_item_udp), 0);
            struct rte_flow_item_udp *item_udp_masks = rte_zmalloc(NULL, sizeof(struct rte_flow_item_udp), 0);
            if (item_udp && item_udp_masks) {
                item_udp->hdr.src_port = match->flow.tp_src;
                item_udp_masks->hdr.src_port = match->wc.masks.tp_src;
                item_udp->hdr.dst_port = match->flow.tp_dst;
                item_udp_masks->hdr.dst_port = match->wc.masks.tp_dst;

                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_UDP;
                flow->items[flow->max].spec = (void *)item_udp;
                flow->items[flow->max].mask = (void *)item_udp_masks;
                INC_FLOW_ITEM_CNT(flow->max);

                /* proto == UDP and ITEM_TYPE_UDP, thus no need for proto match */
                if (ipv4_next_proto_mask) *ipv4_next_proto_mask = 0;
            } else {
                /* Failed allocation */
                VLOG_ERR("failed to allocate RTE_FLOW udp item");
                goto error_exit;
            }
        }

        if ((proto == IPPROTO_SCTP) && (match->wc.masks.tp_src || match->wc.masks.tp_dst)) {
            struct rte_flow_item_sctp *item_sctp = rte_zmalloc(NULL, sizeof(struct rte_flow_item_sctp), 0);
            struct rte_flow_item_sctp *item_sctp_masks = rte_zmalloc(NULL, sizeof(struct rte_flow_item_sctp), 0);
            if (item_sctp && item_sctp_masks) {
                item_sctp->hdr.src_port = match->flow.tp_src;
                item_sctp_masks->hdr.src_port = match->wc.masks.tp_src;
                item_sctp->hdr.dst_port = match->flow.tp_dst;
                item_sctp_masks->hdr.dst_port = match->wc.masks.tp_dst;

                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_SCTP;
                flow->items[flow->max].spec = (void *)item_sctp;
                flow->items[flow->max].mask = (void *)item_sctp_masks;
                INC_FLOW_ITEM_CNT(flow->max);

                /* proto == SCTP and ITEM_TYPE_SCTP, thus no need for proto match */
                if (ipv4_next_proto_mask) *ipv4_next_proto_mask = 0;
            } else {
                /* Failed allocation */
                VLOG_ERR("failed to allocate RTE_FLOW sctp item");
                goto error_exit;
            }
        }

        if ((proto == IPPROTO_ICMP) && (match->wc.masks.tp_src || match->wc.masks.tp_dst)) {
            struct rte_flow_item_icmp *item_icmp = rte_zmalloc(NULL, sizeof(struct rte_flow_item_icmp), 0);
            struct rte_flow_item_icmp *item_icmp_masks = rte_zmalloc(NULL, sizeof(struct rte_flow_item_icmp), 0);
            if (item_icmp && item_icmp_masks) {

                item_icmp->hdr.icmp_type = (uint8_t)ntohs(match->flow.tp_src);
                item_icmp_masks->hdr.icmp_type = (uint8_t)ntohs(match->wc.masks.tp_src);
                item_icmp->hdr.icmp_code = (uint8_t)ntohs(match->flow.tp_dst);
                item_icmp_masks->hdr.icmp_code = (uint8_t)ntohs(match->wc.masks.tp_dst);

                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_ICMP;
                flow->items[flow->max].spec = (void *)item_icmp;
                flow->items[flow->max].mask = (void *)item_icmp_masks;
                INC_FLOW_ITEM_CNT(flow->max);

                /* proto == ICMP and ITEM_TYPE_ICMP, thus no need for proto match */
                if (ipv4_next_proto_mask) *ipv4_next_proto_mask = 0;
            } else {
                /* Failed allocation */
                VLOG_ERR("failed to allocate RTE_FLOW icmp item");
                goto error_exit;
            }
        }

        if ((proto == IPPROTO_TCP) &&
            (match->wc.masks.tp_src || match->wc.masks.tp_dst || match->wc.masks.tcp_flags)) {
            struct rte_flow_item_tcp *item_tcp = rte_zmalloc(NULL, sizeof(struct rte_flow_item_tcp), 0);
            struct rte_flow_item_tcp *item_tcp_masks = rte_zmalloc(NULL, sizeof(struct rte_flow_item_tcp), 0);
            if (item_tcp && item_tcp_masks) {
                item_tcp->hdr.src_port = match->flow.tp_src;
                item_tcp_masks->hdr.src_port = match->wc.masks.tp_src;
                item_tcp->hdr.dst_port = match->flow.tp_dst;
                item_tcp_masks->hdr.dst_port = match->wc.masks.tp_dst;

                item_tcp->hdr.data_off = (uint8_t)(ntohs(match->flow.tcp_flags) >> 8);
                item_tcp_masks->hdr.data_off = (uint8_t)(ntohs(match->wc.masks.tcp_flags) >> 8);
                item_tcp->hdr.tcp_flags = (uint8_t)(ntohs(match->flow.tcp_flags) & 0xff);
                item_tcp_masks->hdr.tcp_flags = (uint8_t)(ntohs(match->wc.masks.tcp_flags) & 0xff);

                flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_TCP;
                flow->items[flow->max].spec = (void *)item_tcp;
                flow->items[flow->max].mask = (void *)item_tcp_masks;
                INC_FLOW_ITEM_CNT(flow->max);

                /* proto == TCP and ITEM_TYPE_TCP, thus no need for proto match */
                if (ipv4_next_proto_mask) *ipv4_next_proto_mask = 0;
            } else {
                /* Failed allocation */
                VLOG_ERR("failed to allocate RTE_FLOW tcp item");
                goto error_exit;
            }
        }

        if (flow->max) {
            flow->items[flow->max].type = RTE_FLOW_ITEM_TYPE_END;
            flow->items[flow->max].spec = (void *)NULL;
            flow->items[flow->max].mask = (void *)NULL;
            INC_FLOW_ITEM_CNT(flow->max);
        }
    }

    int actions_supported = 1;
    int output_specifed = 0;
    int count_idx = -1;

    rte_actions = rte_zmalloc(NULL, sizeof(struct flow_actions) + (actions_len + 2) * sizeof(struct rte_flow_action), 0);
    if (!rte_actions) {
        VLOG_ERR("failed to allocate RTE_FLOW actions");
        goto error_exit;
    }

    if (actions_len == 0) {
        /* Drop-action to be performed */
        rte_actions->actions[rte_actions->max].type = RTE_FLOW_ACTION_TYPE_DROP;
        rte_actions->max++;
        rte_actions->actions[rte_actions->max].type = RTE_FLOW_ACTION_TYPE_END;
        rte_actions->max++;
    } else {

        VLOG_DBG("ODP-IN-PORT %i has HW-PORT-ID %i\n",match->flow.in_port.odp_port, hw_port_id);

        if (actions_supported && nl_actions) {
            const struct nlattr *a;
            unsigned int left;
            NL_ATTR_FOR_EACH_UNSAFE(a, left, nl_actions, actions_len) {
                int type = nl_attr_type(a);
                switch (type) {
                case OVS_ACTION_ATTR_OUTPUT:
                {
                    struct rte_flow_action_mark *id;
                    uint32_t odp_port_out = nl_attr_get_u32(a);
                    uint32_t hw_port = (uint32_t)-1;

                    /* Check if odp-output-id is mapped to a physical port-id. */
                    struct netdev_dpdk *dpdk_dev;

                    /* If output has already one specified - discard offload */
                    if (output_specifed) {
                        actions_supported = 0;
                        break;
                    }

                    id = rte_malloc(NULL, sizeof(struct rte_flow_action_mark), 0);
                    if (id) {
                        LIST_FOR_EACH (dpdk_dev, list_node, &dpdk_list) {
                            if (dev != dpdk_dev) ovs_mutex_lock(&dpdk_dev->mutex);
                            if (odp_port_out == dpdk_dev->odp_port_no) {

                                // is a port in HW
                                hw_port = (uint32_t)(dpdk_dev->hw_port_id);

                                if (dev != dpdk_dev) ovs_mutex_unlock(&dpdk_dev->mutex);
                                break;
                            }
                            if (dev != dpdk_dev) ovs_mutex_unlock(&dpdk_dev->mutex);
                        }

                        if (hw_port == (uint32_t)-1) {
                            /* put output odp-port-id instead and set most significant bit to indicate odp port id */
                            id->id = ((uint32_t)flow_id << 16) | (odp_port_out & FLOW_ID_PORT_MASK) | FLOW_ID_ODP_PORT_BIT;
                        } else {
                            /* output-port in mark id lower 15 bits */
                            id->id = ((uint32_t)flow_id << 16) | (hw_port & FLOW_ID_PORT_MASK);
                        }

                        if (flow_stat_support && (*flow_stat_support != 0)) {
                            rte_actions->actions[rte_actions->max].type = RTE_FLOW_ACTION_TYPE_COUNT;
                            count_idx = rte_actions->max;
                            rte_actions->max++;
                        }

                        rte_actions->actions[rte_actions->max].type = RTE_FLOW_ACTION_TYPE_MARK;
                        rte_actions->actions[rte_actions->max].conf = id;
                        rte_actions->max++;
                        output_specifed++;
                    } else {
                        /* Failed allocation */
                        VLOG_ERR("failed to allocate flow id for RTE_MARK action");
                        goto error_exit;
                    }
                    break;
                }

                default:
                    VLOG_DBG("DO NOT SUPPORT ACTION TYPE %i\n", type);
                    actions_supported = 0;
                    break;
                }
            }
            rte_actions->actions[rte_actions->max].type = RTE_FLOW_ACTION_TYPE_END;
            rte_actions->max++;
        }
    }

    /*
     * Delete flow is indicated by:
     * 		1. flow->max == 0 - no flow configuration specified - match is all zero
     * 		2. all actions_supported
     * Drop flow is indicated by:
     * 		1. flow->max > 0 - some flow specified
     * 		2. no output actions specified
     * 		3. all actions_supported (drop action) and flow_handle specified
     */
    {
        struct rte_flow_error error;
        if (flow_handle == NULL) {
            /* should never happen */
            VLOG_ERR("INTERNAL ERRROR - flow_handle of zero");
            goto error_exit;
        }

        if (*flow_handle != (uint64_t)-1) {
            int ret = rte_flow_destroy(dev->port_id, (struct rte_flow *)*flow_handle, &error);
            if (ret == 0) {
                VLOG_INFO("Flow successfully removed. Handle %lx\n", (long unsigned)*flow_handle);
                /* return flow handle untouched - indicate successfully deleted */
            } else {
                VLOG_INFO("RTE_FLOW DESTROY ERROR: %i : Message : %s\n", error.type, error.message);
              *flow_handle = (uint64_t)-1;
            }
        }


        if (flow->max && rte_actions->max && actions_supported) {
            struct rte_flow *flow_res = NULL;
            int try, i;

            if (VLOG_IS_DBG_ENABLED()) {
                VLOG_DBG("Trying to offload the following RTE_FLOW:\nFlow Items:\n");
                int idx = 0;
                while (idx < flow->max) {
                    printf("%s\n", flow_item_desc[flow->items[idx].type].descr);
                    idx++;
                }
            }

            /* RTE_FLOW trial and error sequence
             * if COUNT/FLOW-STAT supported
             *   1) try with COUNT and without QUEUE in ACTIONS
             *   2) try with COUNT and QUEUE=0 in ACTIONS
             *   3) try without COUNT and with QUEUE=0 in ACTIONS
             * otherwise
             *   1) try without COUNT and QUEUE
             *   2) try without COUNT and with QUEUE=0
             */

            for (try = 0; try < 3; try++) {

                if (VLOG_IS_DBG_ENABLED()) {
                    VLOG_DBG("\n%i try: Flow Actions:\n", try+1);
                    int idx = 0;
                    while (idx < rte_actions->max) {
                        VLOG_DBG("%s\n", flow_action_desc[rte_actions->actions[idx].type].descr);
                        idx++;
                    }
                }

                flow_res = rte_flow_create(dev->port_id, &flow_attr, flow->items, rte_actions->actions, &error);
                if (flow_res != NULL) break; // success!

                if (try == 0) {
                    /* Queue may be needed  - use queue 0 */
                    struct rte_flow_action_queue *queue = rte_zmalloc(NULL, sizeof(struct rte_flow_action_queue), 0);
                    if (queue) {
                        /* insert QUEUE after COUNT or at index 0 */
                        int cnt_idx = (count_idx < 0) ? 0 : count_idx + 1;

                        for (i = rte_actions->max; i > cnt_idx; i--) {
                            rte_actions->actions[i].type = rte_actions->actions[i-1].type;
                            rte_actions->actions[i].conf = rte_actions->actions[i-1].conf;
                        }
                        rte_actions->actions[cnt_idx].type = RTE_FLOW_ACTION_TYPE_QUEUE;
                        rte_actions->actions[cnt_idx].conf = queue; // index = 0
                        rte_actions->max++;
                    } else {
                        VLOG_ERR("failed to allocate queue struct for RTE_QUEUE action");
                        goto error_exit;
                    }
                } else
                    if (try == 1) {
                        if (count_idx < 0) break; // No COUNT support
                        /* remove COUNT */
                        for (i = (count_idx + 1);i < rte_actions->max; i++) {
                            rte_actions->actions[i - 1].type = rte_actions->actions[i].type;
                            rte_actions->actions[i - 1].conf = rte_actions->actions[i].conf;
                        }
                        count_idx = -1;
                        rte_actions->max--;
                    }
            }

            if (flow_res != NULL) {
                VLOG_INFO("Flow successfully offloaded. Handle %lx", (uint64_t)flow_res);
               *flow_handle = (uint64_t)flow_res;

               if (flow_stat_support)
                   *flow_stat_support = (count_idx == 1)?1:0;

            } else {
                VLOG_ERR("RTE_FLOW CREATE ERROR: %i : Message : %s", error.type, error.message);
                *flow_handle = (uint64_t)-1;
            }
        }

error_exit:
        if (flow) {
            while (flow->max) {
                flow->max--;
                if (flow->items[flow->max].spec)
                    rte_free((void *)flow->items[flow->max].spec);
                if (flow->items[flow->max].mask)
                    rte_free((void *)flow->items[flow->max].mask);
            };
            rte_free(flow);
        }
        if (rte_actions) {
            while (rte_actions->max) {
                rte_actions->max--;
                if (rte_actions->actions[rte_actions->max].conf)
                    rte_free((void *)rte_actions->actions[rte_actions->max].conf);
            }
            rte_free(rte_actions);
        }

    }
    return 0;
}


In general, some flows will not be offload-able and will simply exit out. All current fields not supported yet is checked upon
in the beginning of this function. There is still some that could be supported. Must be done later though.
Only 1 output action is currently supported and will exit out if more are specified. The main reason for this is the lack of a generic
output id specification in RTE FLOW. Otherwise this decision should be done in the DPDK PMD module instead.
The output port id is masked into the RTE_TYPE_ACTION_MARK (Flow ID) field together with the cache index of the netdev_hw_offload 
flow cache:

(flow_id << 16) | hw_port_id 
or 
(flow_id << 16) | 0x8000 + odp_port_id

Flow_id is 16 bit and represents the index into the netdev_hw_offload cache. The odp_port_id is used if the hw-port-id id not known.

Since RTE FLOW is trial and error based, a few retries are done based on different actions. The RTE_FLOW_ACTION_TYPE_QUEUE is not 
really needed in this proposal, because the MARK (hint) does tell about the flow hit and target destination. But some NICs may need it. 
You could argue that 1 queue per flow should be created and thus implicitly know the destination. The main reason for not using 1 queue 
per flow, is the limitation of the number of queues typically available. Furthermore, it also makes it harder to implement in OVS 
together with RSS.

The trial and error could be extended possibly a bit more depending on the various NICs special behavior, however, for now it looks like 
this:

if COUNT/FLOW-STAT supported
  1) try with COUNT and without QUEUE in ACTIONS
  2) try with COUNT and QUEUE=0 in ACTIONS
  3) try without COUNT and with QUEUE=0 in ACTIONS
otherwise
  1) try without COUNT and QUEUE
  2) try without COUNT and with QUEUE=0

Note, RTE_FLOW_ACTION_TYPE_COUNT is used to query NIC for Flow statistics. However, it might not be supported.



The get_flow_stats netdev_class method implementation:

static int
netdev_dpdk_get_flow_stats(struct netdev_rxq *rxq, struct netdev_flow_stats *flow_stats)
{
    struct netdev_dpdk *dev = netdev_dpdk_cast(rxq->netdev);
	struct rte_flow_query_count qry_cnt;
	struct rte_flow_error error;

	if (!dev->hw_offload) return EOPNOTSUPP;

	int res = 0;
	int i;
	int port_id = dev->port_id;

	for (i = 0; i < flow_stats->num; i++) {
		qry_cnt.reset = 1;

		res = rte_flow_query(port_id,
				(struct rte_flow *)flow_stats->flow_stat[i].flow_id,
				RTE_FLOW_ACTION_TYPE_COUNT,
				&qry_cnt,
				&error);

		if (res == 0) {
			flow_stats->flow_stat[i].packets = qry_cnt.hits_set?qry_cnt.hits:0;
			flow_stats->flow_stat[i].bytes = qry_cnt.bytes_set?qry_cnt.bytes:0;
		} else {
			flow_stats->flow_stat[i].packets = 0;
			flow_stats->flow_stat[i].bytes = 0;
		}
		flow_stats->flow_stat[i].errors = 0;
	}

	return 0;
}

If supported by NIC and DPDK PMD, flow statistics is queried through this function, using rte_flow_query.



Added 1 module (shown here for completeness):

netdev-hw-offload.c
===================

#include <config.h>
#include "netdev-provider.h"
#include "openvswitch/match.h"
#include "openvswitch/vlog.h"
#include "dp-packet.h"
#include "cmap.h"
#include "netdev-hw-offload.h"

VLOG_DEFINE_THIS_MODULE(netdev_hw_offload);


struct netdev_ufid_map {
	struct cmap_node cmap_node;
	ovs_u128 ufid;
	void    *hw_flow_id;
	uint16_t node_tbl_idx;
	/* Only for netdev providers not able to deliver flow statistics */
	uint64_t packets;
	uint64_t bytes;
};

static struct netdev_ufid_map *ufid_node_tbl[0x10000];
static uint16_t next_node_idx;
static struct ovs_mutex ufid_node_mutex = OVS_MUTEX_INITIALIZER;



static struct netdev_ufid_map *get_ufid_node(const struct cmap *cmap, const ovs_u128 *ufid)
{
	const struct cmap_node *node;
	struct netdev_ufid_map *ufid_node = NULL;

	node = cmap_find(cmap, ufid->u32[0]);

	CMAP_NODE_FOR_EACH(ufid_node, cmap_node, node) {
		VLOG_DBG("ufid map %08x%08x, %lx\n", (unsigned int)ufid_node->ufid.u64.hi,
		        (unsigned int)ufid_node->ufid.u64.lo, (long unsigned int)ufid_node->hw_flow_id);
		if (ufid_node->ufid.u64.hi == ufid->u64.hi && ufid_node->ufid.u64.lo == ufid->u64.lo) {
			/* Found flow in cache. */
			return ufid_node;
		}
	}
	return NULL;
}


void netdev_update_flow_stats(struct netdev_rxq *rxq, struct dp_packet_batch *batch)
{
    uint32_t flow_id, pre_id;
    uint32_t i;
    struct netdev_ufid_map *ufid_node;

    if (rxq->netdev->flow_stat_hw_support == 1) return;

    pre_id = (uint32_t)-1;
    ufid_node = NULL;

    ovs_mutex_lock(&ufid_node_mutex);
    for (i = 0; i < batch->count; i++) {
        flow_id = dp_packet_get_pre_classified_flow_id(batch->packets[i]);

        if (pre_id != flow_id) {
            ufid_node = ufid_node_tbl[(uint16_t)(flow_id >> 16)];
            pre_id = flow_id;
        }
        if (ufid_node != NULL) {
            ufid_node->packets++;
            ufid_node->bytes += dp_packet_size(batch->packets[i]);
        }
    }
    ovs_mutex_unlock(&ufid_node_mutex);
}


int netdev_hw_get_stats_from_dev(struct netdev_rxq *rxq, struct netdev_flow_stats **aflow_stats)
{
	struct netdev_flow_stats *flow_stats = NULL;
	struct netdev_ufid_map *ufid_node = NULL;
	uint32_t i;
	bool hw_support = ((rxq->netdev->flow_stat_hw_support == 1) && netdev_get_flow_stats_supported(rxq));

	*aflow_stats = NULL;
	/* Collect all entries in cmap */
	size_t num = cmap_count(&rxq->netdev->ufid_map);
	if (num == 0) return 0;

	flow_stats = xcalloc(1, sizeof(struct netdev_flow_stats) + num * sizeof(struct flow_stat_elem));
	if (flow_stats == NULL) return -1;

	flow_stats->num = num;
	i = 0;
	CMAP_FOR_EACH(ufid_node, cmap_node, &rxq->netdev->ufid_map) {
		flow_stats->flow_stat[i].flow_id = ufid_node->hw_flow_id;
		flow_stats->flow_stat[i].ufid = ufid_node->ufid;

		if (!hw_support) {
		    flow_stats->flow_stat[i].bytes = ufid_node->bytes;
		    ufid_node->bytes = 0;
            flow_stats->flow_stat[i].packets = ufid_node->packets;
            ufid_node->packets = 0;
		}

		i++;
		if (i > flow_stats->num) break;
	}

	if (flow_stats->num != i) {
		VLOG_WARN("WARNING :: flow_stats->num = %u, i = %i\n", (unsigned int)flow_stats->num, i);
		flow_stats->num = i;
	}

	/* Get all HW flow stats for this device */
	if (hw_support) {
	    int err;
        if ((err = netdev_get_flow_stats(rxq, flow_stats)) != 0) {
            free(flow_stats);
            return err;
        }
	}

	*aflow_stats = flow_stats;
	return 0;
}

int netdev_hw_free_stats_from_dev(struct netdev_rxq *rxq OVS_UNUSED, struct netdev_flow_stats **aflow_stats)
{
	if (aflow_stats == NULL || *aflow_stats) return 0;
	free(*aflow_stats);
	return 0;
}



int netdev_try_hw_flow_offload(struct netdev *netdev, uint32_t hw_port_id, struct match *match, const ovs_u128 *ufid,
		const struct nlattr *actions, size_t actions_len)
{
	int delete_match = 0;
	struct netdev_ufid_map *ufid_node = NULL;

	if (netdev == NULL || !netdev_has_hw_flow_offload(netdev)) return 0;

	ovs_mutex_lock(&netdev->mutex);

	ufid_node = get_ufid_node(&netdev->ufid_map, ufid);

	if (match == NULL) {
		/* Used for deletion of flow */
		match = xcalloc(1, sizeof(struct match));
		if (!match) {
			ovs_mutex_unlock(&netdev->mutex);
			return -1;
		}
		delete_match = 1;
	}

	uint64_t flow_handle;
	int flow_stat_support = netdev->flow_stat_hw_support;

	if (ufid_node != NULL) {
		flow_handle = (uint64_t)ufid_node->hw_flow_id;
	} else {
		flow_handle = (uint64_t)-1;
	}

	ovs_mutex_lock(&ufid_node_mutex);

	/* Find next empty slot for next time ufid node lookup idx. */
    uint16_t idx_for_next_node = (uint16_t)(next_node_idx + 1);
    while ((ufid_node_tbl[idx_for_next_node] != NULL) && (idx_for_next_node != next_node_idx)) {
        idx_for_next_node++;
    }
    if (ufid_node_tbl[idx_for_next_node]) {
        /* ufid node lookup table full */
        if (match && delete_match)
            free(match);
        ovs_mutex_unlock(&ufid_node_mutex);
        ovs_mutex_unlock(&netdev->mutex);
        return 0;
    }


    netdev_hw_flow_offload(netdev, match, hw_port_id, actions, actions_len, &flow_stat_support, next_node_idx, &flow_handle);

	if (flow_handle == (uint64_t)-1) {
		/* HW action failed or not supported by this device */
		if (ufid_node) {
			if (delete_match)
				VLOG_ERR("deletion of flow %08x%08x in HW failed!", (unsigned int)ufid_node->ufid.u64.hi,
				        (unsigned int)ufid_node->ufid.u64.lo);
			VLOG_ERR("ERROR flow removed after failed in HW\n");
			ufid_node_tbl[ufid_node->node_tbl_idx] = NULL;
			cmap_remove(&netdev->ufid_map, &ufid_node->cmap_node, ufid->u32[0]);
		}
		VLOG_DBG("Flow could not be HW accelerated or not supported by this device\n");
	} else {
		/* HW action succeeded */
		if (delete_match) {
			if (ufid_node != NULL) {
	            ufid_node_tbl[ufid_node->node_tbl_idx] = NULL;
				cmap_remove(&netdev->ufid_map, &ufid_node->cmap_node, ufid->u32[0]);

				VLOG_DBG("removed from ufid map %08x%08x, %lx\n", (unsigned int)ufid_node->ufid.u64.hi,
				        (unsigned int)ufid_node->ufid.u64.lo, flow_handle);
			}
		} else {
			VLOG_DBG("Flow added to HW with flow handle: %lx\n", flow_handle);

			if (ufid_node == NULL) {
				/* Create new entry */
				ufid_node = xmalloc(sizeof(struct netdev_ufid_map));
				cmap_insert(&netdev->ufid_map, &ufid_node->cmap_node, ufid->u32[0]);
			} /* otherwise reuse */
			ufid_node->ufid = *ufid;
			ufid_node->hw_flow_id = (void *)flow_handle;
			/* Add unique flow id */
			ufid_node->node_tbl_idx = next_node_idx;

			ufid_node_tbl[next_node_idx] = ufid_node;
			next_node_idx = idx_for_next_node;

			VLOG_DBG("insert into ufid map %08x%08x, %lx\n", (unsigned int)ufid_node->ufid.u64.hi,
			        (unsigned int)ufid_node->ufid.u64.lo, (long unsigned int)ufid_node->hw_flow_id);
		}

		/* Confirm support for flow statistics support by NIC */
		if (netdev->flow_stat_hw_support < 0)
		    netdev->flow_stat_hw_support = flow_stat_support;
	}
    ovs_mutex_unlock(&ufid_node_mutex);

	if (match && delete_match)
		free(match);

	ovs_mutex_unlock(&netdev->mutex);
	return 0;
}


An example of how to test this on a XL710:
The XL710 is apparently limited in the variation of flow classification definitions it can handle, using its RTE FLOW
implementation (FDIR). But a 5-tuple specification seems to work well. 

# ovs-ofctl add-flow ovsbr0 in_port=1,dl_type=0x0800,nw_src=192.168.1.2,nw_dst=192.168.1.8,nw_proto=17,tp_src=10025,\
			tp_dst=10026,actions:output=3
```





转载自：https://www.cnblogs.com/dream397/p/14744030.html