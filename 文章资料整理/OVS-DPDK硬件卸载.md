# OVS-DPDK硬件卸载

## 1.**安装网卡启动，指定ovs-dpdk**

\# yum -y install createrepo

\#yum install kernel-rpm-macros python36-devel

\#yum install unbound



\#./mlnxofedinstall --ovs-dpdk --add-kernel-support

Note: This program will create MLNX_OFED_LINUX TGZ for rhel8.5 under /tmp/MLNX_OFED_LINUX-5.5-1.0.3.2-4.18.0-348.2.1.el8_5.x86_64 directory.

See log file /tmp/MLNX_OFED_LINUX-5.5-1.0.3.2-4.18.0-348.2.1.el8_5.x86_64/mlnx_iso.13511_logs/mlnx_ofed_iso.13511.log



Checking if all needed packages are installed...

Building MLNX_OFED_LINUX RPMS . Please wait...





\# cat /etc/redhat-release

CentOS Linux release 8.5.2111



## 2.安装OVS：

\#sudo yum install -y epel-release

\#sudo yum install -y centos-release-openstack-train

\#sudo yum install openvswitch libibverbs

\#yum clean packages

\#sudo yum install openvswitch libibverbs

\#systemctl start openvswitch.service

\#systemctl enable openvswitch.service

Created symlink /etc/systemd/system/multi-user.target.wants/openvswitch.service → /usr/lib/systemd/system/openvswitch.service.





\#ovs-vsctl show

98002087-4bb7-4d70-8ea0-64335a82f9bb 

ovs_version: "2.12.0"



## 3.**配置大页面：**

\# echo "vm.nr_hugepages = 2048" >> /etc/sysctl.conf; sysctl -p

vm.nr_hugepages = 2048



\# hugeadm --page-sizes

2097152





## 4.配置SR-IOV：

\# ibdev2netdev -v

0000:5e:00.0 mlx5_0 (MT4121 - MCX556A-EDAT) CX556A - ConnectX-5 QSFP28 fw 16.32.1010 port 1 (ACTIVE) ==> ens4f0 (Up)

0000:5e:00.1 mlx5_1 (MT4121 - MCX556A-EDAT) CX556A - ConnectX-5 QSFP28 fw 16.32.1010 port 1 (ACTIVE) ==> ens4f1 (Up)



\# mlxconfig -d /dev/mst/mt4121_pciconf0 set SRIOV_EN=1 NUM_OF_VFS=4



Device #1:

\----------



Device type:  ConnectX5

Name:      MCX556A-EDA_Ax_Bx

Description:  ConnectX-5 Ex VPI adapter card; EDR IB (100Gb/s) and 100GbE; dual-port QSFP28; PCIe4.0 x16; tall bracket; ROHS R6

Device:     /dev/mst/mt4121_pciconf0



Configurations:               Next Boot    New

​     SRIOV_EN              True(1)     True(1)

​     NUM_OF_VFS             4        4



 Apply new Configuration? (y/n) [n] : y

Applying... Done!

-I- Please reboot machine to load new configurations.



### 4.1重启系统使其生效。

\# reboot

\#echo1 > /sys/class/infiniband/mlx5_0/device/mlx5_num_vfs

 

### 4.2配置switchd模式：

\#ibdev2netdev-v

  0000:5e:00.2mlx5_2 (MT4122 - NA) fw 16.32.1010 port1 (ACTIVE) ==> ens4f0v0

 

\#echo0000:5e:00.2 >/sys/bus/pci/drivers/mlx5_core/unbind

\#echoswitchdev > /sys/class/net/ens4f0/compat/devlink/mode

\#echo0000:5e:00.2 > /sys/bus/pci/drivers/mlx5_core/bind

\#systemctl start openvswitch

\#ovs-vsctl list o

![图片](https://mmbiz.qpic.cn/mmbiz_png/akGXyic486nXkUCibXINcqozcKmLIVyjqcI7W8LibLAX3GicuDcX1j2IYg9w3Syv3KQLibkiaW3F0qicpWiawCeJibtsJsw/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

```
#ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-init=true#ovs-vsctl set Open_vSwitch . other_config:hw-offload=true#ovs-vsctl --no-wait set Open_vSwitch . other_config:dpdk-extra="-w 0000:5e:00.0,representor=[0],dv_flow_en=1,dv_esw_en=1,dv_xmeta_en=1"#systemctl restart openvswitch#ovs-vsctl --no-wait add-br br0-ovs -- set bridge br0-ovs datapath_type=netdev#ovs-vsctl add-port br0-ovs pf -- set Interface pf type=dpdk options:dpdk-devargs=0000:5e:00.0#ovs-vsctl add-port br0-ovs representor -- set Interface representor type=dpdk options:dpdk-devargs=0000:5e:00.0,,representor=[0]#ovs-vsctl show
```

![图片](https://mmbiz.qpic.cn/mmbiz_png/akGXyic486nXkUCibXINcqozcKmLIVyjqc42THnraWyd7niavcqKzlC6MiaAvgUDUE5IDnRtFO6Nm9Zb4OXCptqXKg/640?wx_fmt=png&wxfrom=5&wx_lazy=1&wx_co=1)

### 4.3确认流表被卸载：

```
# ovs-appctl dpctl/dump-flows --names -mflow-dump from pmd on cpu core: 6ufid:c97dba5b-125a-4e8b-a492-7824ac91b3ad, skb_priority(0/0),skb_mark(0/0),ct_state(0/0),ct_zone(0/0),ct_mark(0/0),ct_label(0/0),recirc_id(0),dp_hash(0/0),in_port(pf),packet_type(ns=0,id=0),eth(src=d2:5c:7c:2a:85:bd,dst=8e:db:1a:),ipv4(src=10.7.159.78/0.0.0.0,dst=10.7.159.71/0.0.0.0,proto=6/0,tos=0/0,ttl=64/0,frag=no),tcp(src=5001/0,dst=46302/0),tcp_flags(0/0), packets:9702614, bytes:656316736, used:0.683s, flags:S., offloaded:yes, dp:dpdk, actions:repreiflow_bits(5,1)ufid:389612ea-dc10-401c-9ee8-fe2ecc88ed80, skb_priority(0/0),skb_mark(0/0),ct_state(0/0),ct_zone(0/0),ct_mark(0/0),ct_label(0/0),recirc_id(0),dp_hash(0/0),in_port(pf),packet_type(ns=0,id=0),eth(src=1c:34:da:37:0e:10,dst=01:80:c2:), packets:0, bytes:0, used:never, offloaded:yes, dp:dpdk, actions:drop, dp-extra-info:miniflow_bits(5,0)ufid:8d9d99ea-88c4-43a5-a0ff-ba0f3c200425, skb_priority(0/0),skb_mark(0/0),ct_state(0/0),ct_zone(0/0),ct_mark(0/0),ct_label(0/0),recirc_id(0),dp_hash(0/0),in_port(representor),packet_type(ns=0,id=0),eth(src=8e:db:1a:34:fb:2c,dst=pe(0x0800),ipv4(src=10.7.159.71/0.0.0.0,dst=10.7.159.78/0.0.0.0,proto=6/0,tos=0/0,ttl=64/0,frag=no),tcp(src=46302/0,dst=5001/0),tcp_flags(0/0), packets:616883157, bytes:933555140338, used:0.683s, flags:SP., offloaded:yes, dp:dpdkfo:miniflow_bits(5,1)
```

参考文献:

https://www.golinuxcloud.com/configure-hugepages-vm-nr-hugepages-red-hat-7/
原文链接：https://mp.weixin.qq.com/s/hFq2cnDbnQxutzmNrJEsgA
原文作者：魏新宇