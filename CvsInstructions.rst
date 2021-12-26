#language en

<<TableOfContents>>

== Next-Generation Source Control solution ==
/!\ a wide consensus has been reached towards replacing CVS with another more modern Version Control solution. Please see [[Squid3VCS]].


== CVS access to current Squid source ==
## If you need to get CVS, start at [[http://www.cvshome.org/|CVSHome]].

## If you need to learn about CVS, read this great
## [[http://www.network-theory.co.uk/docs/cvsmanual/|reference manual]].

To checkout the current source tree from our CVS server:
{{{
  % setenv CVSROOT ':pserver:anoncvs@cvs.squid-cache.org:/squid'
  % cvs login
}}}
When prompted for a password, enter 'anoncvs'.

{{{
  % cvs checkout squid3
}}}
You can use {{{ cvs -d :pserver:anoncvs@cvs.squid-cache.org:/squid checkout squid3 }}} for a shorter all-in-one method that wont set environment variables.

If you make automake related changes then you will need to bootstrap your tree -
{{{
sh bootstrap.sh
}}}

This may give errors if you don't have the right distribution-time dependencies (libtool, automake > 1.5, autoconf > 2.61).

To update your source tree later, type:

{{{
  % cvs update
}}}

== Browsing the CVS sources ==
To view the CVS history online and browse the current sources use the [[http://www.squid-cache.org/cgi-bin/cvsweb.cgi|CVSWeb interface]].

== Access to CVS developer branches (obsolete) ==
Many works in progress are hosted in our public [[http://devel.squid-cache.org/CVS.html|developer CVS]] repository. Some further information for developers and testers is on the developer site at http://devel.squid-cache.org/

The experimental code CVS repository and server is kindly hosted by [[http://sourceforge.net/|SourceForge]]

The Squid Development projects CVS server and can be reached both anonymously using pserver, online on the web, and using ssh (registered developers only).

==== SSH (requires a registered SourceForge account) ====

{{{
  export CVS_RSH=ssh

  cvs -dYOURUSERID@cvs.devel.squid-cache.org:/cvsroot/squid co -rBRANCHNAME -kk -d squid-BRANCHNAME squid3
}}}

 (watch out for the line wrapping)

==== Web ====
http://squid.cvs.sourceforge.net/squid/squid3/

==== Anonymous pserver ====
{{{
  cvs -d:pserver:anonymous@cvs.devel.squid-cache.org:/cvsroot/squid login
}}}
 ...blank password, just press enter...
{{{
  cvs -d:pserver:anonymous@cvs.devel.squid-cache.org:/cvsroot/squid co -rBRANCHNAME -d squid-BRANCHNAME squid3
}}}

All development in this repository takes place on branches with automatic change tracking from the master version using the scripts described below.

Note: If you are looking for the main Squid sources please see [[Squid3VCS]] and use the Bazaar instead. 


== Access to older Squid version ==
To access older Squid releases use the same procedure as above to login and then checkout the specific version sources

Squid-2, please use this when submitting patches etc

{{{
  cvs checkout -d squid-2 squid
}}}

Squid-2.7.STABLE, for tracking the current STABLE release.
{{{
  cvs checkout -d squid-2.6 -r SQUID_2_7 squid
}}}

----
CategoryObsolete
