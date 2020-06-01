# Spoke machine setup on a remote site

This is a Pi3b installed with 2020-05-27-raspios-buster-lite-armhf.img
- SSH is enabled, hostname is set to "mercury"
- The machine sits on an isolated site and can reach the LAN over wireless.
- It creates an overlay network that gets bridged to the unused eth0 interface.
- Machines connected via eth0 on this "spoke" machine will get bridged to the LAN.

## Update to the latest raspios
```
pi@mercury:~ $ sudo apt update && sudo apt upgrade -y
```

## Install dhcpcd run hook scripts
```
pi@mercury:/lib/dhcpcd/dhcpcd-hooks $ ls
01-test  02-vdev_common  03-vdev_veth	   03-vdev_vxlan_uc  04-vtep_overlay	20-resolv.conf	50-ntp.conf
02-dump  03-vdev_bridge  03-vdev_vxlan_mc  04-bridge_iface   10-wpa_supplicant	30-hostname
```

## Edit dhcpcd.conf to configure the machine as a "spoke" to the extended network
```
pi@mercury:~ $ sudo cp spoke_dhcpcd.conf /etc/dhcpcd.conf
```

## Check install
```
pi@mercury:~ $ sudo reboot
```
After rebooting you will see something as this:

```
pi@mercury:~ $ iwconfig 
eth0      no wireless extensions.

lo        no wireless extensions.

wlan0     IEEE 802.11  ESSID:"xxxxx"  
          Mode:Managed  Frequency:2.412 GHz  Access Point: 34:36:3B:01:02:03   
          Bit Rate=72.2 Mb/s   Tx-Power=31 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on
          Link Quality=54/70  Signal level=-56 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:0  Invalid misc:0   Missed beacon:0

pi@mercury:~ $ ip r
default via 172.17.0.1 dev wlan0 proto dhcp src 172.17.255.219 metric 303 
172.17.0.0/16 dev wlan0 proto dhcp scope link src 172.17.255.219 metric 303 

pi@mercury:~ $ ip l
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1450 qdisc pfifo_fast master extbr state DOWN mode DEFAULT group default qlen 1000
    link/ether b8:27:eb:21:22:23 brd ff:ff:ff:ff:ff:ff
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP mode DORMANT group default qlen 1000
    link/ether b8:27:eb:31:32:33 brd ff:ff:ff:ff:ff:ff
4: extbr: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether 02:00:04:21:22:23 brd ff:ff:ff:ff:ff:ff
5: vxl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master extbr state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:00:04:21:22:23 brd ff:ff:ff:ff:ff:ff

pi@mercury:~ $ ip -d l show vxl0
5: vxl0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue master extbr state UNKNOWN mode DEFAULT group default qlen 1000
    link/ether 02:00:04:21:22:23 brd ff:ff:ff:ff:ff:ff promiscuity 1 minmtu 68 maxmtu 65535 
    vxlan id 314159 group 239.31.41.59 dev wlan0 srcport 0 0 dstport 4789 ttl auto ageing 300 udpcsum noudp6zerocsumtx noudp6zerocsumrx 
    bridge_slave state forwarding priority 32 cost 100 hairpin off guard off root_block off fastleave off learning on flood on port_id 0x8001 port_no 0x1 designated_port 32769 designated_cost 0 designated_bridge 8000.2:0:4:21:22:23 designated_root 8000.2:0:4:21:22:23 hold_timer    0.00 message_age_timer    0.00 forward_delay_timer    0.00 topology_change_ack 0 config_pending 0 proxy_arp off proxy_arp_wifi off mcast_router 1 mcast_fast_leave off mcast_flood on neigh_suppress off group_fwd_mask 0 group_fwd_mask_str 0x0 vlan_tunnel off isolated off addrgenmode eui64 numtxqueues 1 numrxqueues 1 gso_max_size 65536 gso_max_segs 65535 
```
