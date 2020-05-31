# A Bridge Too Far
Wifi client to ethernet bridging via multicast vxlan overlay

## Rationale
 - An AP bridges ethernet network client with wireless clients
 - Machines behind a wireless client cannot be bridged unless using WDS or departing from the WiFi standards
 
 This setup spans vxlan endpoints between wifi clients and a machine on the ethernet LAN, near (or colocated with) the AP.
 
## Description
Designed and tested to work with raspios and dhcpcd using Raspberry Pi 3B and Raspberry Pi 2B hosts.

  - Ethernet LAN, bridged AP, wireless clients are unchanged, machines communicate as usual
  - Ethernet clients behind a wireless client use a local bridged vxlan tunnel endpoint going into the bridged AP, then the ethernet bridged endpoint, and are thus bridged to the LAN
  - Multicast vxlan was preferred to unicast as IP address endpoint management is not a concern. Since the bridging VTEP on the LAN side creates a loop, iptables filtering is mandatory.
  - Due to the tunnel, client machines have to set their ethernet MTU size to the MTU size of the tunnel, typically 1450. 
    - To avoid configuring this on every machine, the DHCP server on the LAN should be able to differenciate clients coming out of the tunnel from "regular" clients, and instruct them to set their ethernet MTU to 1450.
    - To avoid configuring every host on the DHCP server, a valid option would be to colocate the vxlan endpoint with the DHCP server, make it listen on that interface and use specific options for clients that use it.
    - The option of using a DHCP relay listening on the tunnel and adding its identification is used here:
       - Iptables rules are required to block direct communication between tunneled clients and the DHCP server
       - The DHCP server needs to be configured to identify the relay's circuit-id and adapt its response to clients requests coming via the relay
       
## Project files
The bridged endpoint on the ethernet LAN side is called a "hub", the wireless endpoint bridging its ethernet interface for remote clients is called a "spoke".

 - Dhcpcd config files: hub profile, spoke profile
 - Dhcpcd run hooks: management for various virtual network devices, dhcp relay lifecycle, iptables configuration
 
 ## Install pages
  - For the "hub" node
  - For a first "spoke" node
  
