---
title: Cisco VPN Client on DD-WRT Wireless Router
date: 2010-07-31T07:44:13+00:00
tags: [ "technical", "linux", "vpn", "ddwrt", "cisco" ]
draft: false
---

If you connect to a network served by a Cisco VPN concentrator, then you can run the Cisco VPN client on a router, instead of your computer. Running Cisco VPN on a router creates several advantages:

  * Masquerades (NAT) the local network so that all computers behind the router can access the VPN network
  * Re-connects on dropped connections.
  * Splits and sends only traffic destined to the foreign network over the VPN connection.

Requirements:

  * Router that supports DD-WRT: Netgear WNR3500L
  * DD-WRT Firmware containing vpnc: dd-wrt.v24-14311_NEWD-2_K2.6_openvpn.bin
  * Journaling Flash Filesystem (JFFS) enabled on the router

Add the following to the startup script under Administration -> Commands -> Startup

{{<highlight bash "">}}
sleep 15
cat /jffs/vpnc.txt | sh
{{</highlight>}}

Create the file /jffs/vpnc.txt on the router

{{<highlight bash "">}}
mkdir -p /tmp/etc/vpnc
echo '
#!/bin/sh 

#### Set all Variables

vpn_concentrator="vpn.company.com"  # ip or hostname of ipsec vpn concentrator
vpn_keepalive_host="10.1.1.1" # the ip or hostname of a computer, reachable if vpn is established
vpn_groupname="group_name"         # group name hereconnection
vpn_grouppasswd="group_password" # group password here
vpn_username="user_name"             # your username here
vpn_password="user_password"     # your password here

#### Create a local script to split routes

echo "
CISCO_SPLIT_INC=1
CISCO_SPLIT_INC_0_ADDR=10.0.0.0  # IP range to go into first tunnel
CISCO_SPLIT_INC_0_MASK=255.0.0.0 # Subnet Mask for first tunnel
CISCO_SPLIT_INC_0_MASKLEN=8      # Mask length
CISCO_SPLIT_INC_0_PROTOCOL=0
CISCO_SPLIT_INC_0_SPORT=0
CISCO_SPLIT_INC_0_DPORT=0
. /etc/vpnc/vpnc-script
" > /tmp/etc/vpnc/vpnc-script-local
chmod a+x /tmp/etc/vpnc/vpnc-script-local

#### Create the vpnc.conf file

echo "
IPSec gateway $vpn_concentrator
IPSec ID $vpn_groupname
IPSec secret $vpn_grouppasswd
Xauth username $vpn_username
Xauth password $vpn_password
Script /tmp/etc/vpnc/vpnc-script-local  # points to the local script
" > /tmp/etc/vpnc/vpnc.conf

#### Create the vpnc.sh file

pingtest () {
 ping -q -c1 $1 >> /dev/null
 if [ "$?" == "0" ]; then
       echo 1 #reachable 

 else
       echo 0 #not reachable
 fi
}

while [ true ]; do
  # wait until the concentrator is reachable
  while [ "`pingtest $vpn_concentrator`" != "1" ]; do
    echo "Vpnc concentrator $vpn_concentrator is not reachable, sleeping 10"
    sleep 10;
  done

  if [ "`pingtest $vpn_keepalive_host`" == "1" ]; then
    echo "vpn connection active: $vpn_keepalive_host is alive"
    sleep 300;
  else
    echo "vpn connection down: $vpn_keepalive_host is unreachable"
    vpnc-disconnect
    echo "Attempting to start vpnc"
    vpnc /tmp/etc/vpnc/vpnc.conf --dpd-idle 0
    tundev="`ifconfig |grep tun |cut -b 1-4|tail -n 1`"
    iptables -A FORWARD -o $tundev -j ACCEPT
    iptables -A FORWARD -i $tundev -j ACCEPT
    iptables -t nat -A POSTROUTING -o $tundev -j MASQUERADE
  fi
  sleep 1;
done

return 0;
' > /tmp/etc/vpnc/vpnc.sh
chmod a+x /tmp/etc/vpnc/vpnc.sh
/tmp/etc/vpnc/vpnc.sh
{{</highlight>}}