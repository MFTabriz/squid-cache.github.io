##master-page:Features/FeatureTemplate
#format wiki
#language en
#faqlisted yes

= Feature: Add-On Helpers for Request Manipulation =

 * '''Goal''': Support simple customization of request handling for local requirements.

 * '''Status''': Completed. 2.5

 * '''Version''': 2.5

## * '''Developer''': unknown

 * '''More''': <<BR>>
  (NTLM) http://squid.sourceforge.net/ntlm/squid_helper_protocol.html <<BR>>
  (Digest auth)  [[KnowledgeBase/LdapBackedDigestAuthentication]] <<BR>>
  (Kerberos auth) [[ConfigExamples/Authenticate/Kerberos]]

<<TableOfContents>>

== Details ==

Every network and installation have their own criteria for operation. The squid developers and community do not have the time or inclination to write code for every minor situation. Instead we provide ways to easily extend various operations with local add-on scripts or programs we call helpers.

=== What language are helper meant to be written in? ===
Helpers can be written in any language you like. They can be executable programs or interpreted scripts.

The helpers bundled with Squid are currently written in Bash shell script, awk script, perl script, and C++. There are also frameworks available for helpers built in Python or Ruby.

=== How do the helpers communicate with Squid? ===
The interface with Squid is very simple. The helper is passed a limited amount of information on stdin to perform their expected task. The result is passed back to Squid via stdout. With any errors or debugging traces sent back on stderr.

 * The helper is expected to wait for input before responding. Early responses or unexpected lines arriving to Squid on stdin will result in the helper being shut down with errors. If that happens often Squid may terminate as well.

 * The helper is expected to handle multiple lookups by Squid. It is a terrible drag on HTTP speed to be constantly re-starting helpers. If that happens often enough Squid may terminate as well.

See the particular interface protocols below for details about the line syntax the helper is expected to receive and send on each interface.

=== Why is my helper not starting up with Squid? ===
Squid-3.2 and newer support dynamic helper initialization. That means the helper is only started if it needs to be. If Squid is configured with startup=N value greater than 0 you can expect that many of your helper to be started when Squid starts. But this is not necessarily a desirable thing for Squid needing fast startup or restart times.

With startup=0 configured the first HTTP request through Squid is expected to start at least one instance for most of the helpers. But if for example an external ACL is configured and is only tested on rare occasions its helper will not be started until that rare occasion happens for the first time.

=== What happens when Squid shuts down or reconfigures? ===
When shutting down, reconfiguring, or in other times Squid needs to shutdown a helper. Squid schedules a shutdown of the helper, which will happen when all the in-progress lookups are completed.

Shutting down or restarting are limited by the SquidConf:shutdown_timeout which may cause Squid to abort earlier than receiving all the responses. If this happens the client connections are also being terminated just as abruptly as the helper - so the lost helper responses are not an issue.

=== Can I write a helper that talks to Squid on more than one interface? ===
You can. In a way.

Squid runs the configured helper for each interface as a separate child process. Your helper can be written to detect other running instances of itself and communicate between them, effectively sharing memory and/or state outside of Squid regardless of the interface Squid is using to run each instance.

NP: Just keep in mind that the number of instances (children) running on each interface is configurable and could be anything form zero to many hundreds. So do not make any assumptions about which interface another instance is running on.

== Squid operations which provide a helper interface ==

Squid-2.6 and later all support:
 * URL manipulation: re-writing and redirection
  * (SquidConf:url_rewrite_program, SquidConf:url_rewrite_access)
  * Specific feature details at [[Features/Redirectors]]
 * ACL logic tests
  * (SquidConf:external_acl_type)
 * Authentication
  * (SquidConf:auth_param)
  * Specific feature details at [[Features/Authentication]] [[Features/NegotiateAuthentication]]
 * cache file eraser
  * SquidConf:unlinkd_program
 * DNS lookup (removed in Squid-3.5)
  * SquidConf:dns_program

Squid-2.7 (only):
 * HTTP Server redirection replies
  * (SquidConf:location_rewrite_program, SquidConf:location_rewrite_access)
 * Cache object de-duplication
  * (SquidConf:storeurl_rewrite_program, SquidConf:storeurl_rewrite_access)
  * Specific feature details at [[Features/StoreUrlRewrite]]

Squid-2.7 and Squid-3.1+ support:
 * Logging
  * (SquidConf:logfile_daemon)
  * Specific feature details at [[Features/LogModules]]

Squid-3.1+ support:
 * SSL certificate generation (3.1.12.1 and later).

Squid-3.4+ support:
 * Cache object de-duplication
  * (SquidConf:store_id_program, SquidConf:store_id_access, SquidConf:store_id_children, SquidConf:store_id_bypass)
  * Specific feature details at [[Features/StoreID]]
 * SSL certificate validation
  * (SquidConf:sslcrtvalidator_program, SquidConf:sslcrtvalidator_children)
  * Specific feature details at [[Features/SslServerCertValidator]]

Squid-3.1 and later also support [[Features/eCAP|eCAP plugins]] and [[Features/ICAP|ICAP services]] which differ from helper scripts in many ways.

== Helper protocols ==

{i} Squid-2.6 and later all support concurrency, however the bundled helpers and many third-party commercial helpers do not. This is changing, the use of concurrency is encouraged to improve performance. The relevant squid.conf concurrency setting must match the helper concurrency support. The [[Features/HelperMultiplexer|helper multiplexer]] wrapper can be used to add concurrency benefits to most non-concurrent helpers.

 /!\ '''WARNING:''' For every line sent by Squid exactly one line is expected back. Some script language such as perl and python need to be careful about the number of newlines in their output.

 /!\ Note that the helper programs other than logging can not use buffered I/O.

=== Key-Value pairs format ===

{i} Relevant to Squid-3.4 and later

The interface for all helpers has been extended to support arbitrary lists of key=value pairs, with the syntax {{{ key=value }}}.
Some keys have special meaning to Squid, as documented here.
All messages from squid are URL-escaped (the {{{ rfc1738_unescape }}} from rfc1738.h can be used to decode them.
For responses, the safe way is to either URL-escape, or to enclose the value in double_quotes ("); any double-quotes or backslashes (\) in the value need to be prefixed by a backslash, \r and \n are replaced respectively by CR and LF

Some example key values:
{{{
		user=John%20Smith
		user="John Smith"
		user="J. \"Bob\" Smith"
}}}
=== URL manipulation ===

## start urlhelper protocol
Input line received from Squid:
{{{
[channel-ID] URL ip/fqdn ident method [urlgroup] kv-pair
}}}

 channel-ID::
  This is the concurrency channel number. When concurrency is turned off (set to '''1''') this field and the following space will be completely missing.

 URL::
  The URL received from the client. In Squid with ICAP support, this is the URL after ICAP REQMOD has taken place.

 ip::
  This is the IP address of the client. Followed by a slash ('''/''') as shown above.

 fqdn::
  The FQDN rDNS of the client, if any is known. Squid does not normally perform lookup unless needed by logging or ACLs. Squid does not wait for any results unless ACLs are configured to wait. If none is available '''-''' will be sent to the helper instead.

 ident::
  The IDENT protocol username (if known) of the client machine. Squid will not wait for IDENT username to become known unless there are ACL which depend on it. So at the time re-writers are run the IDENT username may not yet be known. If none is available '''-''' will be sent to the helper instead.

 method::
  The HTTP request method. URL alterations and particularly redirection are only possible on certain methods, and some such as POST and CONNECT require special care.

 urlgroup::
  Squid-2 will send this field with the URL-grouping tag which can be configured on SquidConf:http_port. Squid-3.x will not send this field.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || myip=... || Squid receiving address ||
  || myport=... || Squid receiving port ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||

## end urlhelper protocol

==== HTTP Redirection ====

## start redirector protocol
Redirection can be performed by helpers on the SquidConf:url_rewrite_program interface. Lines performing either redirect or re-write can be produced by the same helpers on a per-request basis. Redirect is preferred since re-writing URLs introduces a large number of problems into the client HTTP experience.

The input line received from Squid is detailed by the section above.

Redirectors send a slightly different format of line back to Squid. 

Result line sent back to Squid:
{{{
[channel-ID] [result] [kv-pairs] [status:URL]
}}}

 channel-ID::
  When a concurrency '''channel-ID''' is received it must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. A new URL is presented. ||
  || ERR || Success. No action for this URL. ||
  || BH || Failure. The helper encountered a problem. ||
  . {i} the result field is only accepted by [[Squid-3.4]] and newer.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface for HTTP redirection:
  || message=... || reserved ||
  || status=... || HTTP status code to use on the redirect. Must be one of: 301, 302, 303, 307, 308 ||
  || tag=... || reserved ||
  || ttl=... || reserved ||
  || url=... ||  redirect the client to given URL ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||
  . {i} the kv-pair field is only accepted by [[Squid-3.4]] and newer.
  . {i} the kv-pair returned by this helper can be logged by the '''%note''' SquidConf:logformat code.

 status::
   The HTTP 301, 302 or 307 status code. Please see section 10.3 of RFC RFC:2616 for an explanation of the HTTP redirect codes and which request methods they may be sent on.

 URL::
  The URL to be used instead of the one sent by the client. This must be an absolute URL. ie starting with http:// or ftp:// etc.
 . {i} If no action is required leave status:URL area blank.
 . {i} The '''status''' and '''URL''' are separated by a colon (''':''') as shown above instead of whitespace.

## end redirector protocol

==== URL Re-Writing (Mangling) ====

## start urlrewrite protocol
URL re-writing can be performed by helpers on the SquidConf:url_rewrite_program, SquidConf:storeurl_rewrite_program and SquidConf:location_rewrite_program interfaces.

WARNING: when used on the url_rewrite_program interface re-writing URLs introduces a large number of problems into the client HTTP experience. Some of these problems can be mitigated with a paired helper running on the SquidConf:location_rewrite_program interface de-mangling the server redirection URLs.

## start urlrewrite onlyprotocol

Result line sent back to Squid:
{{{
[channel-ID] [result] [kv-pair] [URL]
}}}

 channel-ID::
  When a concurrency '''channel-ID''' is received it must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. A new URL is presented ||
  || ERR || Success. No change for this URL. ||
  || BH || Failure. The helper encountered a problem. ||
  . {i} the result field is only accepted by [[Squid-3.4]] and newer.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface for URL re-writing:
  || message=... || reserved ||
  || rewrite-url=... || re-write the transaction to the given URL. ||
  || tag=... || reserved ||
  || ttl=... || reserved ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||
  . {i} the kv-pair field is only accepted by [[Squid-3.4]] and newer.
  . {i} the kv-pair returned by this helper can be logged by the '''%note''' SquidConf:logformat code.
##  . {i} if the '''url=''' kv-pair for HTTP redirection is present re-write operation will not be performed.

 URL::
  The URL to be used instead of the one sent by the client. If no action is required leave the URL field blank. The URL sent must be an absolute URL. ie starting with http:// or ftp:// etc.

## end urlrewrite protocol

==== Store ID de-duplication ====

## start storeid protocol
URL to Store-ID mapping can be performed by helpers on the SquidConf:storeid_rewrite_program interface.

WARNING: care must be taken that the URLs de-duplicated onto one shared ID are actually duplicates. Clients needing to revalidate will cause the cached object to be sourced from either of the duplicate locations. If they are not real duplicates this can randomly cause major issues with the client experience.

## start storeid onlyprotocol

Result line sent back to Squid:
{{{
[channel-ID] result kv-pair
}}}

 channel-ID::
  When a concurrency '''channel-ID''' is received it must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. A new storage ID is presented for this URL. ||
  || ERR || Success. No change for this URL. ||
  || BH || Failure. The helper encountered a problem. ||

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface for URL re-writing:
  || message=... || reserved ||
  || store-id=... || set the cache storage ID for this URL. ||
  || tag=... || reserved ||
  || ttl=... || reserved ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||
  . {i} the kv-pair returned by this helper can be logged by the '''%note''' SquidConf:logformat code.

 {i} This interface will also accept responses in the syntax delivered by [[Features/StoreUrlRewrite|Store URL-rewrite]] feature helpers written for [[Squid-2.7]]. However thst syntax is deprecated and such helpers should be upgraded as soon as possible to use this Store-ID syntax.

## end storeid protocol

=== Authenticator ===

==== Basic Scheme ====

## start basicauth protocol
Input line received from Squid:
{{{
[channel-ID] username password
}}}

 channel-ID::
  This is the concurrency channel number. When concurrency is turned off (set to '''1''') this field and the following space will be completely missing.

 username::
  The username field sent by the client in HTTP headers. It may be empty or missing.

 password::
  The password value sent by the client in HTTP headers. May be empty or missing.


Result line sent back to Squid:
{{{
[channel-ID] result [kv-pair]
}}}

 channel-ID::
  When a concurrency '''channel-ID''' is received it must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. Valid credentials. ||
  || ERR || Success. Invalid credentials. ||
  || BH || Failure. The helper encountered a problem. ||
  . {i} the '''BH''' result code is only accepted by [[Squid-3.4]] and newer.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || group=... || reserved ||
  || message=... || A message string that Squid can display on an error page. ||
  || tag=... || reserved ||
  || ttl=... || reserved ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||
  . {i} the kv-pair field is only accepted by [[Squid-3.4]] and newer.
  . {i} the kv-pair returned by this helper can be logged by the '''%note''' SquidConf:logformat code.


## end basicauth protocol

==== Bearer Scheme ====

  . {i} the '''Bearer''' authentication scheme is '''proposed''' to be supported by [[Squid-3.4]] and newer. But not yet implemented.

## start bearerauth protocol
Input line received from Squid:
{{{
channel-ID b64token
}}}

 channel-ID::
  This is the concurrency channel number.

 b64token::
  The opaque credentials token field sent by the client in HTTP headers.


Result line sent back to Squid:
{{{
channel-ID result [kv-pair]
}}}

 channel-ID::
  The concurrency '''channel-ID''' as received. It must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. Valid credentials. ||
  || ERR || Success. Invalid credentials. ||
  || BH || Failure. The helper encountered a problem. ||

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || group=... || reserved ||
  || message=... || A message string that Squid can display on an error page. ||
  || tag=... || reserved ||
  || ttl=... || reserved ||
  || user=... || reserved ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||

## end bearerauth protocol

==== Digest Scheme ====

## start digestauth protocol
Input line received from Squid:
{{{
[channel-ID] "username":"realm"
}}}

 channel-ID::
  This is the concurrency channel number. When concurrency is turned off (set to '''1''') this field and the following space will be completely missing.

 username::
  The username field sent by the client in HTTP headers. Sent as a "double-quoted" string. May be empty. It may be configured to use UTF-8 bytes instead of the ISO-8859-1 received.

 realm::
  The digest auth realm string configured in squid.conf. Sent as a "double-quoted" string.

{i} The '''username''' and '''realm''' strings are both double quoted ('''"''') and separated by a colon (''':''') as shown above.


Result line sent back to Squid:
{{{
[channel-ID] [result] [kv-pair] [hash]
}}}

 channel-ID::
  When a concurrency '''channel-ID''' is received it must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. Valid credentials. Digest HA1 value is presented. ||
  || ERR || Success. Invalid credentials. ||
  || BH || Failure. The helper encountered a problem. ||
  . {i} the '''OK''' and '''BH''' result codes are only accepted by [[Squid-3.4]] and newer.<<BR>>
  . {i} for [[Squid-3.3]] and older the '''OK''' result is not sent, but hash field is.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || group=... || reserved ||
  || ha1=... || The digest HA1 value to be used. This field is only used on '''OK''' responses. ||
  || message=... || A message string that Squid can display on an error page. ||
  || tag=... || reserved ||
  || ttl=... || reserved ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||
  . {i} the kv-pair field is only accepted by [[Squid-3.4]] and newer.
  . {i} the kv-pair returned by this helper can be logged by the '''%note''' SquidConf:logformat code.

 hash::
  The digest HA1 value to be used. This field is only accepted on '''OK''' responses.<<BR>>
  /!\ This field is deprecated on Squid-3.4 and newer, use the '''ha1''' kv-pair instead.

## end digestauth protocol

==== Negotiate and NTLM Scheme ====

## start negotiateauth protocol
 {i} These authenticator schemes do not support concurrency due to the statefulness of NTLM.

Input line received from Squid:
{{{
 request [credentials]
}}}

 request::
  One of the request codes:
  || YR || A new challenge token is needed. This is always the first communication between the two processes. It may also occur at any time that Squid needs a new challenge, due to the SquidConf:auth_param max_challenge_lifetime and max_challenge_uses parameters. The helper should respond with a '''TT''' message. ||
  || KK || Authenticate a user's credentials. The helper responds with either '''OK''', '''ERR''', '''AF''', '''NA''', or '''BH'''. ||

 credentials::
  An encoded blob exactly as received in the HTTP headers. This field is only sent on '''KK''' requests.


Result line sent back to Squid:
{{{
 result [token label] [kv-pair] [message]
}}}

 result::
  One of the result codes:
  || TT || Success. A new challenge '''token''' value is presented. ||
  || AF || Success. Valid credentials. Deprecated by '''OK''' result from Squid-3.4 onwards. ||
  || NA || Success. Invalid credentials. Deprecated by '''ERR''' result from Squid-3.4 onwards. ||
  || OK || Success. Valid Credentials. ||
  || ERR || Success. Invalid credentials. ||
  || BH || Failure. The helper encountered a problem. ||
  . {i} the '''OK''' and '''ERR''' result codes are only accepted by [[Squid-3.4]] and newer.

 token::
  A new challenge '''token''' value is presented. The token is base64-encoded, as defined by RFC RFC:2045.<<BR>>
  {i} NOTE: NTLM authenticator interface on Squid-3.3 and older does not support a '''token''' field. Negotiate authenticator interface requires it on '''TT''', '''AF''' and '''NA''' responses.<<BR>>
  {i} This field must not be sent on '''OK''', '''ERR''' and '''BH''' responses.

 label::
  The label given here is what gets used by Squid for this client request '''"username"'''. This field is only accepted on '''AF''' responses. It must not be sent on any other result code response.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || group=... || reserved ||
  || message=... || A message string that Squid can display on an error page. ||
  || tag=... || reserved ||
  || token=... || The base64-encoded, as defined by RFC RFC:2045, token to be used. This field is only used on '''OK''' responses. ||
  || ttl=... || reserved ||
  || user=... || The label to be used by Squid for this client request as '''"username"'''. With Negotiate and NTLM protocols it typically has the format NAME@DOMAIN or NAME\\DOMAIN respectively. ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||
  . {i} the kv-pair field is only accepted by [[Squid-3.4]] and newer.
  . {i} the kv-pair returned by this helper can be logged by the '''%note''' SquidConf:logformat code.
  . /!\ This field is only accepted on '''OK''', '''ERR''' and '''BH''' responses and must not be sent on other responses.

 message::
  A message string that Squid can display on an error page. This field is only accepted on '''NA''' and '''BH''' responses. From Squid-3.4 this field is deprecated by the '''message=''' kv-pair on '''BH''' responses.

## end negotiateauth protocol

=== Access Control (ACL) ===

## start externalacl protocol
This interface has a very flexible field layout. The administrator may configure any number or order of details from the relevant HTTP request or reply to be sent to the helper.

Input line received from Squid:
{{{
[channel-ID] format-options [acl-value [acl-value ...]]
}}}

 channel-ID::
  This is the concurrency channel number. When concurrency is turned off ('''concurrency=1''') in SquidConf:external_acl_type this field and the following space will be completely missing.

 format-options::
  This is the flexible series of tokens configured as the '''FORMAT''' area of SquidConf:external_acl_type. The tokens are space-delimited and exactly match the order of '''%''' tokens in the configured '''FORMAT'''. By default in current releases these tokens are also URL-encoded according to RFC RFC:1738 to protect against whitespace and binary data problems.

 acl-value::
  Some ACL tests such as group name comparisons pass their test values to the external helper following the admin configured FORMAT. Depending on the ACL these may be sent one value at a time, as a list of values, or nothing may be sent. By default in current releases these tokens are also URL-encoded according to RFC RFC:1738 to protect against whitespace and binary data problems.


Result line sent back to Squid:
{{{
[channel-ID] result [kv-pair]
}}}

 channel-ID::
  When a concurrency '''channel-ID''' is received it must be sent back to Squid unchanged as the first entry on the line.

 result::
  One of the result codes:
  || OK || Success. ACL test matches. ||
  || ERR || Success. ACL test fails to match. ||
  || BH || Failure. The helper encountered a problem. ||
  .The configured usage of the external ACL in squid.conf determines what this result means.<<BR>>
  . {i} the '''BH''' result code is only accepted by [[Squid-3.4]] and newer.

 kv-pair::
  One or more key=value pairs. see SquidConf:external_acl_type for the full list supported by your Squid.

  The key names reserved on this interface:
  || group=... || reserved ||
  || log=... || String to be logged in access.log. Available as '''%ea''' in SquidConf:logformat specifications ||
  || message=... || Message describing the reason. Available as %o in error pages ||
  || password=... || The users password (for login= SquidConf:cache_peer option) ||
  || tag=... || Apply a tag to a request (for both '''ERR''' and '''OK''' results). Only sets a tag, does not alter existing tags. ||
  || ttl=... || reserved ||
  || user=... || The users name (login) ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||

## end externalacl protocol

=== Logging ===
## start logdaemon protocol
Squid sends a number of commands to the log daemon. These are sent in the first byte of each input line:

 || L<data>\n || logfile data ||
 || R\n || rotate file ||
 || T\n || truncate file ||
 || O\n || re-open file ||
 || F\n || flush file ||
 || r<n>\n || set rotate count to <n> ||
 || b<n>\n || 1 = buffer output, 0 = don't buffer output ||

No response is expected. Any response that may be desired should occur on stderr to be viewed through cache.log.
## end logdaemon protocol

=== SSL certificate generation ===

## start sslcrtd protocol
This interface has a fixed field layout.

Input ''line'' received from Squid:
{{{
request size kv-pair [body]
}}}

/!\ ''line'' refers to a logical input. '''body''' may contain \n characters so each line in this format is delimited by a 0x01 byte instead of the standard \n byte.

 request::
  The type of action being requested. Presently the code '''new_certificate''' is the only request made.

 size::
  Total size of the following request bytes taken by the '''key-pair''' parameters and '''body'''.

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || host= || FQDN host name of the domain needing a certificate. ||

 body::
  An optional CA certificate and private RSA key to sign with. If this body field is omitted the generated certificate will be self-signed.
  The content of this field is ASCII-armoured PEM format.
  {{{
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
-----BEGIN RSA PRIVATE KEY-----
...
-----END RSA PRIVATE KEY-----
  }}}

Result line sent back to Squid:
{{{
result size [key-pair] body
}}}

 result::
  One of the result codes:
  || OK || Success. A certificate is ready ||
  || BH || Failure. The helper encountered a problem. ||
  . {i} the '''OK''' and '''BH''' result codes are only accepted by [[Squid-3.3]] and newer.<<BR>>
  . /!\ The helper will display an error message and abort if any error or unexpected event is detected.

 size::
  Total size of the following request bytes taken by the '''body'''.

 key-pair::
  Optional key=value parameters.

  Some of the key=value pairs:
  || host= || FQDN host name of the domain this certificate is for. ||

 body::
  The generated CA certificate. The content of this field is ASCII-armoured PEM format.
  {{{
-----BEGIN CERTIFICATE-----
...
-----END CERTIFICATE-----
  }}}

## end sslcrtd protocol

=== SSL server certificate validator ===

## start sslcrtvd protocol
This interface is similar to the SSL certificate generation interface.

Input ''line'' received from Squid:
{{{
request size [key-pair]
}}}

/!\ ''line'' refers to a logical input. '''body''' may contain \n characters so each line in this format is delimited by a 0x01 byte instead of the standard \n byte.

 request::
  The type of action being requested. Presently the code '''cert_validate''' is the only request made.

 size::
  Total size of the following request bytes taken by the '''key=pair''' parameters.

 key-pair::
  The supported key=value pairs are:
  || domain || FQDN host name or the domain ||
  || cert_'''''ID''''' || Server certificate. The ID is an index number for this certificate. This parameter exist as many as the server certificates are||
  || error_name_'''''ID''''' || The openSSL certificate validation error. The ID is an index number for this error ||
  || error_name_'''''ID''''' || The ID of the certificate which caused error_name_ID ||

Example request:
{{{
cert_validate 1519 host=dmz.example-domain.com
cert_0=-----BEGIN CERTIFICATE-----
MIID+DCCA2GgAwIBAgIJAIDcHRUxB2O4MA0GCSqGSIb3DQEBBAUAMIGvMQswCQYD
...
YpVJGt5CJuNfCcB/
-----END CERTIFICATE-----
error_name_0=X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT
error_cert_0=cert0
}}}

Result line sent back to Squid:
{{{
result size key-pair
}}}

 result::
  One of the result codes:
  || OK || Success. Certificate validated. ||
  || BH || Failure. The helper encountered a problem. ||

 size::
  Total size of the following response bytes taken by the '''key=pair''' parameters.

 key-pair::
  The supported key=value pairs are:
  || cert_'''''ID''''' || A certificate send from helper to squid. The '''ID''' is an index number for this certificate ||
  || error_name_'''''ID''''' || The openSSL error name for the error '''ID''' ||
  || error_reason_'''''ID'''''|| A reason for the error '''ID'''||
  || error_cert_'''''ID''''' || The broken certificate. It can be one of the certificates sent by helper to squid or one of those sent by squid to helper||

Example response message:
{{{
OK 1444 cert_10=-----BEGIN CERTIFICATE-----
MIIDojCCAoqgAwIBAgIQE4Y1TR0/BvLB+WUF1ZAcYjANBgkqhkiG9w0BAQUFADBr
...
398znM/jra6O1I7mT1GvFpLgXPYHDw==
-----END CERTIFICATE-----
error_name_0=X509_V_ERR_DEPTH_ZERO_SELF_SIGNED_CERT
error_reason_0=Checked by Cert Validator
error_cert_0=cert_10
}}}

## end sslcrtvd protocol

=== Cache file eraser ===

## start unlinkd protocol
The unlink() function used to erase files is a blocking call and can slow Squid down. This interface is used to pass file erase instructions to a helper program specified by SquidConf:unlinkd_program.

This interface has a fixed field layout. As of [[Squid-3.3]] this interface does not support concurrency. It requires Squid to be built with '''--enable-unlinkd''' and only cache storage types which use disk files (UFS, AUFS, diskd) use this interface.

Input line received from Squid:
{{{
path
}}}

 path::
  The file to be erased.

Result line sent back to Squid:
{{{
result [kv-pair]
}}}

 result::
  One of the result codes:
  || OK || Success. The file has been removed from cache. ||
  || BH || Failure. The helper encountered a problem. ||

 kv-pair::
  One or more key=value pairs. The key names reserved on this interface:
  || message=... || reserved ||
  || tag=... || reserved ||
  || *_=... || Key names ending in (_) are reserved for local administrators use. ||

## end unlinkd protocol
----
CategoryFeature
