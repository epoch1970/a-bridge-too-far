# Hub machine setup on the LAN

This machine sits on the LAN, where a wireless AP and a DHCP server serve clients.
 - This is a Pi2 installed with 2020-05-27-raspios-buster-lite-armhf.img
 - SSH is enabled, hostname is set to "sun"
 - It creates an overlay network bridged to the LAN
 - Machines from remote "spoke" sites will reach the hub (over wireless) and get bridged to the LAN.

## Update to the latest raspios
```
pi@sun:~ $ sudo apt update && sudo apt upgrade -y
```
## Install dnsmasq to run DHCP and DNS services for the LAN on the hub node

TODO

## Install dhcpcd run hook scripts
```
pi@sun:/lib/dhcpcd/dhcpcd-hooks $ ls
01-test  02-vdev_common  03-vdev_veth	   03-vdev_vxlan_uc  04-vtep_overlay	20-resolv.conf	50-ntp.conf
02-dump  03-vdev_bridge  03-vdev_vxlan_mc  04-bridge_iface   10-wpa_supplicant	30-hostname
```

## Edit dhcpcd.conf to configure the machine as the "hub" to the extended network
```
pi@sun:~ $ sudo cp hub_dhcpcd.conf /etc/dhcpcd.conf
```

## Check install
```
pi@sun:~ $ sudo reboot
```
After rebooting you will see something as this:
```
pi@sun:~ $ ip r
default via 172.17.0.1 dev lanbr src 172.17.0.2 metric 204 
172.17.0.0/16 dev lanbr proto dhcp scope link src 172.17.0.2 metric 204 
172.17.0.0/16 dev extbr proto dhcp scope link src 172.17.254.254 metric 314159 

pi@sun:~ $ ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast master lanbr state UP mode DEFAULT group default qlen 1000
    link/ether b8:27:eb:01:02:03 brd ff:ff:ff:ff:ff:ff
3: lanbr: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether b8:27:eb:01:02:03 brd ff:ff:ff:ff:ff:ff
4: extbr: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 02:00:04:01:02:03 brd ff:ff:ff:ff:ff:ff
5: vxl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master extbr state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:00:04:01:02:03 brd ff:ff:ff:ff:ff:ff
6: _lnk_1@_lnk_0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master extbr state UP mode DEFAULT group default qlen 1000
    link/ether fe:ff:f9:01:02:03 brd ff:ff:ff:ff:ff:ff
7: _lnk_0@_lnk_1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master lanbr state UP mode DEFAULT group default qlen 1000
    link/ether fe:ff:fa:01:02:03 brd ff:ff:ff:ff:ff:ff

pi@sun:~ $ ip -d l show vxl0
5: vxl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master extbr state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:00:04:01:02:03 brd ff:ff:ff:ff:ff:ff promiscuity 1 minmtu 68 maxmtu 65535 
    vxlan id 314159 group 239.31.41.59 dev lanbr srcport 0 0 dstport 4789 ttl auto ageing 300 udpcsum noudp6zerocsumtx noudp6zerocsumrx 
    bridge_slave state forwarding priority 32 cost 100 hairpin off guard off root_block off fastleave off learning on flood on port_id 0x8001 port_no 0x1 designated_port 32769 designated_cost 0 designated_bridge 8000.2:0:5:1:2:3 designated_root 8000.2:0:5:1:2:3 hold_timer    0.00 message_age_timer    0.00 forward_delay_timer    0.00 topology_change_ack 0 config_pending 0 proxy_arp off proxy_arp_wifi off mcast_router 1 mcast_fast_leave off mcast_flood off neigh_suppress off group_fwd_mask 0 group_fwd_mask_str 0x0 vlan_tunnel off isolated off addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 

pi@sun:~ $ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    5  1712 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in _lnk_1 udp dpts:67:68

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   11  3702 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in _lnk_1 udp dpts:67:68
    9  2952 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-out _lnk_1 udp dpts:67:68

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
```

# Setup the main DHCP server to serve remote clients

In this case dnsmasq runs the DHCP service on the LAN. 

This is the extra file I dropped in the dnsmasq configuration directory to make it aware of the tunnel bridge interface:

```
me@router:/etc/dnsmasq.d# cat abtf.conf 
# Overlay network support
# Clients in spoke sites must set their interface to MTU 1450 (native MTU - 50)
# These options make dnsmasq also listen on the tunnel interface ...
interface=extbr
# ... and ask clients to set their interface MTU.
dhcp-option-force=tag:extbr,26,1450
# Some clients do not comply, force the value locally in this case. (Or force all partners to use MTU 1450) 
```
