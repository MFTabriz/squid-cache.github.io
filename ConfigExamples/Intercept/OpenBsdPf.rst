## page was copied from ConfigExamples/InterceptWithPF
##master-page:CategoryTemplate
#format wiki
#language en

= Intercepting traffic with PF on OpenBSD =

by Chris Benech and Amos Jeffries

<<Include(ConfigExamples, , from="^## warning begin", to="^## warning end")>>

<<TableOfContents>>

== Outline ==

The Packet Filter (PF) firewall in OpenBSD 4.4 and later offers traffic interception using several very simple methods.

This configuration example details how to integrate the PF firewall with Squid for interception of port 80 traffic using either NAT-like interception and [[Features/Tproxy4|TPROXY-like]] interception.

More on configuring Squid for OpenBSD can be found in the OpenBSD ports README file:
 http://www.openbsd.org/cgi-bin/cvsweb/~checkout~/ports/www/squid/pkg/README-main

== Squid Configuration ==

=== Fully Transparent Proxy (TPROXY) ===
  /!\ This configuration requires [[Squid-3.3|Squid-3.3.4]] or later.

Squid requires the following build option:
{{{
--enable-pf-transparent
}}}

Use the '''tproxy''' traffic mode flag to instruct Squid that it is receiving intercepted traffic and to spoof the client IP on outgoing connections:
{{{
http_port 3129 tproxy
}}}

=== NAT Interception proxy ===

For [[Squid-3.4]] or later:
{{{
--enable-pf-transparent
}}}

 {i} For [[Squid-3.3]] and [[Squid-3.2]] support for this is not integrated with the --enable-pf-transparent build option. However the IPFW NAT component of Squid is compatible with PF. You can build Squid with these configure options:
{{{
--disable-pf-transparent --enable-ipfw-transparent
}}}

 {i} For [[Squid-2.7]], the default build with no particular configuration options uses the IPFW compatible method.

The squid packages and port for OpenBSD 5.0 and newer are built using this method.

Use the '''intercept''' traffic mode flag to instruct Squid that it is receiving intercepted traffic and to use its own IP on outgoing connections (emulating NAT):
{{{
http_port 3129 intercept
}}}


== pf.conf Configuration ==

In pf.conf, the following changes need to be made.

In the top portion where you set skip on your internal interfaces, remove those lines. They tell PF not to do any processing on packets coming in on an internal interface.
{{{
set skip on $int_if
set skip on $wi_if
}}}

=== OpenBSD 4.4 and later ===

On the machine running Squid, add a firewall rule similar to these...

For IPv6 traffic interception:
{{{
pass in quick inet6 proto tcp from 2001:DB8::/32 to port www divert-to ::1 port 3129
pass out quick inet6 from 2001:DB8::/32 divert-reply
}}}

For IPv4 traffic interception:
{{{
pass in quick on inet proto tcp from 192.0.2.0/24 to port www divert-to 127.0.0.1 port 3129
pass out quick inet from 192.0.2.0/24 divert-reply
}}}

'''IMPORTANT:''' The divert-reply rules are needed to receive replies for sockets that are bound to addresses not local to the machine. If there is no divert-reply rule, cache.log will show a line similar to:
  {{{
2013/04/16 14:28:37 kid1|  FD 12, 127.0.0.1 [Stopped, reason:Listener socket closed job49]: (53) Software caused connection abort
}}}

=== OpenBSD 4.1 to 4.3 ===

 {X} NOTE: OpenBSD older than 4.4 requires [[Squid-3.2]] or older built with '''--enable-pf-transparent''' and only supports the NAT interception method.

{{{
# redirect only IPv4 web traffic into squid 
rdr pass inet proto tcp from 192.168.231.0/24 to any port 80 -> 192.168.231.1 port 3129

block in
pass in quick on $int_if
pass in quick on $wi_if
pass out keep state
}}}

A pointer:

 * Use '''rdr pass''' instead of '''rdr on ...'''  part of the way that PF evaluates packets, it would drop through and be allowed as is instead of redirected if you don't use '''rdr pass'''.

== Troubleshooting ==

 * Make sure and add the '''pass in quick''' lines. Myself I have two internal interfaces, one for wired and one for wireless internet. Although there is a bridge configured, strange things happen sometimes when you don't explicitly allow all traffic on both interfaces. If you don't add these lines, you will lose local network connectivity and have to go to the console to figure it out.

=== No redirection is happening ===

 /!\ Make sure you removed the set '''skip on''' lines.

=== PfInterception: PF open failed: (13) Permission denied ===

This occurs if you are using --enable-pf-transparent and do not have write access to /dev/pf. It is recommended that you change to the {{{getsockname()}}} interface using "divert-to" pf rules with the following configure options:

{{{
--disable-pf-transparent --enable-ipfw-transparent
}}}

If you must use --enable-pf-transparent, change permissions on /dev/pf to allow write access to the userid running squid.

== Testing ==

To test if it worked, use the '''nc''' utility.
Stop squid and from the command line as root type in:
{{{
nc -l 3129
}}}

Then restart squid and try to navigate to a page.

You should now see an output like this:

{{{
<root:openbsd> [/root]
> nc -l 3129
GET /mail/?ui=pb HTTP/1.1
User-Agent: Mozilla/5.0 (compatible; GNotify 1.0.25.0)
Host: mail.google.com
Connection: Keep-Alive
Cache-Control: no-cache
...
}}}

From there on out, just set your browsers up normally with no proxy server, and you should see the cache fill up and your browsing speed up.

----
CategoryConfigExample
