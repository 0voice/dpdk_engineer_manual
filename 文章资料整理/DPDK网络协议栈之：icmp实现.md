# DPDK网络协议栈之：icmp实现

## 1.前言

基于udp协议，在实现 arp 功能后，本机ping完虚拟机之后，虚拟机需要将相关的icmp信息返回给本机。

## 2.先添加相关标识

`#define ENABLE_ICMP 1`

## 3.实现checksum

定义创建icmp包的函数前，需要自己实现checksum



```c++
#if ENABLE_ICMP

static uint16_t ng_checksum(uint16_t *addr, int count)
{

	register long sum = 0;

	while (count > 1)
	{
		sum += *(unsigned short *)addr++;
		count -= 2;
	}

	if (count > 0)
	{
		sum += *(unsigned char *)addr;
	}

	while (sum >> 16)
	{
		sum = (sum & 0xffff) + (sum >> 16);
	}

	return ~sum;
}

#endif

```

## 4.定义创建icmp包的函数

思路与send一样，只不过要发送的包不是arp包，而是icmp包

```c++
// 根据不同的code处理
#if ENABLE_ICMP

static int ng_encode_icmp_pkt(uint8_t *msg, uint8_t *dst_mac,
							  uint32_t sip, uint32_t dip, uint16_t id, uint16_t seqnb)
{

	// 1 ether
	struct rte_ether_hdr *eth = (struct rte_ether_hdr *)msg;
	rte_memcpy(eth->s_addr.addr_bytes, gSrcMac, RTE_ETHER_ADDR_LEN);
	rte_memcpy(eth->d_addr.addr_bytes, dst_mac, RTE_ETHER_ADDR_LEN);
	eth->ether_type = htons(RTE_ETHER_TYPE_IPV4);

	// 2 ip
	struct rte_ipv4_hdr *ip = (struct rte_ipv4_hdr *)(msg + sizeof(struct rte_ether_hdr));
	ip->version_ihl = 0x45;
	ip->type_of_service = 0;
	ip->total_length = htons(sizeof(struct rte_ipv4_hdr) + sizeof(struct rte_icmp_hdr));
	ip->packet_id = 0;
	ip->fragment_offset = 0;
	ip->time_to_live = 64;			  // ttl = 64
	ip->next_proto_id = IPPROTO_ICMP; //注意这里，与之前ip包的不同
	ip->src_addr = sip;
	ip->dst_addr = dip;

	ip->hdr_checksum = 0;
	ip->hdr_checksum = rte_ipv4_cksum(ip);

	// 3 icmp
	struct rte_icmp_hdr *icmp = (struct rte_icmp_hdr *)(msg + sizeof(struct rte_ether_hdr) + sizeof(struct rte_ipv4_hdr));
	icmp->icmp_type = RTE_IP_ICMP_ECHO_REPLY;
	icmp->icmp_code = 0;
	icmp->icmp_ident = id;
	icmp->icmp_seq_nb = seqnb;

	icmp->icmp_cksum = 0;
	icmp->icmp_cksum = ng_checksum((uint16_t *)icmp, sizeof(struct rte_icmp_hdr));

	return 0;
}

#endif

```

## 5.在mbuf中存储icmp包

注意，此时参数不需要端口号，因为icmp是在网络层的协议，此时并不需要端口号。

```c++
#if ENABLE_ICMP

static struct rte_mbuf *ng_send_icmp(struct rte_mempool *mbuf_pool, uint8_t *dst_mac,
									 uint32_t sip, uint32_t dip, uint16_t id, uint16_t seqnb)
{

	const unsigned total_length = sizeof(struct rte_ether_hdr) + sizeof(struct rte_ipv4_hdr) + sizeof(struct rte_icmp_hdr);

	struct rte_mbuf *mbuf = rte_pktmbuf_alloc(mbuf_pool);
	if (!mbuf)
	{
		rte_exit(EXIT_FAILURE, "rte_pktmbuf_alloc\n");
	}

	mbuf->pkt_len = total_length;
	mbuf->data_len = total_length;

	uint8_t *pkt_data = rte_pktmbuf_mtod(mbuf, uint8_t *);
	ng_encode_icmp_pkt(pkt_data, dst_mac, sip, dip, id, seqnb);

	return mbuf;
}

#endif

```

## 6.main函数中添加

！！！注意：这里代码与处理IP协议并列，代码顺序如下：

![在这里插入图片描述](https://img-blog.csdnimg.cn/502db14e3d334ac7babc2567a5a9c1d2.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZW5jaGFudGVkb3Zv,size_20,color_FFFFFF,t_70,g_se,x_16)

```c++
#if ENABLE_ICMP
			if (IPPROTO_ICMP == iphdr->next_proto_id)
			{
				struct rte_icmp_hdr *icmphdr = (struct rte_icmp_hdr *)(iphdr + 1);

				// 加上打印信息
				struct in_addr addr;
				addr.s_addr = iphdr->src_addr;
				printf("icmp ---> src: %s ", inet_ntoa(addr));

				if (icmphdr->icmp_type == RTE_IP_ICMP_ECHO_REQUEST)
				{
					// 如果是08则实现ping
					addr.s_addr = iphdr->dst_addr;
					printf(" local: %s , type : %d\n", inet_ntoa(addr), icmphdr->icmp_type);

					struct rte_mbuf *txbuf = ng_send_icmp(mbuf_pool, ehdr->s_addr.addr_bytes,
														  iphdr->dst_addr, iphdr->src_addr, icmphdr->icmp_ident, icmphdr->icmp_seq_nb);

					rte_eth_tx_burst(gDpdkPortId, 0, &txbuf, 1);
					rte_pktmbuf_free(txbuf);

					rte_pktmbuf_free(mbufs[i]);
				}
			}
#endif

```

## 7.运行结果

首先，arp已经添加好了

![在这里插入图片描述](https://img-blog.csdnimg.cn/fd1038b7f15e4a639fac276d5ba396c6.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZW5jaGFudGVkb3Zv,size_20,color_FFFFFF,t_70,g_se,x_16)

此时，ping虚拟机
![在这里插入图片描述](https://img-blog.csdnimg.cn/1d428e1bf2484dfe871f8d344987000a.png)
完成了icmp功能后

`./build/dpdk_icmp`

再次ping，则得到：
![在这里插入图片描述](https://img-blog.csdnimg.cn/8f84ff6402dc4f3190f07413d1754546.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZW5jaGFudGVkb3Zv,size_20,color_FFFFFF,t_70,g_se,x_16)
此时，已经有相应的icmp信息了。







原文链接：https://blog.csdn.net/qq_45617555/article/details/122162531

原文作者：enchantedovo