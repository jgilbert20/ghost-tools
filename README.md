# Ghost Tools

Transparently turns all of your existing servers, laptops, S3 buckets and dropboxes into one big de-duplicating content aware virtual filesystem that can archive, backup, replicate, synchronize and restore. GFS is zero-infrastucture, and respects all of your existing file locations and usage patterns. Inspired by the power of merkle trees and git.

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
- Create online and offline archives that can be partially or entirely locally cached
- Instantly tell you if a file is mirrored in other places and the last time its bytes were checksumed, even on offline storage
- Handle complex copies and file restores by sourcing files from the most efficient and promixal locations (think "mini CDN for my stuff")
- Backup files with super-efficient incremental, delta and baselines
- Snapshot versions of files and directories for recovery, reconstruction or even just bitrot detection
- Instantly search and extract files from zip files, tar files, backups
- Cache nearly any deterministic file operation
- Respect all of your existing file locations and usage patterns

GFS pretends your existing as-is filesystems, cloud storage buckets and servers are all one large content aware file system inside a universal (but very familiar) namespace. This remapping is done transparently without any effort or new filenames to remember. In fact, you won't even be aware of the remapping most of the time. 

Inside this "ghost FS", all of the awesome properties of a content-aware file store are available. 

- Instantly locate any file or directory by checksum
- Rapidly detect filesystem changes
- Easily remove (or restore) duplicate files

Inside GhostFS, we've implemented a set of general purpose tools to handle synchronization, backups, archival, snapshots, etc. Also inside this "pretend" name space are low level primatives that support deduplication, chunking, merges, splits, compression, encryption in a generalized and pluggable way.

But its just pretend. On the outside of gfs, your files are exactly what they've always been, and don't have to move to new locations or new names.

GFS's game of "pretend" is fast and quite resiliant. It transforms your filesystem into a content-aware semantics on a *on-demand* basis. Files are rarely checksumed more than once. And the metadata about GFS transforms is spread around in a highly recoverable (but always deletable) fashion. GFS is principaled that correctness of operations is invarient to metadata -- the correctness is verifiable with actual files on disk. Metadata is used to cache. 

Best of all: GFS imposes no conventions - by default your backups and files, repositories and archives stay as 100% files and directories you can recognize, search and recover at any time. They don't even have to be renamed. That means you can walk away from GFS -- destory some or all of its metadata -- and your backups stay valid and recoverable on a per-file (or even per-directory) basis. 

GFS gives you nearly instant, hyper-efficient incremental backups. It lets you spread your files over many drives with and without redudancy. GFS is indifferent if you use gfs(1) or other more familiar tools (Finder, cp(1), rsync(1)) to move files around. And it does this all without reliance on opaque databases, demons, or persistent processes. Nor does it require creation of "repositories" or "volumes" - if its a writeable directory, it can contain blobs.

Sound too good to be true? The downsides:

- GFS is not complete
- Its primary audience is individuals (maybe small workgroups) without complex permissions and <20M files
- To some, GFS's internals will seem overly complicated (most likely because it's design goals differ from similiar projects.)
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



# Things that work

GFS is in progress. It has many working (but only lightly tested) tools. Most of these tools are exceedingly careful about overwriting data. There are only a few gotchas:

- For debugging reasons, GFS will create some files in the directory you run it to store its databases and object stores. (Basically like an automatic git init.) Once we get to a more mature point, these will live in the user's home directory.

- Some working functions are implemented using the "old way" and others are in the "new way". Both ways work and rely on stable plumbing and are 100% intercompatable with each other. (Thanks in large part to very careful object separation of concerns.)
    - The "old way" was to treat directories as basically part of a pathname. Although ghosts are created for both directories and files, the handling logic is granular to the file. Althought nested files and directories work perfectly with no compromise to integry, they are not as awesome as the case where the directories are first class objects that contain hashes. Operations like rsync, compare, lsdup are built this way. They rely on "ghost stores" to do everything with collections of ghosts. 
    -  The new way is more awesome -- directories are first class objects that can be hashed, etc. In this case, all operations occur at the directory level, and lookups can be uber-fast because GFS can instantly recognize directories that have or haven't changed and can even use its own data structures to avoid directory enumeration. In this architecture, ghosts themselves can operate as first-class containers. One day, the ghost storage architecture will go away. 

## The content aware layer - present status

It works, and so far, its prety stable. Every file GFS discovers in its work is added to this layer. And when changes are discovered, invalid data is revoked. For instance, if you use GFS to move or copy files into a directory, its will automatically revoke the checksums for the parents. If you rename files, same thing. 

Furthermore, it will recognize that files have become deleted and remove those hash layer. (Test this to be sure!)

## The VFS layer

Most testing has happened at the fs layer. SO just use pathnames as usual. Behind the scenes, all of your paths are re-written into a cleaner, virtual file path.

    myfile.txt

becomes

    fs:root:myfile.txt

You can also specify objects by their hash with a carret.

    $ gfs checksum test
    SHA1/FULL/f1d2d2f924e986ac86fdf7b36c94bcdf32beec15  fs:root:test
    $ gfs ls ^f1d2d2f924e986ac86fdf7b36c94bcdf32beec15
    f   SHA1/FULL/f1d2d2f924e986ac86fdf7b36c94bcdf32beec15  Sat Jun 25 18:27:51 2016(3.81 min)  ^f1d2d2f924e986ac86fdf7b36c94bcdf32beec15

Although its worth noting that currently GFS will take any object with that hash at random. It could be something in its store or checkpointed, etc.

You may have to escape the carret depending on your shell.

Ready for even more awesomeness? There is some partial addressing to the object store. Examples:

    $ echo test > bar
    $ gfs store bar obj:core:check-it
    # this takes your file and puts it into the object store 
    # and assigns a pointer

You can also extract things this way

    $ gfs store obj:core:check-it newfile
    $ diff test newfile
    $







## Checkpointing

You can checkpoint a directory. Doing this makes sure that the state of each objects is recorded and that GFS is aware of it. Once an particular version of a file is checkpointed, GFS can reach it with its hash lookups 

    gfs checkpoint dir-a
    fs:root:dir-a: 58   2   SHA1/FULL/18dae1e68a91393530c0b49ac32a42e1b63e6e1a

This says there are 2 files, consuming 58 bytes and the hash of the directory is '18dae...'.

## Rsync

A very clever RSYNC operation that breaks down its work into efficient operations: copy, haul, local copy, etc. It also does all of this in a special transform object that self checks each operations before they are performed (it basically runs a consistency check on its file operation plan as a double check.)

    gfs rsync dira dirb

Its not well tested, but it does seem to work. The only problem is that its sort of tied down to the old "file" architecture and doesn't do smooth cool stuff with directory hashes. (Basically that just means it will check every single file even if it doesn't need to. But that still will usually be blazingly fast due to caching.)

## List Duplicates

The formatting isn't pretty, but you can list duplicates inside a directory. This is done in a super efficient manner that does not require any unnecessary checksums. 

    $ gfs lsdup .
    Duplicate report: size X (target dups / source dups ) filename
    607405 X (1/0) fs:root:stuff-to-backup/screen1.png 
    607405 X (1/0) fs:root:stuff-to-backup/screen1-duplicated.png 
    13928 X (2/0) fs:root:hasDups/file1 
    13928 X (2/0) fs:root:file2 
    13928 X (2/0) fs:root:file1.gsfmoved-20160614 
    12288 X (1/0) fs:root:small2/__gsf_dbd/hash-to-afn.db 
    LSDup finished
    I found 24 files that were duplicated in target (.),
    when searching among the target plus source files: 
    occupying 1341886 unnecessary space across the whole filesystem over 134 files.
    Within the target itself, this represents 1341886 bytes.
    There were 50 in the target directory

Lsdup doesn't care (today) if you checkpoint or not. It is using some of the older code that will descend into directories and hash up a list of all of the files. 

## Comparing trees

Another neat little tool that tells you if two directories differ, and how.

    $ gfs compare dir-a dir-b
    fs:root:dir-b -> Spurious
    fs:root:dir-b/dir-a -> Missing
    fs:root:dir-b/dir-a/a -> Missing
    fs:root:dir-b/dir-a/b -> Missing
    fs:root:dir-b/test1 -> Spurious
    fs:root:dir-b/test33 -> Spurious

It does this by exercising the ghost transform functionality, which is identical to rsync. Once again, its a file-by-file operation, and doesn't use the awesomeness of directory hashes. But it does work, and doens't have any known bugs (at least not in its light testing.)

## Store

You can copy files in a 100% ghost-aware way. Doing this automatically updates the ghost databases and metadata and also has the benefit of not copying files that are already there.

    # date > quxx
    $ gfs store quxx foo
    $ diff quxx foo

## Move

You can also move files. Like store, this fully uses the content aware layer. 


## Plumbing operations

There are a few great little tools that show the content engine at work. These are also all working as well as many others in the debug section:

    gfs debug hashinfo <hash>

    gfs debug write-aln "text to write" file.txt

    gfs debug mkdir directory




    # Set foo object to point to bar 
    # (pointers aren't used for much right now, but that will change.)
    gfs pointer foo bar





# Swiss Army Knife Description

GFS is an overlay snapshoting and directory reconstruction tool. Its not really
meant for version control, but managing someone's life which may be scattered
across many different asset types and locations. GFS is entirely built around
command line tools. It doesn't watch the filesystem for changes.

key features
1) highly accelerated content-aware rsync - a file is never retransmitted unless needed

2) worried you still wantt o eb sure you have a backup of a wedding photos? 
    this has tools that let you flag and manage bit-rot and verify your stuff is where
    you think it is

3) work with large file trees both on a desktop and laptop? GFS can tell you instantly
    which chnages you've made on your laptop that are not yet on your desktop and push them
    to the desktop efficiently
4) in the field and want to be sure files you are working on are protected?
    GFS can make sure any file is either in your dropbox, on your machine at home
5) trying to free up space on your laptop? do you have a copy of that big 10 GB ISO file
    at home? find out quickly if you can ditch it. 



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





