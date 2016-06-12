# Ghost Tools

Simple zero-infrastructure tools for safety managing large collections of your digital files (backups, archival, replication, etc), inspired by git and related tools. Instant content-awarne file operations without any hassle.

Imagine all of your EXISING, as-is filesystems and organization schemes that you use on a daily basis were magically, natively content-aware and you had drop in replacement tools for things you do now that leveraged content awareness for backup, archive, snapshots, synchronization, etc. This is the premise of "ghost FS".

Why? Recently, there have been some awesome proliferation of de-duplicating, content-based storage tools like bup, attic, camilstore, git, git-annex, etc. But these tools are usually confined to backup or version control even though the same principals can easily work for archiving, file search, distributed storage, caching, and even just regular syncing files. 

So why not build a general purpose tool that does all of these things? And lets make it work without any hard requirement for metadata servers, SQL databases, redis, etc. In other words, just a simple command line utility.

Along the way, we'll also get rid of the annoying assumptions and conventions that these other tools sometimes require (like .git directories or backup descriptors) so you're always free to break storage conventions, make your own backups, and abandon, ignore (or return to) GFS at any time with minimum breakage.

GFS's philosphy is "Go ahead without me. I'm smart -- I can catch up later."

	gfs rsync workdir remote-dir
	gfs rsync workdir jgilbert@server.com:remote-dir

	<mess around with stuff on remote-dir using cp/mv>

	# just works
	gfs rsync workdir jgilbert@server.com:remote-dir


In a familiar, UNIX-like way, GFS can:

- Rsync files and directories without ever transmitting a single unnecessary byte, even when files and directories are renamed or re-ordered behind its back
- Create online and offline archives that can be partially locally cached
- Instantly tell you if a file is mirrored in other places and the last time its bytes were checksumed
- Handle complex copies and file restores by sourcing files from the most efficient and promixal locations (think "mini CDN for my stuff")
- Backup files with super-efficient incremental, delta and baselines
- Snapshot versions of files for recovery later
- Instantly search and extract files from zip files, tar files, backups
- Cache nearly any deterministic file operation

GFS pretends your existing as-is filesystems, cloud storage buckets and servers are all one large content aware file system inside a universal (but very familiar) namespace. This remapping is done transparently without any effort or new filenames to remember. In fact, you won't even be aware of the remapping most of the time. Inside this "ghost FS", all of the awesome properties of a content-aware file store are available, and we've implemented a set of general purpose tools to handle synchronization, backups, archival, snapshots, etc. Also inside this "pretend" name space are low level primatives that support deduplication, chunking, merges, splits, compression, encryption in a generalized and pluggable way.

But its just pretend. On the outside of gfs, your files are exactly what they've always been, and don't have to move to new locations or new names.

GFS's game of "pretend" is fast and quite resiliant. It transforms your filesystem into a content-aware semantics on a on-demand basis. Files are rarely checksumed more than once. And the metadata about GFS transforms is spread around in a highly recoverable (but always deletable) fashion. GFS is principaled that correctness of operations is invarient to metadata.

Best of all: GFS imposes no conventions - by default your backups and files, repositories and archives stay as 100% files and directories you can recognize, search and recover at any time. They don't even have to be renamed. That means you can walk away from GFS -- destory some or all of its metadata -- and your backups stay valid and recoverable on a per-file (or even per-directory) basis. 

GFS gives you nearly instant, hyper-efficient incremental backups. It lets you spread your files over many drives with and without redudancy. GFS is indifferent if you use gfs(1) or other more familiar tools (Finder, cp(1), rsync(1)) to move files around. And it does this all without reliance on opaque databases, demons, or persistent processes. Nor does it require creation of "repositories" or "volumes."

Sound too good to be true? The downsides:

- GFS is not complete
- GFS's test suite is not complete (about 5% done)
- GFS is not built on git (even though we admire and love git, its not right for our use cases)
- GFS is fast because it is highly reliant on lazy caching, which doesn't suit some software developers who want formal deterministic behavior from their caching
- Remote stuff not implemented yet (but already designed and easy to implement)
- Its open source, so you'll get what you pay for (or are willing to fork)
- Its internal concepts could be hard to grok since they emphasize generality
- Some of the more esoteric functions like chunking, rdiffs, etc are not implemented yet.
- There is no FUSE / local mount support yet (but implementing this is trivial)
- GFS is written in a portable version of perl 
- Because the metadata file formats are not yet stable, new versions of GFS may run with degraded performance for a while
- GFS can be RAM intensive for really massive file operations. (300-2000 bytes per file)




# Rationale






# Introduction

Ghost Tools (gfs) is a maximally unobtrusive utility that manages universal state and locations of files you care about. This means that as files are scattered about in your life, across different drives or storage providers, GFS knows where things are gone and how to get them back. It has been especially designed to avoid any damons, persistent state, databases or cloud systems. Everything it does will continue to work no matter what kind of large axe you take to it’s “private” files.

The first rule of GFS is that every file or thing you want to protect continues to live exactly where you have it now. 

Lets take a simple example. I store my main photos in the following directory:

	apollo:~/PhotoFSX/LR-PhotoFS-LT $ ls -l
	total 0
	drwxr-xr-x+    20 jgilbert  staff     680 Jun 13  2015 2015-04-18
	drwxr-xr-x+     4 jgilbert  staff     136 Oct 10  2015 2015-10-09
	drwxr-xr-x+    28 jgilbert  staff     952 Feb 27 10:41 2016-02-27
	drwxr-xr-x+    31 jgilbert  staff    1054 May 21 09:44 Isabella 2014 Part 05
	drwxr-xr-x+    16 jgilbert  staff     544 May 21 09:49 Isabella 2015 Part 06
	drwxr-xr-x+     5 jgilbert  staff     170 May 29 21:06 New Hampshire Drone May 2016
	drwxr-xr-x+     3 jgilbert  staff     102 May 21 10:02 Rejects
	drwxr-xr-x+     8 jgilbert  staff     272 Apr 20  2015 Rejects to move 2015April
	drwxr-xr-x+   556 jgilbert  staff   18904 May 21 09:48 San Deigo Family 2015
	drwxr-xr-x+    92 jgilbert  staff    3128 Nov  8  2015 Vermont 2015 Unsorted
	drwxr-xr-x+    13 jgilbert  staff     442 Jun 15  2014 cambridge
	drwxr-xr-x+ 10132 jgilbert  staff  344488 Nov 11  2015 iPhone MASTER
	drwxr-xr-x+     2 jgilbert  staff      68 Aug 17  2013 iphone


As you can see

gfs chvolumes

rule #2 - GFS still works even if you blow away its state and still loves you


gfs addvol sdfsdfsdf
gfs checkpoint

gfs verify X, Y
		# tells you what has changed since last checkpoints








# Motivation

If you're anything like me, you've got files scattered in an annoying number of different places. Much of this is compounded by the fact that i am both a tech geek (with lots of old programs, projects etc) and a photo-geek who inisists on saving raw photos and photoshop originals. 

In short, I am a data hoarder. I don't keep everything, but I keep a lot.

In 2016, I have important data scattered over a variety of places because it is impractical to unify it into one place due to mobility:

- My main mac laptop (800GB)
- My main mac desktop (3TB internal) plus a SATA RAID-8TB and a DroboPro-16Tb
- An old Drobo that sometimes I connect to keep spare copies of things
- A few old hard drives on my desk with video render files I probably can delete
- An archive of my files before 2012 at my parent's house (safety net backup)
- An archive of some more recent things at my office across town
- A dropbox folder (using only 100 GB)
- A google drive folder
- An old laptop that I may or may not have fully moved out from 2013
- A Win7/10 gaming machine that may or may not have other files

How do you manage all of this stuff? How do you figure out what is a copy, what is a backup, and what is a primary storage for something? How do you make sure you have a DR plan? What do you do if you have tens of terrabytes of files? And what would happen if you main machines died -- how would you reconstruct your digital life scattered over so many places?

There are a few possible soluitons

- Buy an insanely large cloud storage or backup solution and a cable connection fast enough to upload it
- Be extremely organized and personally disciplined and never make a mistake
- Run your own server farm of owncloud or camilstore or infinite and become your own infrastructure provider

But there is another way. The power of technology!

# Design Criteria

Ghost tool's design critera are unusual and probably won't work for everyone. The key principals that have worked well for my own digital packrat nature and govorn the use of this tool

- Be close to zero-install - everything is a perl script with base CPAN libraries
- Use checksums and careful design to NEVER send anything twice
- Familiar - syntax nearly identical to rsync
- Largely ignore filesystem metadata (tags and permissions) - most people don't care about these things, or use EXIF or XMP
- Require no databases or even meta-files (but if you can stand them, they'll make everything go much faster!)
- Always keep copies of things in organized directories similiar to the way they were found when backed up. That way, if things blow up you can manually recover everything and see your stuff in a familiar way - no proprietary tools needed!
- Everything is driven from command line like git - no servers!
- Paranoia - nothing is presumed to last forever, all storage has provisions for re-verification
- Archival occurs at the file and directory level because those are the units of things that matter to people - data is not divided and scattered to the wind

# Pros and Cons

## Advantages

- Feels a lot like rsync and copy tools but smarter
- Tiny footprint and low maintanance

## Disadvantages

- Not going to easily give you any sort of GUI or web server
- Doesn't provide peer-to-peer trusted storage




# History - Why this tool, why now?

Originally, I created a tool called powercut in 2005-6, back when I was a window's junkie. It did many of these same things but was a design nightmare. I was uninformed by much cleaner examples of content aware file storage like GIT. The design was based off of code I wrote at the beginning of my career to manage file collections for a knowledge management solution called KnowledgeRe@ch. (That old system had to be cool -- it had an "@" sign in the name!). Back before Carbonite and Mozy launched, I wrote a whole distributed backup tool called DSB in Java that I planned to launch as a commercial service. It had a nifty design that would have probably made me some money if I had continued the process of patenting it (I abandoned the pending patent in 2007.) DSB was great because it was self-organizing and fully distributed like Crashplan, and had a zero-trust model for internal replication. (Think a lightweight version Tahoe-FS.) Fast forward a few years while I got my MBA, and I gave up trying to maintain Java code and the servers needed to run it for myself and went back to simple command line tools. 

Now its 2016 and time for a rewrite. I've learned what works and doesn't work in managing file replication and archival, and what the bare minimum of the principals needed to make such things work well. The design principals are clearer to me. And I've had the opportunity to learn what I like and don't like about newer technologies like git, Infinit, Dropbox, etc.

# example

make a copy of some file you care about
here is a simple example of how to back it up and manage it

also make a simple video / GIF tutorial





