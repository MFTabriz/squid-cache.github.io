##master-page:CategoryTemplate
#format wiki
#language en

= WCCP 2 with TPROXY on Ubuntu 12.04 =
by ''Eliezer Croitoru''

<<Include(ConfigExamples, , from="^## warning begin", to="^## warning end")>>

<<TableOfContents>>

== What is WCCP? ==
WCCP stands for [[http://en.wikipedia.org/wiki/Web_Cache_Communication_Protocol|"Web Cache Communication Protocol"]]

What is good about WCCP?
WCCP allows separation of duties between the network and the application and there for Auto redundency.

the router has couple junctions that it can intercept on routing level dynamicly packets.
on every interface\vlan there is a "IN" and "OUT".
IN stands for incoming packets and OUT stands for OUTGOING packets.
the WCCP daemon on the cisco router gets information about the Cache supplier and service.
then on the cisco router we can define ACLs to apply the service on besides the Cache settings supplied by the cache.

the Cache supplier can interact in two ways with cisco devices:
GRE tunnel and Layer 2 SWITCHING forwarding.
when used with a GRE tunnel all the traffic that comes and goes to the client are transfered to the proxy on the GRE tunnel instead

the cisco router forwards packets to "hijack" encapsulated in the gre tunnel to the proxy.
(the proxy should know what to do with these packets.)
 the proxy do what ever it wants with the session.

on regular intercept\nat proxy the request will be requested from the origin server using it's own ip on the regular interface.
so the acls that will be neede to apply on the cisco are:
"capture only these specific ip and ports"
but on tproxy mode since the IP of the client is spoofed if we will apllly these same ACLs we will end up with an endless loop.
so instead of applying regulare WCCP ACLs we are applying another ACL built in WCCP and this is the EXLUDE.

the EXCLUDE applies only on Interface (or vlan interface) so we need to separte the traffic of the clients and the proxy.
in our case we use another interface.
on the router we use interface f1/0 for clients, f1/0 for the proxy and  f0/0 to the internet.

we apply the intercepting acls in the f0/0 interface so any port 80 destination will be intercepted.




== Outline ==

Steps to config squid in TPROXY mode with WCCP v2.
These steps are for setting [[Squid-3.1]] with [[Features/Tproxy4|TPROXYv4]], IP spoofing and Cisco WCCP. 

they apply to Ubuntu 12.04 LTS manually and not with automatic network setup of Ubuntu "/etc/network/interfaces"  file.
since i have seen it is not explained in a User Friendly way until now i decided to write it down.

it is based on [[http://bloggik.net/index.php/articles/networks/18-cisco/38-squid-tproxy-wccp|this guy which his name i dont know Russian tutorial]]



== Basic assumptions on you ==
You know the difference between TPROXY and intercept mode of squid.

you do know basic Networking and cisco cli basics.

you do know what a GRE tunnel is.

== Toplogy ==
Topology at: [[http://www1.ngtech.co.il/squid/wccp2.svg|svg]] 

{{attachment:wccp2_f.png}}

== Steps ==
Requirements on ubuntu:
basic ubuntu server ships with iptunnel iprourte2 and all iptables modules needed for the task.


{{{
#!highlight bash
#!/usr/bin/bash

echo "Loading modules.."
modprobe -a nf_tproxy_core xt_TPROXY xt_socket xt_mark ip_gre gre


LOCALIP="10.80.2.2"
CISCODIRIP="10.80.2.1"
CISCOIPID="192.168.10.127"

echo "changing routing and reverse path stuff.."
echo 0 > /proc/sys/net/ipv4/conf/lo/rp_filter
echo 1 > /proc/sys/net/ipv4/ip_forward

echo "creating tunnel..."
iptunnel add wccp0 mode gre remote $CISCOIPID local $LOCALIP dev eth1
ifconfig wccp0 127.0.1.1/32 up

echo "creating routing table for tproxy..."
ip rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100

echo "creating iptables tproxy rules..."
iptables -A INPUT  -i lo -j ACCEPT
iptables -A INPUT  -p icmp -m icmp --icmp-type any -j ACCEPT
iptables -A FORWARD -i lo -j ACCEPT
iptables -A INPUT  -s $CISCODIRIP -p udp -m udp --dport 2048 -j ACCEPT
iptables -A INPUT -i wccp0 -j ACCEPT
iptables -A INPUT -p gre -j ACCEPT

iptables -t mangle -F
iptables -t mangle -A PREROUTING -d $LOCALIP -j ACCEPT
iptables -t mangle -N DIVERT
iptables -t mangle -A DIVERT -j MARK --set-mark 1
iptables -t mangle -A DIVERT -j ACCEPT
iptables -t mangle -A PREROUTING -p tcp -m socket -j DIVERT
iptables -t mangle -A PREROUTING -p tcp --dport 80 -j TPROXY --tproxy-mark 0x1/0x1 --on-port 3129



}}}

add into squid.conf the next lines:
{{{
#add change the src subnet to the list of clients subnets allowed.
acl clients src 10.80.0.0/16

http_access allow clients

http_port 127.0.0.1:3128 
http_port 3129 tproxy

# replace 10.80.2.1 with your cisco router directly connected interface
wccp2_router 10.80.2.1
wccp_version 2
wccp2_rebuild_wait on
wccp2_forwarding_method 1
wccp2_return_method 1
wccp2_service standard 0
wccp2_service dynamic 80
wccp2_service dynamic 90
wccp2_service_info 80 protocol=tcp flags=src_ip_hash priority=240 ports=80
wccp2_service_info 90 protocol=tcp flags=dst_ip_hash,ports_source priority=240 ports=80


}}}

=== Cisco ios settings ===
{{{
conf t

ip access-list extended wccp
 permit ip 10.80.3.0 0.0.0.255 any
ip access-list extended wccp_to_inside
 permit ip any 10.80.3.0 0.0.0.255
exit
ip wccp version 2
ip wccp web-cache
ip wccp 80 redirect-list wccp
ip wccp 90 redirect-list wccp_to_inside

interface FastEthernet0/0
 ip wccp 80 redirect out
 ip wccp 90 redirect in

interface FastEthernet0/1
 ip wccp redirect exclude in
}}}


=== Preparation ===
we will prepare the Cisco before anything else:
conf t
ip access-list extended wccp
permit ip host 10.26.9.171 any
permit ip 10.26.5.0 0.0.0.255 any
permit ip 10.26.7.0 0.0.0.255 any
...
exit
ip access-list extended wccp_to_inside
permit ip any host 10.26.9.171
permit ip any 10.26.5.0 0.0.0.255
permit ip any 10.26.7.0 0.0.0.255
exit
???????? ????????, ??? ????? ????? ??????, ?? ?????? ?????? ?????????? ??????? ?? ????? ????????????? ????? ???????????????? ?? Squid, ? ?????? ?????? ?????????? ?????? ?? ???????? ???? ??? ????? ????????????? ????? ??? ?? ???????????????? ?? Squid.
????? ???????? wccp:
ip wccp 80 redirect-list wccp
ip wccp 90 redirect-list wccp_to_inside
?????? ????? ???????? ??????????????? ?? ?????? ???????????. ? ????? ?????? (??. ??????? ????) ?? ?????????? f0/2:
interface f0/2
ip wccp 80 redirect out
ip wccp 90 redirect in
? ?? ?????????? f0/1:
interface f0/1
ip wccp redirect exclude in




=== Routing Configuration ===

As per the [[Features/Tproxy4|TPROXYv4]] regular configuration:
{{{
ip rule add fwmark 1 lookup 100
ip route add local 0.0.0.0/0 dev lo table 100
}}}

=== iptables ===

=== WCCP Configuration ===

 * WCCP related iptables rules need to be created next...this and further steps are only needed if L4 WCCPv2 is used with a router, and not L2 WCCP with a switch.

{{{
iptables -A INPUT -i gre0 -j ACCEPT

iptables -A INPUT -p gre -j ACCEPT
}}}
 * For the WCCP udp traffic that is not in a gre tunnel:

{{{
iptables -A RH-Firewall-1-INPUT -s 10.48.33.2/32 -p udp -m udp --dport 2048 -j ACCEPT
}}}
|| {i} '''note:''' || When running '''iptables''' commands, you my find that you have no firewall rules at all. In this case you will need to create an input chain to add some of the rules to. I created a chain called '''LocalFW''' instead (see below) and added the final WCCP rule to that chain. The other rules stay as they are. To do this, learn iptables...or something *LIKE* what is listed below: ||


{{{
iptables -t filter -NLocalFW
iptables -A FORWARD -j LocalFW
iptables -A INPUT -j LocalFW
iptables -A LocalFW -i lo -j ACCEPT
iptables -A LocalFW -p icmp -m icmp --icmp-type any -j ACCEPT
}}}
=== Building Squid ===
After preparing the kernel and iptables as above.

 * Build [[http://www.squid-cache.org/Versions/v3/3.1/|Squid 3.1 source]] as noted in the Squid readme and tproxy readme, enabling netfilter with:

{{{
--enable-linux-netfilter
}}}
|| /!\ || --enable-linux-tproxy was phased out because tproxy has been more tightly integrated with iptables/netfilter and Squid. ||


 * Configure squid as noted in the squid and tproxy readmes.

{{{
http_port 3129 tproxy
}}}
|| /!\ || A special http_port line is recommended since tproxy mode for Squid can interfere with non-tproxy requests on the same port. ||


----
 . CategoryConfigExample
