
# Network_usecases

# Resources :

- https://openclassrooms.com/fr/courses/857447-apprenez-le-fonctionnement-des-reseaux-tcp-ip/851033-la-creation-dinternet-le-modele-osi#/id/r-2150182
- https://monoinfinito.wordpress.com/series/setting-up-a-linux-gatewayrouter-a-guide-for-non-network-admins/
- https://infoloup.no-ip.org/dns-dynamique-via-dhcp/
- https://sandilands.info/sgordon/building-internal-network-virtualbox
- https://doc.ubuntu-fr.org/iptables

# Install commands :

- `sudo apt update`   
- `sudo apt upgrade `
- `sudo apt-get install -y net-tools tcpdump traceroute dnsutils ifupdown`

# Usefull commands :

 - `tcpdump -i enp0s3` listen on interface enp0s3 and dump the output.
 - `netstat -r` show all kernel route table
 - `route [add|del]...` manage kernel route
 - `iptables [-S|-L]` show OS/network routing rules.
 - `ip a` show current network interface data.

# Usefull file :

 - /etc/sysctl.conf
 - /proc/sys/net/ipv4/ip_forward
 - /etc/network/interfaces


# Part 1: Simple network with virtualBox :

- Create three vms :
- Install net-tools, tcpdump, traceroute, dnsutils and ifupdown.
- All internal network cards should be configured with static address, and the address 10.0.1.0/24



## Bastion/ Routeur :

### Network card connected in nat :

- Network card in NAT : configuration/réseau/NAT

- etc/network/interfaces: 

		auto lo 
		iface lo inet loopback
		
		auto epn0s3
		iface enpos3 inet dhcp

### Network card internal network :
- Network card internal network : 
Configuration/réseau/internal network

- /etc/network/interfaces: (à ajouter à la suite de ce fichier)  

		auto enp0s8
		iface enp0s8 inet static
		    address 10.0.1.0.1
		    netmask 255.255.255.0
		    network 10.0.1.0
		    broadcast 10.0.1.0.1

### Ajout de règles dans iptables  (et les rendre persistantes)
  - Décommenter la ligne `net.ipv4.ip_forward=1 dans etc/sysctl.conf`
  - `sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE` pour masquer toutes les actions avant enp0s3.
  - `sudo apt-get install iptables-persistent lib` pour rendre cmd iptables persistant.
  - Vérifier les rules dans /etc/iptables/rules.V4
  - Si changement des rules : faire une sauvegarde avec `sudo systemctl -p`
 
## Alice et Bob le client :

- Should be able to ping all vms and internet
- Network card internal network : configuration/réseau/internal network
- /etc/network/interfaces: (2 parties à remplir)

		 auto lo
		 iface lo inet loopback
		 
		 auto enp0s8
		 iface enp0s8 inet static
		    address 10.0.1.0.2
		    netmask 255.255.255.0
		    network 10.0.1.0
		    broadcast 10.0.1.0.1



# Part 2: DHCP and DNS

> ## Etape 1: 

## Bastion should be a DNS server and a DHCP server :

   - isc-dhcp-server service installed, configured and launched
  - bind9 service installed configured and launched (dns service).
  - Configure static adress in 192.168.42.0/24 in /etc/network/interfaces :     
	
		auto lo
	    iface lo inet loopback
	    
	    # interface coté internet n'a pas besoin d' être statique :
	    auto enp0s3
	    iface enpos3 inet dhcp
	    
	    # interface réseau interne statique :
	    iface enp0s8 inet static
	      address 192.168.42.1
	      netmask 255.255.255.0
	      network 192.168.42.0
	      broadcast 192.168.42.255

## Alice et Bob le client :

 - Should get an ip from bastion and being able to ping each other and the internet
 
    ### Modifier adresse  :

		 iface enp0s8 inet dhcp


## Renommer les machines :
   - `sudo hostnamectl set-hostname <name>`
    - `sudo vi /etc/hosts' => 127.0.1.1 <name>`


> ## Etape 2: 

## Bastion :

### Configurer DHCP:
  
  - fichier /etc/dhcp/dhcpd.conf
        
        subnet 192.168.42.0 netmask 255.255.255.0 {
	        range 192.168.42.10 192.168.42.254;
	        option routers 192.168.42.1;
	        option domain-name-servers 8.8.8.8, 8.8.4.4; 
        }
  
 - Ajouter les hosts :
 
	      host Alice {
	        hardware ethernet <MAC>;
	        fixed-address 192.168.42.2
	      }

	  ### Ajouter enp0s8 comme dhcp serveur :
   
  - dans '/etc/default/isc-dhcp-server
      `INTERFACESv4="enp0s8"`

	   ### Restart avec les bonnes modifications :
	   `sudo systemctl restart isc-dhcp-server`
    
	   ### Vérifier si ça tourne :
	   `sudo systemctl status isc-dhcp-server`

### Configurer serveur DNS :
   - installer bind9  `sudo apt-get install bind9`
    
	   ### Pour rediriger la requête vers google :
    - etc/bind/named.conf.options
   
	        forwarders {
	          8.8.8.8;
	          8.8.4.4;
	          };
          
		 ### Désactiver le DNS par défaut d'Ubuntu :
   
    - dans le fichier etc/systemd/resolved.conf
        `DNSStublisterner=no`
    - `sudo systemctl restart systemd-resolved`
    
	   ### Les modifs ont elles été prises en compte :
   
    - `sudo systemctl status systemd-resolved`
    - `sudo systemctl disable systemd-resolved`
    
	   ### Supprimer fichier simlink :
   
    - `sudo rm /etc/resolv.conf`
    - `sudo systemctl stop systemd-resolved`
    -  reboot
    
	   ### Changer le DNS utilisé :
    - dans /etc/dhcp/dhpcd.conf:
      `option domain-name-server <IP bastion>`
    - `sudo systemctl restart isc-dhcp-restart`

## Alice et Bob :

   - /etc/systemd/resolved.conf:
	   `DNS=<IP bastion>`
        `DNSStublisterner=no`
   - `sudo systemctl restart systemd-resolved`
    - `sudo systemctl status systemd-resolved`
    - `sudo systemctl disable systemd-resolved`
    - `sudo rm /etc/resolv.conf`
    - `sudo systemctl stop systemd-resolved`
    - reboot


	  ### Vérifier que le bon DNS est utilisé :
  
  - `nslookup google.com`
  - Pour Ubuntu, le serveur par défaut est 127.0.0.53:53 (aver 53 le port DNS par défaut)
  - Pour forcer requête à passer par un serveur DNS en particulier (voir si ça marche) `nslookup google.com <ip serveur DNS>`
 
