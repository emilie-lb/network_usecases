
# Network

# Resources

I would advice to do the openclassrooms courses on the network tcp/ip.
It cover up to the 4th ISO layer.

https://openclassrooms.com/fr/courses/857447-apprenez-le-fonctionnement-des-reseaux-tcp-ip/851033-la-creation-dinternet-le-modele-osi#/id/r-2150182

if you can't access because of a popup, disable javascript.

Also

https://monoinfinito.wordpress.com/series/setting-up-a-linux-gatewayrouter-a-guide-for-non-network-admins/
https://infoloup.no-ip.org/dns-dynamique-via-dhcp/

## Work

### Part 1: Simple network

Using virtualBox:

Create three vms.

tips: install net-tools,tcpdump,traceroute,dnsutils, on each if it is not installed.


The first one, let's call it `bastion` should have two network card one connected in nat the other on internal network.
All other vms shall have one network card connected to internal network.

All internal network cards should be configured with static address, and the address 10.0.1.0/24

`Bastion` should have ip forward activated, even after a reboot. It should have rules to forward (iptable). being able to ping every other vms and internet.
`Alice` should be able to ping all vms and internet
`Bob` should be able to ping all vms and internet

## Usefull commands

 - `tcpdump -i enp0s3`: listen on interface enp0s3 and dump the output.
 - `netstat -r`: show all kernel route table
 - `route [add|del]...`: manage kernel route
 - `iptable [-S|-L]`: show OS/network routing rules.
 - `ip a`: show current network interface data.

## Usefull file

 - `/etc/systectl.conf`
 - `/proc/sys/net/ipv4/ip_forward`
 - `/etc/network/interfaces`


### Part 2: DHCP and DNS

- `Bastion`: should be a DNS server and a DHCP server
    - isc-dhcp-server service installed, configured and launched (use a new network range of address. like 192.168.42.0/24).
    - bind9 service installed configured and launched (dns service).

- other server should get an ip from `bastion` and being able to ping each other and the internet

## Usefull commands and links

 - `/etc/default/isc-dhcp-server`
 - `/etc/dhcp/dhcpd.conf`
 - `/etc/bind9/`
 - `named-checkzone`

### Part 3: More complexe network

Every host that communicate should be able to communicate with full domain name (`alice.domain.tld`)
You can try an openwrt distribution which is a router with a WebUI.
You should duplicate the vms once all necessary packages are installed it will be faster than reinstalling the os. !Don't forget to reset the MAC Adress on the network card!

 - `Bastion` the entrance router:
   - port forwarding to: webserver1, webserver2, for ssh service.
   - has ssh server install
   - WebServer installed to forward request to `webserver[1-2]`
 - `webserver1`: webserver rendering a page with wrote `WebServer 1`
 - `webserver2`: webserver rendering a page with wrote `WebServer 2`
 - `dhcp`: dhcp server
   - has two network cards
   - should have a virtual network card for every subnetwork of router2 and act as a dhcp on these network.
 - `dns`: dns server
 - `router2`: entrance of subnetworks:
   - `admin`: can't communicate directly with subnetworks
     - `alice`: simple debian
     - `bob`: simple debian
   - `services`: network communicate with users
     - `printer`: a simple debian
     - `fax`: a simple debian
   - `users`: network communicate with service
     - `cedric`: simple debian
     - `denise`: simple debian
