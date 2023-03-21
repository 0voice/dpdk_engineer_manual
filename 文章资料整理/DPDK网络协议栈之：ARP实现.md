# DPDK网络协议栈之：ARP实现

## 1.先添加相关标识

`\#define ENABLE_ARP`		

## 2.定义创建arp包的函数

思路与send一样，只不过要发送的包不是udp包，而是arp包




```c++
#if ENABLE_ARP


static int ng_encode_arp_pkt(uint8_t *msg, uint8_t *dst_mac, uint32_t sip, uint32_t dip) {

	// 1 ethhdr
	struct rte_ether_hdr *eth = (struct rte_ether_hdr *)msg;
	rte_memcpy(eth->s_addr.addr_bytes, gSrcMac, RTE_ETHER_ADDR_LEN);
	rte_memcpy(eth->d_addr.addr_bytes, dst_mac, RTE_ETHER_ADDR_LEN);
	eth->ether_type = htons(RTE_ETHER_TYPE_ARP);

	// 2 arp 
	struct rte_arp_hdr *arp = (struct rte_arp_hdr *)(eth + 1);
	arp->arp_hardware = htons(1);
	arp->arp_protocol = htons(RTE_ETHER_TYPE_IPV4);
	arp->arp_hlen = RTE_ETHER_ADDR_LEN;
	arp->arp_plen = sizeof(uint32_t);
	arp->arp_opcode = htons(2);

	rte_memcpy(arp->arp_data.arp_sha.addr_bytes, gSrcMac, RTE_ETHER_ADDR_LEN);
	rte_memcpy( arp->arp_data.arp_tha.addr_bytes, dst_mac, RTE_ETHER_ADDR_LEN);

	arp->arp_data.arp_sip = sip;
	arp->arp_data.arp_tip = dip;
	
	return 0;

}

#endif

```

## 3.在mbuf中存储arp包

从mempool中拿出mbuf，并且将包信息设置好




```c++
#if ENABLE_ARP


static struct rte_mbuf *ng_send_arp(struct rte_mempool *mbuf_pool, uint8_t *dst_mac, uint32_t sip, uint32_t dip) {

	const unsigned total_length = sizeof(struct rte_ether_hdr) + sizeof(struct rte_arp_hdr);

	struct rte_mbuf *mbuf = rte_pktmbuf_alloc(mbuf_pool);
	if (!mbuf) {
		rte_exit(EXIT_FAILURE, "rte_pktmbuf_alloc\n");
	}

	mbuf->pkt_len = total_length;
	mbuf->data_len = total_length;

	uint8_t *pkt_data = rte_pktmbuf_mtod(mbuf, uint8_t *);
	ng_encode_arp_pkt(pkt_data, dst_mac, sip, dip);

	return mbuf;
}

#endif

```

## 4.main函数中

在收到arp request的时候，dpdk响应发送相应的arp信息到源地址
！！！这里的代码与处理IP协议并列，且代码放在判断是否为ipv4协议前


​            

```c++
#if ENABLE_ARP
            if (ehdr->ether_type == rte_cpu_to_be_16(RTE_ETHER_TYPE_ARP))
            {
                // 如果是arp的协议，需要转成arp结构（以太头后）
                struct rte_arp_hdr *ahdr = rte_pktmbuf_mtod_offset(mbufs[i], struct rte_arp_hdr *, sizeof(struct rte_arp_hdr));
                
                // 打印接收到的数据相关信息
                struct in_addr addr;
				addr.s_addr = ahdr->arp_data.arp_tip;
				printf("arp ---> src: %s ", inet_ntoa(addr));

				addr.s_addr = gLocalIp;
				printf("local: %s \n", inet_ntoa(addr));
                
                if (ahdr->arp_data.arp_tip == gLocalIp)
                {

                    // 接收数据response的响应
                    struct rte_mbuf *arpbuf = ng_send_arp(mbuf_pool, ahdr->arp_data.arp_sha.addr_bytes,
                                                          ahdr->arp_data.arp_tip, ahdr->arp_data.arp_sip); // response里的源ip就是request里的目的ip

                    rte_eth_tx_burst(gDpdkPortId, 0, &arpbuf, 1); // 将数据send出去
                    rte_pktmbuf_free(arpbuf);

                    rte_pktmbuf_free(mbufs[i]);
                }

                continue;
            }
#endif


```


## 5.运行结果

先删掉相关的arp信息

![在这里插入图片描述](https://img-blog.csdnimg.cn/2b763e6256a54af9abda7ee155746a5e.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZW5jaGFudGVkb3Zv,size_18,color_FFFFFF,t_70,g_se,x_16)

主机ping虚拟机：
![在这里插入图片描述](https://img-blog.csdnimg.cn/01bc2005d24c4c5bbf8feba74e638203.png)
可以发现dpdk将arp信息响应给了主机，主机自动添加上了响应的arp信息

![在这里插入图片描述](https://img-blog.csdnimg.cn/dd38c02e96534a03b13137dc9c3422df.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAZW5jaGFudGVkb3Zv,size_20,color_FFFFFF,t_70,g_se,x_16)

原文链接：**https://blog.csdn.net/qq_45617555/article/details/121974920**

原文作者：enchantedovo