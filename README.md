# TP5-Reseau

## I. Préparation du lab

## Config et Checklist des IP des VMs.

Pour rentrer les IP Statiques on fait sur chaque VM :

`sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s3`

On arrive là: 

```
NAME=enp0s3
DEVICE=enp0s3

BOOTPROTO=static
ONBOOT=yes

IPADDR=10.5.2.10 pour client1. => (client2 aura 10.5.2.11 et server1 et aura 10.5.1.10)
NETMASK=255.255.255.0
```
Après on fait:

```
sudo ifdown enp0s3
sudo ifup enp0s3
```

**Définition du nom de domaine je l'ai fait sur les 3 VMs, ici c'est `client1`.**

`echo 'server.tp5.b1' | sudo tee /etc/hostname`

`sudo nano /etc/hosts` :
```
10.5.2.11 client2 client2.tp5.b1
10.5.1.10 server1 server1.tp5.b1
```
## Config et Checklist des IP des Routeurs
.
Le réseau qui contient les routeurs, est en /30. 
Cela suffit car il n'y à que 2 machines à l'intérieur.


**Définition des IPs statiques**

Pour `router1` on fait :
```
show ip int br
conf t
interface ethernet 0/0
ip address 10.5.1.254 255.255.255.0
no shut
exit
interface ethernet 0/3
ip address 10.5.3.1 255.255.255.252
no shut
exit
exit
```
Et pour `router2` :

```
show ip int br
conf t
interface ethernet 0/0
ip address 10.5.2.254 255.255.255.0
no shut
exit
interface ethernet 0/3
ip address 10.5.3.2 255.255.255.252
no shut
exit
exit
```
**Définition du nom de domaine**

Pour `router1` on fait :

```
conf t
hostname router1.tp5.b1
exit
```
Pour `router2` on fait :
```
conf t
hostname router2.tp5.b1
exit
```

## Checklist routes

J'ai fait tous les fichiers host en même temps que hostname.

* router1.tp5.b1
```
conf t
ip route 10.5.2.0 255.255.255.0 10.5.3.2
```
* router2.tp5.b1
```
conf t
ip route 10.5.1.0 255.255.255.0 10.5.3.2
```
* server1.tp5.b1
```
sudo nano /etc/sysconfig/network-script/route-enp0s3
(dans le fichier) 10.5.2.0/24 via 10.5.1.254 dev enp0s3
sudo systemctl restart network
```
* client1.tp5.b1
```
sudo nano /etc/sysconfig/network-script/route-enp0s3
(dans le fichier) 10.5.1.0/24 via 10.5.2.254 dev enp0s3
sudo systemctl restart network
```
* client2.tp5.b1
```
sudo nano /etc/sysconfig/network-script/route-enp0s3
(dans le fichier) 10.5.1.0/24 via 10.5.2.254 dev enp0s3
sudo systemctl restart network
```

**Depuis client1**
```
ping server1
PING server1 (10.5.1.10) 56(84) bytes of data.
64 bytes from server1 (10.5.1.10): icmp_seq=2 ttl=62 time=21.3 ms
64 bytes from server1 (10.5.1.10): icmp_seq=3 ttl=62 time=24.8 ms
```
**Depuis client2**
```
ping server1
PING server1 (10.5.1.10) 56(84) bytes of data.
64 bytes from server1 (10.5.1.10): icmp_seq=1 ttl=62 time=23.2 ms
64 bytes from server1 (10.5.1.10): icmp_seq=2 ttl=62 time=31.3 ms
```

# Mise en place du serveur DHCP

Hop j'ai renommé `client2` en `dhcp-net2.tp5.b1`

* J'ai ajouté la carte NAT dans virtual box puis démarré la machine.
* Ensuite j'ai fait `sudo yum install -y dhcp`
* J'ai éteint la VM
* J'ai relancer la VM dans GNS3.

Dans le fichier `sudo nano /etc/dhcp/dhcpd.conf` de dhcp-net2.tp5.b1 :

```
# dhcpd.conf

# option definitions common to all supported networks
option domain-name "tp5.b1";

default-lease-time 600; 
max-lease-time 7200; 

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;

# Use this to send dhcp log messages to a different log file (you also
# have to hack syslog.conf to complete the redirection).
log-facility local7;

subnet 10.5.2.0 netmask 255.255.255.0 { 
  range 10.5.2.50 10.5.2.70;
  option domain-name "tp5.b1"; 
  option routers 10.5.2.254; 
  option broadcast-address 10.5.2.255;
}
```
Ensuite on fait `sudo systemctl enable dhcpd`

Puis sur `client1` :

* IP dynamique
`sudo nano /etc/sysconfig/network-scripts/ifcfg-enp0s3`

```
NAME=enp0s3
DEVICE=enp0s3

BOOTPROTO=dhcp
ONBOOT=yes
```
`sudo ifdown enp0s3 `et` sudo ifup enp0s3`

**Résultat** de `ip a`
```
enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:35:66:94 brd ff:ff:ff:ff:ff:ff
    inet 10.5.2.50/24 brd 10.5.2.255 scope global noprefixroute dynamic enp0s3
       valid_lft 598sec preferred_lft 598sec
    inet6 fe80::a00:27ff:fe35:6694/64 scope link
       valid_lft forever preferred_lft forever
```
* `dhclient`
```
sudo dhclient -v -r

ip a

enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:35:66:94 brd ff:ff:ff:ff:ff:ff
    inet 10.5.2.51/24 brd 10.5.2.255 scope global dynamic enp0s3
       valid_lft 598sec preferred_lft 598sec
    inet6 fe80::a00:27ff:fe35:6694/64 scope link
    
 ```
 
* Capture du Dora sur wireshark

![screen du dora](https://github.com/Sascha40/TP5-Reseau/blob/master/sreen%20dora.png)

_ _ _
J'ai manqué de temps pour faire la bonus etc. Désolé 😕
