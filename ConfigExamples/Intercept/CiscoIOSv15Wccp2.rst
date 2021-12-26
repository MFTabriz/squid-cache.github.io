##master-page:CategoryTemplate
#format wiki
#language en

= Variant I: Routed DMZ witch WCCPv2 =

 ''by YuriVoinov''

== Configuring a Cisco IOS 15.5(3)M2 with WCCPv2 using ISR G2 router ==

<<Include(ConfigExamples, , from="^## warning begin", to="^## warning end")>>

<<TableOfContents>>

=== Outline ===

This configuration passes HTTP/HTTPS traffic (both port 80 and 443) over [[https://en.wikipedia.org/wiki/Web_Cache_Communication_Protocol|WCCPv2]] to proxy box for handling. It is expected the that the box will contain squid 3.x/4.x for processing the traffic.

The routers runs Cisco IOS Software, Version 15.5(3)M2, with SECURITYK9 and DATAK9 technology packs activated and have two physical interfaces - GigabitEthernet0/0 which connected to LAN switch, and GigabitEthernet0/1 (IP 192.168.200.2) connected to DMZ with proxy. Proxy has IP 192.168.200.3 in this example. WCCPv2 configured on router 2911.

{{attachment:Network_scheme.png | Network scheme}}

Router has both router/switch functionality, so we can use both GRE/L2 redirection methods.

 . {i} Note: Beware - you must have NAT configuted on your squid's box, and you must have squid built with OS-specific NAT support.
 . {i} Note: When using managed switch in DMZ, be sure proxy box port in the same VLAN/has the same encapsulation as router port with WCCP activated. Otherwise router can't do WCCP handshake with proxy.

## start feature include
=== Cisco IOS 15.5(3)M2 router ===
{{{
!
ip cef
ip wccp web-cache redirect-list WCCP_ACCESS
ip wccp 70 redirect-list WCCP_ACCESS
no ipv6 cef
!
!
!
interface GigabitEthernet0/1
 ip address 192.168.200.2 255.255.255.0
 ip wccp web-cache redirect out
 ip wccp 70 redirect out
!
!
ip route 0.0.0.0 0.0.0.0 192.168.200.1
!
ip access-list extended WCCP_ACCESS
 remark ACL for HTTP/HTTPS
 remark Squid proxies bypass WCCP
 deny   ip host 192.168.200.3 any
 remark LAN clients proxy port 80/443
 permit tcp 192.168.0.0 0.0.255.255 any eq www 443
 remark all others bypass WCCP
 deny   ip any any
!
!
}}}
 . {i} Note: ip wccp web-cache can redirect only HTTP (port 80), so to redirect HTTPS we create another dynamic wccp-service 70 (in Cisco WCCP documentation this number dedicated to HTTPS. In general, number in range 1-254, it does not matter, but remember it to specify in squid config). Also remember, SECURITYK9/DATAK9 technology packs need to be activated only in case HTTPS interception. They are not used for only HTTP redirection.

Also beware, when proxy is stopped - all HTTP/HTTPS traffic bypass it and passthrough default route to next hop (or last resort gateway).

## end feature include

## start feature include
=== Squid HTTP/HTTPS WCCPv2 configuration ===
{{{
# WCCPv2 parameters
wccp2_router 192.168.200.2
wccp2_forwarding_method l2
wccp2_return_method l2
wccp2_rebuild_wait off
wccp2_service standard 0
wccp2_service dynamic 70
wccp2_service_info 70 protocol=tcp flags=dst_ip_hash,src_ip_alt_hash,src_port_alt_hash priority=231 ports=443
}}}

 . {i} Note: Squid must be built with WCCPv2 support.
 . {i} Note: Squid box has configured default router pointed to 192.168.200.1 (Ge0/1 on 2901) - last resort gateway.
 . {i} Note: This example uses L2 redirecting (for OSes without native GRE support). Beware, wccp2_rebuild_wait sends "Here I am" message to router when proxy is ready to serve requests, without cache rebuilding complere. Also, both - router and proxy - uses port 2048/udp to communicate with WCCP. So, this port must be open in firewalls. The most important: When using l2 redirection, both - WCCP-enabled router port and proxy - must share the same L2 network segment.
 . {i} Note: If your choose GRE for communication with router and proxy, remember: you must have configured GRE on your proxy box!

## end feature include

==== Security ====

To avoid denial-of-service attacks, you can enforce authentification between proxy(proxies) and router. To do that you need to setup WCCP services on router using passwords:

{{{
ip wccp web-cache redirect-list WCCP_ACCESS password 0 foo123
ip wccp 70 redirect-list WCCP_ACCESS password 0 bar456
}}}

If your router has service password-encryption enabled (to do that you need to apply next command in router global configuration):

{{{
service password-encryption
}}}

after defining your WCCP services on router, passwords will be encrypted:

{{{
ip wccp web-cache redirect-list WCCP_ACCESS password 7 0600002E1D1C5A
ip wccp 70 redirect-list WCCP_ACCESS password 7 121B0405465E5A
}}}

Then change WCCP service definitions in squid.conf:

{{{
#	MD5 service authentication can be enabled by adding
#	"password=<password>" to the end of this service declaration.
wccp2_service standard 0 password=foo123
wccp2_service dynamic 70 password=bar456
}}}

Then restart squid and check redirection is working.

 . {i} Note: Beware your squid.conf contains '''any''' passwords in plain-text! Protect it as by as protect proxy box from unauthorized access!

=== Setup verification ===

To verify setup up and running execute next commands on WCCP-enabled router:

 {{attachment:Check_WCCP_1.png | Check WCCP HTTP/HTTPS}}
 {{attachment:Check_WCCP_2.png | Check WCCP HTTP/HTTPS}}
 {{attachment:Check_WCCP_3.png | Check WCCP Interfaces/summary}}
 {{attachment:Check_WCCP_up_and_running.png | Check WCCP up and running}}

=== QUIC/SPDY protocol blocking ===

 . {i} Note: In most modern installations you may want (and you must) to block alternate protocols: SPDY and/or QUIC. To do that, please use '''[[http://wiki.squid-cache.org/KnowledgeBase/Block%20QUIC%20protocol|this instructions]]'''.

=== Conclusion ===

This configuration example used on Cisco 2911 with Squid 3.x/4.x. As you can see, you can configure your environment for different ports interception.

 . {i} Note: '''Performance''' is more better against PBR (route-map), WCCP uses less CPU on Cisco's devices. So, WCCP is preferrable against route-map. Also note, l2 redirection has hardware support and less overhead, than gre, which has only software processing (on CPU).
 . {i} Note: This configuration was tested and fully operated on Cisco iOS versions 15.4(1)T, 15.4(3)M, 15.5(1)T, 15.5(2)T1, 15.5(3)M, 15.5(3)M2 and 15.6(2)T. {OK} {OK} {OK}

= Variant II: Switch L3 as WCCPv2 router =

 ''by YuriVoinov'' and Svyatoslav Voinov

== Configuring a Cisco IOS 15.0(2)SE9 with WCCPv2 using aggregation switch ==

=== Outline ===

This configuration passes HTTP/HTTPS traffic (both port 80 and 443) from switch L3 over [[https://en.wikipedia.org/wiki/Web_Cache_Communication_Protocol|WCCPv2]] to proxy box for handling. It is expected the that the box will contain squid 3.x/4.x for processing the traffic.

In this example uses Cisco 3750 aggregation switch in L3 mode as WCCPv2 router. The switch runs Cisco IOS Software, with appropriate version, in this example Version 15.0(2)SE9, with IPSERVICEK9 technology pack and has physical 1 Gbps interfaces. Proxy has two aggregated ports, IP's 192.168.201.10 (2 Gbps aggregate, aggr1) and 192.168.201.11 (4 Gbps aggregate, aggr2) in this example. WCCPv2 uses L2 redirection with assignment method '''mask'''. Switch only support WCCP "IN" redirection.

 {{attachment:Network_scheme2.png | Network scheme 2}}

=== Cisco IOS 15.0(2)SE9 switch ===

{{{
ip routing
 
ip wccp source-interface Vlan201
ip wccp web-cache redirect-list WCCP_ACCESS
ip wccp 70 redirect-list WCCP_ACCESS

interface GigabitEthernet1/0/15
 no switchport
 ip address 192.168.200.4 255.255.255.0

interface Vlan201
 ip address 192.168.201.1 255.255.255.0
 ip wccp web-cache redirect in
 ip wccp 70 redirect in

ip route 0.0.0.0 0.0.0.0 192.168.200.1

ip access-list extended WCCP_ACCESS
 remark ACL for HTTP/HTTPS
 remark Squid proxies bypass WCCP
 deny   ip host 192.168.201.10 any
 deny   ip host 192.168.201.11 any
 remark LAN clients proxy port 80/443
 permit tcp 192.168.0.0 0.0.255.255 any eq www 443
 remark all others bypass WCCP
 deny   ip any any
}}}

 . {i} Note: "WCCP is supported only on the SDM templates that support PBR: access, routing, and dual IPv4/v6 routing." (from Cisco documentation)

=== Squid HTTP/HTTPS WCCPv2 configuration ===

{{{
# -------------------------------------
# WCCPv2 parameters
# -------------------------------------
wccp2_router 192.168.201.1
wccp2_forwarding_method l2
wccp2_return_method l2
wccp2_rebuild_wait off
wccp2_service standard 0
wccp2_service dynamic 70
wccp2_service_info 70 protocol=tcp flags=dst_ip_hash,src_ip_alt_hash,src_port_alt_hash priority=231 ports=443
# Cisco Routers uses hash (default), switches - mask
wccp2_assignment_method mask

balance_on_multiple_ip on
}}}

 . {i} Note: As usual, it's expected your Squid already configured with HTTP and HTTPS ports.

==== Security ====

 . {i} Note: When using authenticated WCCP, like previous example, please note, Cisco equipement has passwork length limit (no more than 8 symbols on routers and most switches) and password strength limit - you can use only letters and numbers. Also was discovered additional password length limit on switches like 3750 with some old iOS, in this cases you'll be forced to reduce password length to 6-7 symbols.

=== Conclusion ===

This configuration example used on Cisco 3750 aggregation switch with Squid 3.x/4.x. As you can see, you can configure your environment for different ports interception.

 . {i} Note: Initial setup was created, tested and fully operated on switch WS-C3750G-16TD-S with iOS 12.2(55)SE10 IPSERVICEK9 with Squid 3.5.19. {OK}
 . {i} Note: This configuration was tested and fully operated on Cisco iOS version 15.0(2)SE9 on appropriate switch (see next note). {OK} {OK} {OK}
 . {i} Note: Be '''VERY CAREFUL''': Cisco 3750 series has much submodels, not at all compatible with iOS 15.x. Partially, C3750G-16TD-S can run iOS 12.2(55)SE series only. Read [[http://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/release/15-0_2_se/release/notes/OL25301.html|Cisco Release Notes]] first when choose iOS release. Quote from: " Not all Catalyst 3750 and 3560 switches can run this release. These models are not supported in Cisco IOS Release 12.2(58)SE1 and later: WS-C3560-24TS, WS-C3560-24PS. WS-C3560-48PS, WS-C3560-48TS, WS-C3750-24PS, WS-C3750-24TS, WS-C3750-48PS, WS-C3750-48TS, WS-3750G-24T, WS-C3750G-12S, WS-C3750G-24TS, WS-C3750G-16TD. For ongoing maintenance rebuilds for these models, use Cisco IOS Release 12.2(55)SE and later (SE1, SE2, and so on)." Also note, WCCP on this series of switches are supported starting from 12.2(37)SE iOS.
 . {i} Note: Against Variant I, switches hardware accelerated redirections don't shown in '''sho ip wccp''' command output. Check redirection work via Squid's access.log.

= Variant III: Switched ISR G2 router with WCCPv2 =

 ''by YuriVoinov'' and Svyatoslav Voinov

== Configuring a Cisco IOS 15.5(3)M2 with WCCPv2 using switched ISR G2 router with convergent switch board ==

=== Outline ===

This configuration passes HTTP/HTTPS traffic (both port 80 and 443) over [[https://en.wikipedia.org/wiki/Web_Cache_Communication_Protocol|WCCPv2]] to proxy box for handling. It is expected the that the box will contain squid 3.x/4.x for processing the traffic.

The routers runs Cisco IOS Software, Version 15.5(3)M2, with SECURITYK9 and DATAK9 technology packs activated. Router contains convergent switch board with four 100 Mbps or 1 Gbps ports. WCCPv2 configured on router 2911.

 {{attachment:Network_scheme3.png | Network scheme 3}}

=== Cisco IOS 15.5(3)M2 router ===

{{{
!
ip dhcp pool 100
 network 192.168.100.0 255.255.255.0
 default-router 192.168.100.1 
 dns-server 192.168.100.1 
 lease 30
!
ip dhcp pool 101
 network 192.168.101.0 255.255.255.0
 default-router 192.168.101.1 
 dns-server 192.168.101.1 
 lease 30
!
!
ip cef
ip wccp source-interface Vlan201
no ipv6 cef
!
!
interface FastEthernet0/0/0
 switchport access vlan 201
 no ip address
!
interface FastEthernet0/0/1
 switchport access vlan 201
 no ip address
!
interface FastEthernet0/0/2
 switchport mode trunk
 no ip address
!         
interface FastEthernet0/0/3
 switchport access vlan 100
 no ip address
!
!
interface Vlan100
 ip address 192.168.100.1 255.255.255.0
 ip wccp web-cache redirect in
 ip wccp 70 redirect in
 ip nat inside
 ip virtual-reassembly in
!
interface Vlan101
 ip address 192.168.101.1 255.255.255.0
 ip wccp web-cache redirect in
 ip wccp 70 redirect in
 ip nat inside
 ip virtual-reassembly in
!         
interface Vlan201
 ip address 192.168.201.1 255.255.255.0
 ip wccp web-cache redirect in
 ip wccp 70 redirect in
 ip nat inside
 ip virtual-reassembly in
!
ip nat inside source list NAT interface GigabitEthernet0/1 overload
ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/1 EXTERNAL_ISP_IP
!
ip access-list extended WCCP_ACCESS
 remark ACL for HTTP/HTTPS
 remark Squid proxies bypass WCCP
 deny   ip host 192.168.201.10 any
 remark LAN clients proxy port 80/443
 permit tcp 192.168.0.0 0.0.255.255 any eq www 443
!
}}}

 . {i} Note: Whenever clients and proxy connected via switch module, router uses '''hash''' assignment method itself.

=== Squid HTTP/HTTPS WCCPv2 configuration ===

{{{
# -------------------------------------
# WCCPv2 parameters
# -------------------------------------
wccp2_router 192.168.201.1
wccp2_forwarding_method l2
wccp2_return_method l2
wccp2_rebuild_wait off
wccp2_service standard 0
wccp2_service dynamic 70
wccp2_service_info 70 protocol=tcp flags=dst_ip_hash,src_ip_alt_hash,src_port_alt_hash priority=231 ports=443
# Cisco Routers uses hash (default), switches - mask
wccp2_assignment_method hash
}}}

==== Security ====

 . {i} Note: You can use authenticated WCCP like first example.

=== Conclusion ===

This configuration example uses switched ISR-G2 2911 router as central insfrastructure device. You can define as many client vlans as you required using access switches downstream infrastructure, and: on port Gi0/1 need to configure closed firewall with NAT due to security reasons.

= Variant IV: VRF with WCCPv2 =

 ''by YuriVoinov'' and Svyatoslav Voinov

=== Outline ===

This configuration passes HTTP/HTTPS traffic (both port 80 and 443) from [[https://en.wikipedia.org/wiki/Virtual_routing_and_forwarding|VRF]]-enabled router over [[https://en.wikipedia.org/wiki/Web_Cache_Communication_Protocol|WCCPv2]] to proxy box for handling. It is expected the that the box will contain squid 3.x/4.x for processing the traffic. You can have many VRF routers (on [[https://en.wikipedia.org/wiki/Multiprotocol_Label_Switching|MPLS]] router box) with the as many proxy boxes as you wish, for example, in ISP.


----
CategoryConfigExample
