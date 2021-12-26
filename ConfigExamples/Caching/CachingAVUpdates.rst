##master-page:CategoryTemplate
#format wiki
#language en

= Caching AV updates =

by ''YuriVoinov''

<<Include(ConfigExamples, , from="^## warning begin", to="^## warning end")>>

<<TableOfContents>>

== Outline ==

One of most common task is caching AV (antivirus) updates.

== Usage ==

To do this you can modify refresh_pattern to your Squid.

== Squid Configuration File ==

Paste the configuration file like this:

{{{

# AV updates
refresh_pattern -i \.symantecliveupdate.com\/.*\.(7z|irn|m26)		4320	100%	43200	reload-into-ims
refresh_pattern -i .*dnl.*\.geo\.kaspersky\.(com|ru)\/.*\.(zip|avc|kdc|nhg|klz|d[at|if])	4320	100%	43200	reload-into-ims
refresh_pattern -i \.kaspersky-labs.(com|ru)\/.*\.(cab|zip|exe|ms[i|p])	4320	100%	43200	reload-into-ims
refresh_pattern -i \.kaspersky.(com|ru)\/.*\.(cab|zip|exe|ms[i|p]|avc)	4320	100%	43200	reload-into-ims
refresh_pattern -i \.avast.com\/.*\.(vp[u|aa])		4320	100%	43200	reload-into-ims
refresh_pattern -i \.avg.com\/.*\.(bin)		4320	100%	43200	reload-into-ims
refresh_pattern -i \.securiteinfo.com\/.*\.([h|n]db|ign2)	14400	100%	518400	reload-into-ims

}}}


----
CategoryConfigExample
