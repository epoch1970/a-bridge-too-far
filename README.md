# A Bridge Too Far
Wifi client to ethernet bridging via multicast vxlan overlay

## Rationale
 - An AP bridges ethernet network client with wireless clients
 - Machines behind a wireless client cannot be bridged unless using WDS or otherwise departing from wireless standards

This setup spans VXLAN Tunnel End Points (VTEP) between wifi clients and a machine on the ethernet LAN, next to (or colocated with) the AP. Enpdoints are bridged with ethernet on each side.

Remote machines behind a wireless client go through the tunnel and are part of the LAN.

The machine configured as bridged VTEP on the ethernet LAN side is called a "hub", machines configured as wireless VTEPs bridged to a local ethernet interface are called "spokes".

```
                                         ++ "Hub" RPi ++           ++ "Spoke" RPi ++
                                         | DHCP server |       (((++ WLAN Client   |         ++ PC#3 +-----+
                                     +---+ Bridge      |           + 192.168.1.6   |     +---+ 192.168.1.8 |
                                     |   | WLAN AP     ++))) +==============+ VTEP |     |   +-------------+
                                     |   | 192.168.1.2 |     ‖     +        Bridge +-----+
                                     |   | Bridge      +     ‖     +---------------+     |   ++ PC#4 +-----+
                 ++ Router +---+     |   | VTEP +============‖                           +---+ 192.168.1.9 |
(Internet)+-+WAN++ Firewall    ++LAN++   | 192.168.1.3 +     ‖                               +-------------+
                 | 192.168.1.1 |     |   +-------------+     ‖     ++ "Spoke" RPi ++
                 +-------------+     |                       +==============+ VTEP |     ++ PC#5 +------+
                                     |   ++ PC#2 +-----+           +        Bridge +-----+ 192.168.1.10 |
                                     +---+ 192.168.1.4 |       (((++ WLAN Client   |     +--------------+
                                     |   +-------------+           | 192.168.1.7   |
                                     |                             +---------------+
                                     |   ++ PC#1 +-----+
                                     +---+ 192.168.1.5 |
                                         +-------------+
```
 
## Description

Designed and tested to work with raspios using Raspberry Pi 3B. Built upon software packages dhcpcd, dnsmasq and hostapd.

  - This design uses the wireless network as the fabric of a network switch. Wireless performance is the overriding factor in the performance of the extended network
     - The project presents an all-in-one "hub", acting as bridged AP and bridged VTEP. Its location should be optimized so that "spokes" receive the best possible wireless signal
     - Running a separate Access Point is a possible alternative
  - Multicast vxlan was preferred to unicast as IP address endpoint management is not a concern with multicast.
  - Due to the extra headers required for the tunnel, remote machines have to set their TCP MTU size to the payload size of the tunnel, here 1450 instead of 1500.
    - To avoid configuring MTU manually on every remote machine, the DHCP server on the LAN can instruct them to set their MTU via DHCP option 26. For this the DHCP server must be able to differenciate "regular" clients (MTU 1500) from clients coming out of the tunnel (MTU 1450.)
    - The option of colocating the DHCP server with the tunnel is used here:
       - The DHCP server identifies from which network interface the request comes from and adapt its response, adding DHCP option 26 when necessary
       - Iptables rules are required to block requests from going out of the tunnel bridge interface into the LAN bridge interface, as the server might then reply to the "echo" on the LAN with a regular lease
    - Using a DHCP relay listening on the tunnel interface and adding its identification (circuit-id) is a possible alternative in order to identify the source of the request
    - Using a DHCP server in each remote site is a possible alternative
  - Testing involved 1 hub, 3 spokes with 1 or 2 client each. The AP was either running off the built-in wireless interface in a Raspberry Pi 3b, or a separate hardware AP. Testing focused on reliability and stability, mostly by playing audio synchronised across remote clients (with Logitech Media Server on the LAN)

## Project files
 - Dhcpcd config files: hub profile, spoke profile
 - Dhcpcd run hooks: management for various virtual network devices, iptables configuration

## Install pages
 - For the hub node
 - For a first spoke node (additional nodes would be identical)
