# vxlan转发原理

openstack neutron组件也用到了vxlan，还有dvr，在云计算环境到底分布式网关好还是集中式网关好，vxlan对称还是非对称转发好，什么样的控制平面好，要对这些问题得出结论不管怎样先得深刻理解vxlan转发流程。

三层转发就是查路由表转发，云计算环境下一个租户一个vpn，一个vpn对应一个vrf，一个租户很多vxlan，同一vxlan内二层转发，vxlan之间三层转发，而且还有一些vxlan要出公网，分布式网关中有一个网关作为boarder，把云上路由发布出去或者引进外部路由，这个boader主备或者主主多活负载分担。

##  1.集中式网关

一个租户一个vpn，简化一下一个租户在openstack neutron上有一个router，这个router就是图中的一个VRF。一个vxlan就是neutron中一个network，有一个vni，一个network只有一个子网，这样跨vxlan转发就是跨了网转发，一个vxlan就是一BD(bridge domain)，BD里有MAC转发表，根据MAC做二层转发，BD上有一个bdif接口连接到IP VRF上，流量到这个上去就查路由做三层转发，从其它bdif口出去，就到其它vxlan的BD上了。同时一个BD上还有vxlan tunnel口和vlan口，BD做vlan和vxlan互相转换，一个BD只能有一个vlan id和一个vxlan id，vlan id是本地id，不同节点相同BD上的vxlan id必须相同，vlan id可以不一样。vxlan tunnel是full mesh的，单播报文在BD查MAC找到出口tunnel，tunnel上有源和目的IP，做vxlan encap，然后default vrf出去，vxlan报文来了，解析出vxlan id，根据vxlan id找到BD，在BD里添加vlan tag，然后查MAC转发给本地。对于BUM报文，就是报文目的MAC上BD上没有或者目的MAC比较特殊，vxlan有头端复制或者核心复制，头端复制，由BD复制在所有vxlan tunnel口发一份，vxlan encap的目的IP是远端IP，核心复制就是BD不复制，vxlan encap的目的IP是一个组播IP，由underlay路由器进行复制，underlay得支持组播协议，三层组播协议太复杂了。

VRF中需要路由，BD上需要MAC，假如路由静态配置或者自动生成直连路由，BD上MAC由flooad-learn生成，再加头端复制，vxlan就可以玩转了。但还有个问题，tunnel是怎么建立的，远端IP怎么得来的，图上我把tunnel简化了，而且画在BD上，其实不是这样的，tunnel是所有BD共享的，而且未必需要full-mesh，两个计算节点上的虚拟机没有相同的vni，而且是不同租户的没有三层互通的需求，那么这两台计算节点没必要建立tunnel，BD是共享tunnel的，一个BD一个vni，其实就是vni和tunnel的绑定关系，如果有控制面帮助自动建立tunnel，搞定tunnel和vni的绑定关系，再同步MAC和IP，那么就完美了，下次再写。

![img](https://pic4.zhimg.com/80/v2-4125464c3415ff16a099d7202ac2fc1b_720w.webp)

左和中间是计算节点，右是网络计算，所有节点都有vxlan encap/decap能力，是VTEP。图中有两个租户，每个租户有两个vxlan，第一个vxlan对应的IP段是192.168.1.0/24，网关是192.168.1.1，子网里有几个虚拟机，第二vxlan对应的IP段是192.168.2.0/24，网关是192.168.2.1。流量跨不跨节点，流量跨不跨vxlan，组合起来就是四种场景，再加流量出外网就是五种场景了。

- 不跨节点不跨vxlan

图中蓝色的线，本节点BD二层转发。

- 不跨节点跨vxlan

不存在，要跨vxlan一定得有三层转发，三层在网络节点上，那么一定跨节点了。

- 跨节点不跨vxlan

图中绿色的线，左和中间BD都进行二层转发。

- 跨节点跨vxlan

画中黄色的线，左和中间BD二层转发，右三层转发。

- 出外网

画中紫色的线，中间二层转发，右三层转发。

##  2.分布式网关非对称转发

集中式网关的问题就是网关压力太大，所有跨vxlan和出外网的流量都要走网关节点，即使是同一计算节点上两个虚拟机跨vxlan互通流量也要出本计算节点在网关节点上绕了一圈。把网关复制很多份，每个计算节点也放置一份，网关配置一模一样的IP和MAC，网关只对本节点的ARP做应答，对网关的ARP请求不允许从vxlan tunnel口出去，这样同一计算节点跨vxlan就在本计算节点就能搞定了。此时网络节点只保留出外网的功能，网络节点不能再配置网关的MAC和IP，得另外配置一个子网的IP，出外网的流量在计算节点网关做策略重定向到网络节点，这就是neutron vxlan dvr模式。

![img](https://pic1.zhimg.com/80/v2-7efe12c14efe5a7b8826ca6625b0bdb8_720w.webp)

- 不跨节点不跨vxlan

图中蓝色的线，BD二层转发。

- 不跨节点跨vxlan

画中红色的线，本节点两个BD二层转发，VRF三层转发。

- 跨节点不跨vxlan

图中绿色的线，左和中间BD二层转发。

- 跨节点跨vxlan

画中黄色的线，2.3和1.5通信，流量从2.3出来，BD2进行二层转发，左节点VRF进行三层转发，到了左BD1，encap BD1的vni转发到中间BD1，再到1.5，流量从1.5回来时，BD1进行二层转发，中间节点VRF进行三层转发，到了中间BD2，encap BD2的vni转发到左BD2，再到2.3。路径不对称，来回encap的vni也不一样。

- 出外网

画中紫色的线，2.5到8.8，出去时2.5到自己的网关进行VRF进行三层转发，路由重定向到网络节点，还从进来的bdif出，再回到的BD2进行vxlan encap，到了网络节点的BD2，再到网络节点VRF进行三层转发，再做NAT出去了，回来了在网络节点做NAT，VRF转发到BD2，BD2到中间节点的BD2，再到2.5，路径不对称，来回encapR vni一样，路由有点复杂。

## 3.分布式网关对称转发

非对称转发跨节点跨vxlan时流量路径不对称，不对称导致有状态的防火墙会把正常流量丢了，流量统计和QOS等功能不好找地方落地，还有一个问题是vni不对称，流量用的vni是目的vxlan的vni，流量去时和回来时vni不一样，所以出现了l3 vni，每个租户一个l3 vni，跨节点跨vxlan转发用l3 vni，走l3 vni的BD，IP VRF中虚拟机的IP用32位精确匹配，用路由控制去l2 bd还是l3 bd，这下把路径和vni不对称问题都解决了。

![img](https://pic4.zhimg.com/80/v2-a366f8d3391150fc75e140815a12b317_720w.webp)

- 不跨节点不跨vxlan

图中蓝色的线，BD二层转发。

- 不跨节点跨vxlan

画中红色的线，本节点两个BD二层转发，VRF三层转发。

- 跨节点不跨vxlan

图中绿色的线，左和中间BD二层转发。

- 跨节点跨vxlan

画中黄色的线，2.3和1.5通信，流量从2.3出来，BD2进行二层转发，左节点VRF进行三层转发，走l3 vni的BD，encap l3 vni转发到中间l3 vni BD，再到VRF进行三层转发，中间BD1进行二层转发到1.5，回来相反，每个节点进行了二层和三层转发，相比非对称转发多了一次三层转发。

- 出外网

画中紫色的线，2.5到8.8，出去时2.5到自己的网关进行VRF进行三层转发，从l3 vni出到网络节点，网络节点VRF进行三层转发，再做NAT出去了，回来了在网络节点做NAT，VRF转发从l3 vni出，到了中间节点的VRF转发回来，路径是对称。

## 4.对称和非对称比较

非对称方案VRF中路由不需要32位精确匹配，只要bdif上网关IP配置上就能自动生成路由，再额外一条defaul路由把出外网流量导引到网络节点上去，网络节点上有所有vxlan的BD，还得有vxlan中一个IP地址，和网关不同。问题是BD上MAC超级多，要有本vxlan中所有虚拟机的MAC。左和中间两个BD是二层互通的，两个BD上都有网关，网关IP和MAC一样的，那么BD上学习到的网关MAC到底在bdif上还是tunnel口呢，网关MAC出本节点BD时源mac必须替换掉成本vtep的router mac，目的MAC保持虚拟机的MAC，BD上的网关MAC就在bdif口上，对网关arp请求也不能从tunnel口出去。

对称转发需要配置l3 vni，每个节点上VRF连接l3 vni的bdif口配置的IP和MAC都不相同，这个MAC就是system mac或者说是router mac，每个vtep不同，l3 vni转发时内层mac就用这个mac，ip其实可有可没有，有好理解，一个router mac对应一个IP，路由通告时下一跳就是这个mac，l3 vni转发时是这个IP，目的MAC是这个IP对应的router mac，源MAC是本vtep的router MAC。对称转发要求VRF中要有所有虚拟机的路由，VTEP之间一定要能同步IP，还有一条出外网路由到网络节点。网络节点只要有l3 vni就可以。既然VRF中所有路由都有了，那么跨节点不跨vxlan能不能用l3 vni转发呢，其实可以的，网关开启arp代答，本节点IP不代答，tunnel口抑制arp出去，把所有跨节点流量吸引到网关上去，网关做三层转发，这样BD上MAC表就很小了，只要有网关和本BD上虚拟机MAC就行了。





原文链接：https://zhuanlan.zhihu.com/p/342476389  原文作者：惠伟