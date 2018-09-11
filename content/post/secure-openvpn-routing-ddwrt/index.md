---
title: Secure OpenVPN Routing between Multiple Networks using DD-WRT
date: 2010-08-30T02:17:38+00:00
tags: [ "technical", "networking", "routing", "ddwrt", "openvpn" ]
draft: false
---
## Description

I've created a secure routed VPN network between all of my family's home networks.  Here's what it looks like, followed by how I did it.

**DD-WRT Network Diagram**
{{<figure src="ddwrt_openvpn.png" title="">}}

Here's an overview of the components:

  * Home Network 
      * OpenVPN concentrator
      * Netgear WNR3500L (480Mhz CPU, 8MB Flash, 64MB RAM)
      * DD-WRT Mega/Big (Includes OpenVPN), with jffs enabled
      * Local Network: 192.168.1.0/24
      * Dynamic DNS: xxxx.dyndns.org
  * Remote Client Networks 1 and 2 
      * OpenVPN client
      * Linksys WRT54G (266Mhz CPU, 4MB Flash, 16MB RAM)
      * DD-WRT VPN (Includes OpenVPN), with jffs enabled
      * Client Network 1 
          * Local Network: 192.168.2.0/24
          * Dynamic DNS: yyyy.dyndns.org
      * Client Network 2 
          * Local Network: 192.168.3.0/24
          * Dynamic DNS: zzzz.dyndns.org

## Generate the Certificates

First, install OpenVPN on your development machine

{{<highlight bash "">}}
sudo apt-get install openvpn
{{</highlight>}}

Second, run this little script to create a server certificate and several client certificates. These certificates will be directly used in the OpenVPN concentrator and client configuration files.  You will need a single unique client certificate for every OpenVPN client that will connect to the OpenVPN server. When running this script, accept defaults for all of the prompts.

{{<highlight bash "">}}
#!/bin/bash

mydir=`pwd`
mydate=`date +%Y%m%d%H%M`
mykeygendir="$mydir/${mydate}_keygen"

echo mykeysdir

cp -r /usr/share/doc/openvpn/examples/easy-rsa/2.0 $mykeygendir
pushd .
cd $mykeygendir
source ./vars
export KEY_COUNTRY="US"
export KEY_PROVINCE="CA"
export KEY_CITY="San Francisco"
export KEY_ORG="yourdomain.com"
export KEY_EMAIL="you@yourdomain.com"
export KEY_CN="yourdomain.com"
export KEY_OU="yourdomain.com"
export KEY_NAME="yourdomain.com"

yes "" | ./clean-all
yes "" | ./build-ca
./build-key client1
./build-key client2
./build-key client3
./build-key client4
./build-key client5
./build-key client6
./build-key client7
./build-key client8
./build-key client9
./build-key-server server
./build-dh
{{</highlight>}}

## Home Network OpenVPN Concentrator

Add the following to DD-WRT -> Administration -> Commands -> Startup

{{<highlight bash "">}}
sleep 15
cat /jffs/openvpn_server.txt | sh
{{</highlight>}}

Customize and copy the file contents under the DD-WRT flash path: /jffs/openvpn_server.txt.

{{<highlight bash "">}}
mkdir -p /tmp/etc/openvpn/ccd
echo '
#!/bin/sh

#### Set all Variables

lan_network=192.168.1.0
vpn_network=192.168.100.0

#### Create the openvpn.conf file

echo "
# Tunnel options
mode server     # Set OpenVPN major mode
proto udp       # Setup the protocol
port 1194       # TCP/UDP port number
dev tun         # TUN/TAP virtual network device
keepalive 15 60 # Simplify the expression of --ping
daemon          # Become a daemon after all initialization
verb 3          # Set output verbosity to n
comp-lzo        # Use fast LZO compression

# vpn tun subnet info
server $vpn_network 255.255.255.0
push "route $lan_network 255.255.255.0"
push "route 192.168.2.0 255.255.255.0"
push "route 192.168.3.0 255.255.255.0"
route 192.168.2.0 255.255.255.0
route 192.168.3.0 255.255.255.0

# OpenVPN server mode options
client-to-client # tells OpenVPN to internally route client-to-client traffic
duplicate-cn     # Allow multiple clients with the same common name

# TLS Mode Options
tls-server       # Enable TLS and assume server role during TLS handshake
ca ca.crt        # Certificate authority (CA) file
dh dh1024.pem    # File containing Diffie Hellman parameters
cert server.crt  # Signed certificate of local peers
key server.key   # Private key of local peers

client-config-dir /tmp/etc/openvpn/ccd/
" > openvpn.conf

echo "
iroute 192.168.2.0 255.255.255.0
" > ccd/client1

echo "
iroute 192.168.3.0 255.255.255.0
" > ccd/client2

#### Create the key files

echo "-----BEGIN CERTIFICATE-----
<PASTE CA.CRT CONTENTS HERE>
-----END CERTIFICATE-----" > ca.crt

echo "-----BEGIN RSA PRIVATE KEY-----
<PASTE SERVER.KEY CONTENTS HERE>
-----END RSA PRIVATE KEY-----" > server.key

echo "-----BEGIN CERTIFICATE-----
<PASTE SERVER.CRT CONTENTS HERE>
-----END CERTIFICATE-----" > server.crt

echo "-----BEGIN DH PARAMETERS-----
<PASTE DH1024.PEM CONTENTS HERE>
-----END DH PARAMETERS-----" > dh1024.pem

#### Set the firewall rules

iptables -I INPUT -p udp --dport 1194 -j ACCEPT
iptables -I INPUT -i tun+ -j ACCEPT
iptables -I FORWARD -i tun+ -j ACCEPT
iptables -I INPUT -s $vpn_network/24 -j ACCEPT
iptables -I FORWARD -s $vpn_network/24 -j ACCEPT
iptables -I FORWARD -s 192.168.1.0/24 -j ACCEPT
iptables -I FORWARD -s 192.168.2.0/24 -j ACCEPT
iptables -I FORWARD -s 192.168.3.0/24 -j ACCEPT

#### Start the program

# The command [openvpn --config openvpn.conf --daemon] does not work so use the following
sleep 2
ln -s /usr/sbin/openvpn /tmp/etc/openvpn/openvpn
/tmp/etc/openvpn/openvpn --config /tmp/etc/openvpn/openvpn.conf
' > /tmp/etc/openvpn/openvpn.sh
chmod a+x /tmp/etc/openvpn/openvpn.sh
cd /tmp/etc/openvpn
./openvpn.sh
{{</highlight>}}

## Remote Network OpenVPN Clients

Add the following to DD-WRT -> Administration -> Commands -> Startup

{{<highlight bash "">}}
mkdir -p /tmp/etc/openvpn
echo '
#!/bin/sh

#### Set all Variables

lan_network=192.168.2.0
vpn_network=192.168.100.0

#### Create the openvpn.conf file

echo "
# Tunnel options
proto udp       # Setup the protocol
port 1194       # TCP/UDP port number
dev tun         # TUN/TAP virtual network device
keepalive 15 60 # Simplify the expression of --ping
daemon          # Become a daemon after all initialization
verb 3          # Set output verbosity to n
comp-lzo        # Use fast LZO compression

# vpn tun subnet info
pull            # Fetch the routes from the server

# TLS Mode Options
tls-client       # Enable TLS and assume client role during TLS handshake
ca ca.crt        # Certificate authority (CA) file
cert client.crt  # Signed certificate of local peers
key client.key   # Private key of local peers

remote aaaa.dyndns.org
resolv-retry infinite
nobind
" > openvpn.conf

#### Create the key files

echo "-----BEGIN CERTIFICATE-----
<PASTE CA.CRT CONTENTS HERE>
-----END CERTIFICATE-----" > ca.crt

echo "-----BEGIN RSA PRIVATE KEY-----
<PASTE CLIENT.KEY CONTENTS HERE>
-----END RSA PRIVATE KEY-----" > client.key

echo "-----BEGIN CERTIFICATE-----
<PASTE CLIENT.CRT CONTENTS HERE>
-----END CERTIFICATE-----" > client.crt

#### Set the firewall rules

iptables -I INPUT -p udp --dport 1194 -j ACCEPT
iptables -I INPUT -i tun+ -j ACCEPT
iptables -I FORWARD -i tun+ -j ACCEPT
iptables -I INPUT -s /24 -j ACCEPT
iptables -I FORWARD -s /24 -j ACCEPT
iptables -I FORWARD -s 192.168.1.0/24 -j ACCEPT
iptables -I FORWARD -s 192.168.2.0/24 -j ACCEPT
iptables -I FORWARD -s 192.168.3.0/24 -j ACCEPT

#### Start the program

# The command [openvpn --config openvpn.conf --daemon] does not work so use the following
sleep 2
ln -s /usr/sbin/openvpn /tmp/etc/openvpn/openvpn
/tmp/etc/openvpn/openvpn --config /tmp/etc/openvpn/openvpn.conf
' > /tmp/etc/openvpn/openvpn.sh
chmod a+x /tmp/etc/openvpn/openvpn.sh
cd /tmp/etc/openvpn
./openvpn.sh
{{</highlight>}}
