# Spoke machine setup on a remote site

This is a Pi3b installed with 2020-05-27-raspios-buster-lite-armhf.img
- SSH is enabled, hostname is set to "mercury"
- The machine sits on an isolated site and can reach the LAN over wireless.
- It creates an overlay network that gets bridged to the unused eth0 interface.
- Machines connected via eth0 on this "spoke" machine will get bridged to the LAN.

## Update to the latest raspios
```
pi@sun:~ $ sudo apt update && sudo apt upgrade -y
...
```

## Install dhcpcd run hook scripts
```
pi@mercury:/lib/dhcpcd/dhcpcd-hooks $ ls
01-test  02-vdev_common  03-vdev_veth      03-vdev_vxlan_uc  04-vtep_overlay        10-wpa_supplicant  30-hostname
02-dump  03-vdev_bridge  03-vdev_vxlan_mc  04-bridge_iface   10-dhcp_relay_overlay  20-resolv.conf     50-ntp.conf
```

## Edit dhcpcd.conf to configure the machine as a "spoke" to the extended network
```
pi@mercury:~ $ sudo su
root@mercury:/home/pi# cd /etc
root@mercury:/etc# cp dhcpcd.conf dhcpcd.conf_orig

root@mercury:/etc# nano dhcpcd.conf

...
#interface eth0
#fallback static_eth0

# See VTEP config below
denyinterfaces vxl0 extbr eth0

# Wireless LAN client interface, DHCP configuration
interface wlan0

# Vxlan Tunnel End Point (VTEP) configuration
# Processed by dhcpcd run hook "04-vtep_overlay"
# Type "spoke" creates additional interface vxl0
# To allow clients on the remote site to actually reach the network, eth0 is linked into the VTEP
# See the run hook for available options
env vdev_vtep=extbr
env vtep_type=spoke
env vtep_under=wlan0
env vtep_link=eth0
```
## Check install

```
root@mercury:/etc# reboot
```
After rebooting you will see something as this

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
```
