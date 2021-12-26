##master-page:Features/FeatureTemplate
#format wiki
#language en
#faqlisted no

= Feature: ACL type "Random" =

 * '''Goal''': Implement an ACL type which would match randomly with a given probability.
 * '''Status''': done
 * '''Version''': 3.2
 * '''Developer''': AmosJeffries
 * '''More''': Bug Bug:1239


= Details =

The ACL of type "random" will accept a single value in one of three formats:

 * A:B - matching randomly an average A requests for every B non-matches. A and B may not be zero.

 * A/B - matching randomly an average of A requests out of total B requests. A and B may not be zero.

 * 0.NNNN - matching randomly any given request with .NNNN probability.
   Range is between zero to one, excluding zero and one themselves.

All three of these matches are proportional. The first two formats are provided for ease of configuration. They are converted to a decimal threshold as shown in the third format.

Every test, a new random number is generated and checked against the stored value. If the random number is within the threshold range of possibility the ACL will match.

 {i} To debug this ACL use SquidConf:debug_options 28,3 and watch for lines beginning with "ACL Random".

= Use Cases =
== Uplink Load Balancing ==
When used within SquidConf:tcp_outgoing_address or SquidConf:tcp_outgoing_tos selection this ACL permits load to be roughly split between multiple links based on their relative capacity.

This requires some additional configuration at the operating system level to ensure that the address or TOS values assigned are routed out the appropriate uplink. It is no use doing this in Squid if all traffic ends up going out the default anyway.

 * ''' Example 1:''' Split two uplinks roughly 50% of traffic each:
{{{
acl fiftyPercent random .5

# a random 50% go here
tcp_outgoing_address 192.0.2.1 fiftyPercent

# the rest go here
tcp_outgoing_address 192.0.2.2

# NP: operating system required to route packets from 192.0.2.1 and 192.0.2.2 out separate uplinks.
}}}


 * '''Example 2:''' Split traffic one third to each of three peers.
{{{
acl third random 1/3

# 33% traffic goes here
cache_peer_access peerOne allow third
cache_peer_access peerOne deny all

# 30% traffic goes here
cache_peer_access peerTwo allow third
cache_peer_access peerTwo deny all

# remaining traffic goes here
cache_peer_access peerOne allow all
}}}


 * '''Example 3:''' Split traffic one third to each of two peers and direct.
{{{
acl third random 1/3
acl half random 1/2

# 33% traffic goes direct
always_direct allow third

# 33% traffic goes here (half of what did not go direct already)
cache_peer_access peerOne deny half
cache_peer_access peerOne allow all

# remaining traffic goes here
cache_peer_access peerTwo allow all

# NP: if both peers are down DIRECT will be used as a backup.
}}}




== Log sampling of traffic ==
When used in SquidConf:access_log directives this ACL permits a small random proportion of requests to be logged. Rather than all traffic or some only matching fixed criteria.

 * '''Example 1:''' Log one line randomly out of every 100 requests.
{{{
# log 1, skip 99
acl logSmallSample random 1:99

# small log 1 of every 100 requests...
access_log /var/log/squid/access-sample.log squid logSmallSample

# old style complete log
access_log /var/log/squid/access.log squid
}}}

== Others? ==
Other use cases may be possible. If you know of one not already covered here we are interested to know what it is.

----
CategoryFeature
