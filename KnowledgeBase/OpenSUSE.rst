##master-page:CategoryTemplate
##master-date:Unknown-Date
#format wiki
#language en

## This page is for the availability, contact and troubleshooting information
## relevant to a single operating system distribution on which Squid may be installed
##
## For consistency and wiki automatic inclusion certain headings and high level layout
## needs to be kept unchanged.
## Feel free to edit within each section as needed to make the content flow easily for beginners.
##


= Squid on OpenSUSE =

<<TableOfContents>>

== Pre-Built Binary Packages ==

## Details briefly covering critical information for user contact and problem reporting...
##
'''Maintainer:''' unknown

'''Bug Reporting:''' https://bugzilla.novell.com/buglist.cgi?quicksearch=squid

==== Squid-3.1 ====

Install Procedure:

 /!\ Seeking information. Help?

## Exact sequence of command-line commands or GUI actions used to install this package on the distro.
##{{{
##...
##}}}

==== Squid-2.7 ====

Install Procedure:
{{{
...
}}}

== Compiling ==

 /!\ There is just one known problem. The Linux system layout differs markedly from the Squid defaults. The following ./configure options are needed to install Squid into the Linux structure properly: 

{{{
 --prefix=/usr
 --sysconfdir=/etc/squid
 --bindir=/usr/sbin
 --sbindir=/usr/sbin
 --localstatedir=/var
 --libexecdir=/usr/sbin
 --datadir=/usr/share/squid
}}}

== Troubleshooting ==


----
CategoryKnowledgeBase SquidFaq/BinaryPackages
