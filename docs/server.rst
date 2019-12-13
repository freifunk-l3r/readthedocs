Gateway setup
=============

Gateways connect the mesh by allowing access to a local VPN. For Babel
networks, the wireguard VPN is recommended. Fastd is still possible albeit
slower.

The gateway setup is described in docker files that will be run with
host-network mode. The following services are recommended:

Wireguard/babel/mmfd/l3roamd
----------------------------

The docker image klausdieter371/wG-docker which is hosted on
https://github.com/christf/wg-docker.git provides the essential mesh and vpn
services. The container can be started using the compose file.
It requires a bit of environment:

network
~~~~~~~

The address in the environment variable OWNIP must be assigned to one of the network devices of the gateway. It is possible to use an ifup script for that:

.. code:: bash

 #!/bin/bash
 ip -6 r d default
 ip -6 r a default via fe80::1 dev eth0 src 2a01:4f8:1c1c:71b5::1
 
 # lookup clat prefix in freifunk routing table
 ip -6 ru a to fdff:ffff:ffff::/48 lookup 10
 ip -6 ru a to fdff:ffff:fffe::/48 lookup 10
 
 # reach the rest of the batman network
 ip -6 r a fda9:26e:5805::/64 dev backend-gw2 proto static
 
 ip -6 a a fda9:26e:5805:bab1:aaaa::1/64 dev eth0
 ip -6 r a fda9:26e:5805::2 dev backend-gw2 proto static t 12
 ip -6 r a fda9:26e:5805::2 dev backend-gw2 proto static t 10
 ip -6 r a 2000::/3 from fda9:26e:5805::/48 dev backend-gw2 proto static t 10
 ip -6 r a 2000::/3 from fda9:26e:5805::/48 dev backend-gw2 proto static t 12
 ip -6 r a fda9:26e:5805::/48 dev backend-gw2 proto static t 10
 ip -6 r a fda9:26e:5805::/48 dev backend-gw2 proto static t 12
 ip6tables -I INPUT 1 -i babel-wg-+ -s fe80::/64  -p udp -m udp --dport 6696  -j ACCEPT
 ip6tables -I INPUT 1 -i babel-wg-+ -s fe80::/64  -p udp -m udp --dport 27275  -j ACCEPT
 ip6tables -I INPUT 1 -i babel-wg-+ -s fda9:026e:5805:bab1::/64  -p udp -m udp --dport 6696  -j ACCEPT
 ip6tables -I INPUT 1 -i babel-wg-+ -s fda9:026e:5805:bab1::/64  -p udp -m udp --dport 27275  -j ACCEPT
 ip6tables -I INPUT 1 -i babel-wg-+ -p udp -m udp --dport 5523  -j ACCEPT
 ip6tables -t mangle -A FORWARD -o babel-wg-+ -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
 iptables -t mangle -A FORWARD -o babel-wg-+ -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
 ip6tables -t mangle -A OUTPUT -o babel-wg-+ -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
 iptables -t mangle -A OUTPUT -o babel-wg-+ -p tcp -m tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
 exit 0


Configuration of the container via environment-file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can use the docker-compose file to start the container

.. code:: bash
 
  version: "2.1"
 services:
   wg:
     image: klausdieter371/wg-docker
     network_mode: "host"
     privileged: true
     cap_add: 
       - NET_ADMIN
     devices: 
       - /dev/net/tun:/dev/net/tun
     sysctls:
       - net.ipv6.conf.all.forwarding=1
       - net.ipv6.conf.all.accept_redirects=0
       - net.ipv4.conf.all.rp_filter=0
     restart: unless-stopped
     env_file: /root/wg-docker/wg-docker-env

The following kernel modules must be loaded - it is suggested to place this content in /etc/modules-load.d/wg.conf

.. code:: bash

 ip6table_filter
 ip6_tables
 wireguard

The Container is parametrized by the below environment file:

.. code:: bash

 DEBUG=
 
 WGSECRET=abcdefghi
 HOST_IP=
 APT_PROXY_PORT=
 
 # this is the whole net-wide prefix
 WHOLENET=fda9:26e:5805::/48 
 
 # the prefix for infrastructure. This must be in $WHOLENET
 NODEPREFIX=fda9:26e:5805:bab1::/64
 
 # the prefix in which the clients pick their address. This must be in $WHOLENET
 CLIENTPREFIX=fda9:26e:5805:bab0::/64 
 
 # the next-node address according to site.conf
 NEXTNODE=fda9:26e:5805:bab0::1
 
 # assign this address to one of the docker hosts nic 
 OWNIP=fda9:26e:5805:bab1:aaaa::1
 
 
 # tcp port where the broker listens for connections
 BROKERPORT=40000       
 
 # start of udp portrange accepting wireguard connections
 STARTPORT=40000
 
 # end of udp portrange accepting wireguard connections
 ENDPORT=41000
 
 # control port of local babeld
 BABELPORT=33123
 
 #MTU of the VPN in this network
 MTU=1374
 
 # wireguard command
 WG=/usr/bin/wg
 
 # accepting inbound wireguard connections on this interface
 WAN=eth0
 
 # allow MAXCONNECTIONS concurrent vpn connections
 MAXCONNECTIONS=150
 
 # this file contains the secret key
 PRIVATEKEY=/etc/wg-broker/secret
 
 # this is the l3roamd socket
 L3ROAMDSOCK=/var/run/l3roamd.sock
 MMFDSOCK=/var/run/mmfd.sock
 MESHIFS="backend-bab1 backend-bab2"

Firewall
--------

There is no setup script for a firewall yet. Make sure the required services are allowed to use for the network zones as appropriate:

* mmfd
* l3roamd
* babeld
* dns64
* gre


DNS64 -- klausdieter371/docker-dns64
------------------------------------

The docker image klausdieter371/docker-dns64 enables a named as dns64 server

Network setup
~~~~~~~~~~~~~

/etc/network/if-up.d/docker-dns64 is a good place to set up the ip address on which the server will listen for dns queries:

.. code:: bash

 #!/bin/bash
 ip -6 a a fda9:26e:5805:bab1:53::1/64 dev eth0

Configuration of the container via environment file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: bash

 # listen on this ipv6 IP address. this is the address from the domain configuration in the firmware.
 DNS64_IP6_LISTEN=fda9:26e:5805:bab1:53::1
 # no need to listen on a given ipv4 ip
 DNS64_LISTEN=127.0.0.1
 # clients within this range may use named
 CLIENT_ACL=fda9:26e:5805::/48
 # this depends on your jool configuration - use the same prefix.
 DNS64_PREFIX=64:ff9b::/96


Compose-File
~~~~~~~~~~~~

Use the compose file from   https://github.com/christf/docker-dns64.git

.. code:: bash

 Version: "2.1"
 services:
   dns64:
     image: klausdieter371/docker-dns64
     network_mode: "host"
     restart: unless-stopped
     env_file: /root/docker-dns64/dns64-env


NAT64
-----

... this is the place were the setup will be documented once it is dockerized ...



exit
----

There are two deployment schemes possible: with NAT6 or without. Freifunk
Magdeburg uses the former, Freifunk Frankfurt the latter. If NAT is to be used,
it is recommended to do this on the gateway itself, not on the exit.


GRE
---

* load nf_conntrack_proto_gre such that gre streams are recognized as valid packets - and place the module in /etc/modules-load.d/gre.conf
* use the configuration for a gre tunnel in /etc/network/interfaces.d

