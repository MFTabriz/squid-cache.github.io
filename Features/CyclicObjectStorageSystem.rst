<<TableOfContents>>

== What is COSS? ==

COSS is a Cyclic Object storage system originally designed by Eric Stern.  COSS works with a single file, and each stripe is a fixed size an in a fixed position in the file. The stripe size is a compile-time option.  

As objects are written to a COSS stripe, their place is pre-reserved and data is copied into a memory copy of the stripe. Because of this,
the object size must be known before it can be stored in a COSS filesystem. (Hence the max-size requirement with a coss cache_dir.)

When a stripe is filled, the stripe is written to disk, and a new memory stripe is created.

== Does it perform better? ==

Yes.  At the time of writing COSS is the fastest performing cache_dir available in squid.  Because COSS cache_dirs can only store small cache objects, they need to be combineds with another cache_dir type (aufs, diskd or ufs) in order to allow caching of larger objects.  Because COSS takes care of the small objects more efficiently, the non-COSS cache_dirs also perform more efficiently because they have a small number of larger objects to deal with.

== How do I use it? ==


You need to run Squid version
[[http://www.squid-cache.org/Versions/v2/2.6|2.6]] or later to be able to run a stable version of COSS.

To configure Squid for COSS, use the ''--enable-storeio'' option (and the --enable-coss-aio-ops to enable async I/O):
{{{
% ./configure --enable-storeio=coss,ufs
}}}

== If I use COSS, do I have to wipe out my current cache? ==


Yes.  COSS uses a single file or direct partition access to store objects.  To prepare a file or disk for COSS you need to run the following command:
{{{
dd if=/dev/zero bs=1048576 count=<size> of=<outfile>
}}}

where:

'''<size>''' is the size of the COSS partition in MB

'''<outfile>''' is the partition or filename that you want to use as the COSS store

== What options are required for COSS? ==

The minimum configuration for a COSS partition is as follows:
{{{
cache_dir coss <file> <size> max-size=<max-size>
cache_swap_log /var/spool/squid/%s
}}}

where:

'''<file>''' is the partition or filename that you want to use as the COSS store (you will need to pre-create the file if it doesn't exist)

'''<size>''' is the size of the COSS cache_dir in MB

'''<max-size>''' is the size of the largest object that this cache_dir can store.  This value can not be bigger then 1MB in the default configuration.

The cache_swap_log option should be set to a directory that squid has write access to.  This is used to store all the swap.state files for all cache_dirs, and needs to be set when using COSS because COSS does not have a normal filesystem that it can store this information on.

== Are there any other configuration options for COSS? ==

COSS partitions have a number of different configuration options available.  These options are:

{{{
block-size=<n>
}}}

This will limit the maximum size for a COSS cache_dir (where the size is calculated as the size of the disk space + the size of any membufs) as follows:

n=512  - 8192 MB

n=1024 - 16384 MB

n=2048 - 32768 MB

n=4096 - 65536 MB

n=8192 - 131072 MB

'''The default value for block-size is 512 bytes.'''

{{{
overwrite-percent=<n>
}}}

This will allow a trade-off between the size a COSS cache_dir will grow to, the accuracy of the LRU algorithm and the amount of disk I/O bandwidth used.  <n> must be between 0 and 100.

If it is set to 0, the COSS cache_dir will always copy any cache hits to the current disk stripe. This reduces the amount of unique data that the cache will store, increases the amount of disk bandwidth used but makes the LRU algorithm work perfectly.  

If it is set to 100, the COSS cache_dir will never copy any cache hits to the current stripe.  This will mean that all objects will be stored exactly once, reducing the total disk bandwidth used, but it effectively makes the disk a FIFO (ie popular objects ony stay in the cache_dir for as long as it takes for COSS to loop back tot he original stripe).

'''The default value for overwrite-percent of 50 is a good balance between the two extremes.'''

{{{
max-stripe-waste=<n>
}}}

This option sets the maximum amount of space that a COSS cache_dir will waste when writing a stripe to disk.  Every time a COSS stripe is written, it will waste up to max-size worth of space.  This becomes a problem if max-size is set to a larger value (eg is max-size is 512K when a COSS stripe is 1MB, up to 50 of the space in that stripe could be written to disk with no data).  max-stripe-waste overcomes this problem by dynamically reducing the max-size value to ensure that only <n> bytes of space will be wasted on each stripe write.

'''The max-stripe-waste option is not set by default.'''

{{{
membufs=<n>
}}}

This option determines the maximum number of stripes that COSS will use to send cache hits to clients.  It is designed to limit the amount of memory that a given COSS cache_dir can cause squid to use.  Once squid runs out of membufs, it starts to move all objects to the current disk stripe, effectively ignoring the overwrite-percent setting.

'''The default value for membufs is 10.'''

{{{
maxfullbufs=<n>
}}}

This option sets the maximum number of stripes that are full, but waiting to be freed that this cache_dir will hold in memory.  Once again, this is a setting to limit the amount of memory that a given COSS cache_dir can grow to use.

Each cache_dir will reserve the last 2 maxfullbufs for cache hits (ie they will only be used when squid runs out of membufs).  This is designed to allow a higher hit rate at the expense of storing new objects in the cache.

'''The default is to leave the maxfullbufs option as unlimited (ie we can always accept new objects).'''


== Store index rebuilding ==

The current (Squid 2.6) COSS implementation needs to scan the whole data-file to rebuild the object index, which happens every time squid is reconfigured or rotates the logfiles. This implies a spike in CPU load when those activities are performed.

== Examples ==
{{{
cache_dir coss /var/spool/squid/coss 100 block-size=512 max-size=131072
}}}

 * This will use a file with the filename /var/spool/squid/coss
 * The cache_dir will store up to 100MB worth of data
 * The block size is 512 byte 
 * Objects that are up to 131072 bytes long will be stored.

{{{
cache_dir coss /dev/sdf1 34500 max-size=524288 max-stripe-waste=32768 block-size=4096 maxfullbufs=10
}}}

 * This will use the /dev/sdf1 partition
 * The cache_dir will store up to 34500MB worth of data
 * The block size is 4096 bytes
 * Objects that are up to 524288 bytes long will be stored.
 * If a given stripe has less than 524288 bytes available, this cache_dir will only accept smaller objects until there is less than 32768 bytes available in the stripe.
 * If the default stripe size of 1MB is not changed, up to 10MB will be used for stripes that are waiting to be written to disk.
