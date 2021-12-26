= Schedule for Feature Removals =

Certain features are no longer relevant as the code improves and are planned for removal. Due to the possibility they are being used we list them here along with the release version they are expected to disappear. Warnings should also be present in the code where possible.

|| '''Version<<BR>>Removed''' || '''Feature''' || '''Why''' ||
|||| ||
|||| '''Deprecated in 4.0''' ||
|| 4.0 || ESI custom parser || Limited usability and repeated security issues. ||
|||| ||
|||| '''Deprecated in 3.5''' ||
|| TBD || [[Features/BumpSslServerFirst|ssl_bump server-first]] || Obsolete and broken security. Superceded by [[Features/SslPeekAndSplice|peek-n-splice]] ||
|| TBD || SSLv3 support || Obsolete and very broken security. But still widely used so no firm ETA. ||
|| 4.0 || SSLv2 support || RFC RFC:6176 compliance. ||
|||| ||
|||| '''Deprecated in 3.4''' ||
|| 3.5 || COSS storage type || Superceded by ROCK storage type ||
|||| ||
|||| '''Deprecated in 3.3''' ||
|| TBD || [[Features/SslBump|ssl_bump client-first]] || Obsolete and very broken security. Superceeded by [[Features/BumpSslServerFirst|ssl_bump server-first]] ||
|||| ||
|||| '''Deprecated in 3.2''' ||
|| TBD || cachemgr_passwd || Security is better controlled by login SquidConf:acl in the SquidConf:http_access configuration ||
|| TBD || cachemgr.cgi || Merger of report functionality into the main squid process obsoletes it as a stand-alone application. ||
|||| ||
|||| '''Deprecated in 3.1''' ||
|| TBD || error_directory files with named languages || Superseded by ISO-639 translations in [[Translations|langpack]] ||
|| TBD || Netmask Support in ACL || CIDR or RFC-compliant netmasks are now required by 3.1. Netmask support full removal yet to be decided. ||
|| 3.5 || dnsserver and DNS external helper API || Internal DNS client now appears to satisfy all use-cases. ||
|| 3.2 || Multiple languages per error page. || Superseded by auto-negotiation in 3.1+ ||
|| 3.2 || TPROXYv2 Support || TPROXYv4 available from 3.1 and native Linux kernels ||
|| 3.1 || libcap 1.x || libcap-2.09+ is required for simpler code and proper API usage. ||
|| 3.1 || cache_dir null || Default for cache_dir has been changed to not using a disk cache. ||
