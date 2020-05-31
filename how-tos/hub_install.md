# Hub machine setup on the LAN

This machine sits on the LAN, where a wireless AP and a DHCP server serve clients.
 - This is a Pi2 installed with 2020-05-27-raspios-buster-lite-armhf.img
 - SSH is enabled, hostname is set to "sun"
 - It creates an overlay network bridged to the LAN
 - Machines from remote "spoke" sites will reach the hub (over wireless) and get bridged to the LAN.

## Update to the latest raspios
```
pi@sun:~ $ sudo apt update && sudo apt upgrade -y
...
```

## Install a DHCP relay agent
```
pi@sun:~ $ sudo DEBIAN_FRONTEND=noninteractive apt install -y isc-dhcp-relay
...
The following NEW packages will be installed:
  isc-dhcp-relay libirs-export161 libisccfg-export163
0 upgraded, 3 newly installed, 0 to remove and 0 not upgraded.
...
```
Disable the service as it is run dynamically by dhcpcd:
```
pi@sun:~ $ sudo systemctl disable isc-dhcp-relay
isc-dhcp-relay.service is not a native service, redirecting to systemd-sysv-install.
Executing: /lib/systemd/systemd-sysv-install disable isc-dhcp-relay
```

## Install dhcpcd run hook scripts
```
pi@sun:/lib/dhcpcd/dhcpcd-hooks $ ls
01-test  02-vdev_common  03-vdev_veth      03-vdev_vxlan_uc  04-vtep_overlay        10-wpa_supplicant  30-hostname
02-dump  03-vdev_bridge  03-vdev_vxlan_mc  04-bridge_iface   10-dhcp_relay_overlay  20-resolv.conf     50-ntp.conf
```

## Edit dhcpcd.conf to configure the machine as the "hub" to the extended network
```
pi@sun:~ $ sudo su
root@sun:/home/pi# cd /etc
root@sun:/etc# cp dhcpcd.conf dhcpcd.conf_orig

root@sun:/etc# nano dhcpcd.conf
...
#interface eth0
#fallback static_eth0

# See VTEP config below
denyinterfaces vxl0 _lnk_*

# DHCP relay on the hub for automagic client MTU setup - processed by run hook "10-dhcp_relay_overlay"
# If you want to disable the dhcp relay, enable the following
#nohook dhcp_relay_overlay
# Enter the IPv4 address of the DHCP server to relay to 
env dhcp_server_ipv4=172.17.0.2

# Bridges to manage - processed by run hook "03-vdev_bridge"
# See the run hook for available options
env vdev_bridges=lanbr

# Vxlan Tunnel End Point (VTEP) configuration
# Processed by dhcpcd run hook "04-vtep_overlay"
# Type "hub" creates additional interfaces vxl0 _lnk_0 _lnk_1
# See the run hook for available options
env vdev_vtep=extbr
env vtep_type=hub
env vtep_under=lanbr

# LAN bridge device lanbr, DHCP configuration
interface lanbr

# VTEP bridge device extbr, static IP address required to relay remote DHCP
# client requests to the DHCP server on the LAN.
# Use a free IP address in the LAN network. Must not take priority over the 
# LAN bridge, set a high metric FIXME CHECK THAT
# Processed by dhcpcd run hook "04-bridge_iface"
interface extbr
metric 314159
static ip_address=172.17.254.254/16

# No IP configuration for bridge member interfaces
# Processed by dhcpcd run hook "04-bridge_iface"
interface eth0
env parent_bridge=lanbr
noipv4
noipv6
noipv4ll
```
## Check install
```
root@sun:/etc# reboot
```
After rebooting you will see something as this
```
pi@sun:~ $ ip r
default via 172.17.0.1 dev lanbr proto dhcp src 172.17.255.10 metric 203 
172.17.0.0/16 dev lanbr proto dhcp scope link src 172.17.255.10 metric 203 
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

pi@sun:~ $ ps waxu | grep -e ^root.*dhcrelay
root       643  0.0  0.5   8212  5152 ?        Ss   14:38   0:00 /usr/sbin/dhcrelay -pf /var/run/vtep_dhcprelay.pid -q -a -D -iu lanbr -id extbr 172.17.0.2

pi@sun:~ $ sudo iptables -L -n -v
Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
   45 14598 DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in _lnk_0 udp dpts:67:68

Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
89461   23M DROP       udp  --  *      extbr   0.0.0.0/0            239.31.41.59         PKTTYPE = multicast
  480  169K DROP       udp  --  *      *       0.0.0.0/0            0.0.0.0/0            PHYSDEV match --physdev-in _lnk_1 udp dpts:67:68

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         
```

# Setup the main DHCP server to work with the relay

In this case dnsmasq runs the DHCP service On the LAN.

This is the extra file I dropped in the dnsmasq configuration directory to make it aware of the presence of the DHCP relay, try and force the appropriate MTU setting for clients in satellite sites.

YMMV according to your DHCP server. The circuit-id presented by dhcrelay is the incoming lease request interface name, i.e. the VTEP interface "extbr" configured in the hub machine.
```
me@router:/etc/dnsmasq.d# cat abtf.conf 
# Overlay network support
# Clients in spoke sites must set their interface to MTU 1450 (native MTU - 50)
# Lease requests transmitted by the relay on the hub node include the name of the receiving interface.
# These options map the interface name (extbr) to a dnsmasq tag, and ask clients to set their interface MTU.
# Some clients do not comply, force the value locally. (Or force all partners to use MTU 1450) 

# Lease request relayed from interface "extbr" -> tag
dhcp-circuitid=set:spoke,extbr
# Tag -> ask client to configure its ethernet MTU (DHCP option 26)
dhcp-option-force=tag:spoke,26,1450
```
