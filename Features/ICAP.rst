##master-page:CategoryTemplate
#format wiki
#language en
#faqlisted yes #completed yes

= Feature: ICAP (Internet Content Adaptation Protocol) =
 * '''Status''': Completed.

 * '''Version''': 3.0

 * '''Developer''': AlexRousskov

<<TableOfContents>>

== Background ==
Internet Content Adaptation Protocol (RFC [[http://www.rfc-editor.org/rfc/rfc3507.txt|3507]], subject to [[http://www.measurement-factory.com/std/icap/|errata]]) specifies how an HTTP proxy (an ICAP client) can outsource content adaptation to an external ICAP server. Most popular proxies, including Squid, support ICAP. If your adaptation algorithm resides in an ICAP server, it will be able to work in a variety of environments and will not depend on a single proxy project or vendor. No proxy code modifications are necessary for most content adaptations using ICAP.

 . '''Pros''': Proxy-independent, adaptation-focused API, no Squid modifications, supports remote adaptation servers, scalable.
 '''Cons''': Communication delays, protocol functionality limitations, needs a stand-alone ICAP server process or box.

One proxy may access many ICAP servers, and one ICAP server may be accessed by many proxies. An ICAP server may reside on the same physical machine as Squid or run on a remote host. Depending on configuration and context, some ICAP failures can be bypassed, making them invisible to proxy end-users.

=== ICAP Servers ===
While writing yet another ICAP server from scratch is always a possibility, the following ICAP servers can be modified to support the adaptations you need. Some ICAP servers even accept custom adaptation modules or plugins.

 * [[http://c-icap.sourceforge.net/|C-ICAP]]
 * [[http://spicer.measurement-factory.com/|Traffic Spicer]]
 * [[http://www.poesia-filter.org/|POESIA]]
 * original [[http://www.icap-forum.org/documents/other/icap-server10.zip|reference implementation]] by Network Appliance.

The above list is not comprehensive and is not meant as an endorsement. Any ICAP server will have unique set of pros and cons in the context of your adaptation project.

More information about ICAP is available on the ICAP [[http://www.icap-forum.org/|Forum]]. While the Forum site has not been actively maintained, its members-only [[http://www.icap-forum.org/chat/|newsgroup]] is still a good place to discuss ICAP issues.


== Squid Details ==
[[Squid-3.0]] and later come with integrated ICAP support. Pre-cache REQMOD and RESPMOD vectoring points are supported, including request satisfaction. Squid-2 has limited ICAP support via a set of poorly maintained and very buggy patches. It is worth noting that the Squid developers no longer officially support the Squid-2 ICAP work.


== Squid Configuration ==

 (!) ICAP server configuration should be detailed in the server documentation. Squid is expected to work with any of them.

 {i} The configuration of Squid-3 underwent a change between [[Squid-3.0]] and [[Squid-3.1]]

=== Squid 3.1 ===
## TODO: get this verified by an expert or successful use.
The following example instructs [[Squid-3.1]] to talk to two ICAP services, one for request and one for response adaptation:

{{{
icap_enable on

icap_service service_req reqmod_precache 1 icap://127.0.0.1:1344/request
adaptation_access service_req allow all

icap_service service_resp respmod_precache 0 icap://127.0.0.1:1344/response
adaptation_access service_resp allow all
}}}

 * [[http://www.squid-cache.org/Doc/config/adaptation_access|adaptation_access]]
 * [[http://www.squid-cache.org/Doc/config/adaptation_service_set|adaptation_service_set]]

 * [[http://www.squid-cache.org/Doc/config/icap_client_username_encode|icap_client_username_encode]]
 * [[http://www.squid-cache.org/Doc/config/icap_client_username_header|icap_client_username_header]]
 * [[http://www.squid-cache.org/Doc/config/icap_connect_timeout|icap_connect_timeout]]
 * [[http://www.squid-cache.org/Doc/config/icap_default_options_ttl|icap_default_options_ttl]]
 * [[http://www.squid-cache.org/Doc/config/icap_enable|icap_enable]]
 * [[http://www.squid-cache.org/Doc/config/icap_io_timeout|icap_io_timeout]]
 * [[http://www.squid-cache.org/Doc/config/icap_persistent_connections|icap_persistent_connections]]
 * [[http://www.squid-cache.org/Doc/config/icap_preview_enable|icap_preview_enable]]
 * [[http://www.squid-cache.org/Doc/config/icap_preview_size|icap_preview_size]]
 * [[http://www.squid-cache.org/Doc/config/icap_send_client_ip|icap_send_client_ip]]
 * [[http://www.squid-cache.org/Doc/config/icap_send_client_username|icap_send_client_username]]
 * [[http://www.squid-cache.org/Doc/config/icap_service|icap_service]]
 * [[http://www.squid-cache.org/Doc/config/icap_service_failure_limit|icap_service_failure_limit]]
 * [[http://www.squid-cache.org/Doc/config/icap_service_revival_delay|icap_service_revival_delay]]

=== Squid 3.0 ===
## Pulled from release notes as-is.
The following example instructs [[Squid-3.0]] to talk to two ICAP services, one for request and one for response adaptation:

{{{
icap_enable on

icap_service service_req reqmod_precache 1 icap://127.0.0.1:1344/request
icap_class class_req service_req
icap_access class_req allow all

icap_service service_resp respmod_precache 0 icap://127.0.0.1:1344/response
icap_class class_resp service_resp
icap_access class_resp allow all
}}}
There are other options which can control various aspects of ICAP. See their configuration guide entries for more details:

 * [[http://www.squid-cache.org/Doc/config/icap_access|icap_access]] 
 * [[http://www.squid-cache.org/Doc/config/icap_class|icap_class]]
 * [[http://www.squid-cache.org/Doc/config/icap_client_username_encode|icap_client_username_encode]]
 * [[http://www.squid-cache.org/Doc/config/icap_client_username_header|icap_client_username_header]]
 * [[http://www.squid-cache.org/Doc/config/icap_connect_timeout|icap_connect_timeout]]
 * [[http://www.squid-cache.org/Doc/config/icap_default_options_ttl|icap_default_options_ttl]]
 * [[http://www.squid-cache.org/Doc/config/icap_enable|icap_enable]]
 * [[http://www.squid-cache.org/Doc/config/icap_io_timeout|icap_io_timeout]]
 * [[http://www.squid-cache.org/Doc/config/icap_persistent_connections|icap_persistent_connections]]
 * [[http://www.squid-cache.org/Doc/config/icap_preview_enable|icap_preview_enable]]
 * [[http://www.squid-cache.org/Doc/config/icap_preview_size|icap_preview_size]]
 * [[http://www.squid-cache.org/Doc/config/icap_send_client_ip|icap_send_client_ip]]
 * [[http://www.squid-cache.org/Doc/config/icap_send_client_username|icap_send_client_username]]
 * [[http://www.squid-cache.org/Doc/config/icap_service|icap_service]]
 * [[http://www.squid-cache.org/Doc/config/icap_service_failure_limit|icap_service_failure_limit]]
 * [[http://www.squid-cache.org/Doc/config/icap_service_revival_delay|icap_service_revival_delay]]

----
 CategoryFeature
