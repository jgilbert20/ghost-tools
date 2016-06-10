# Ghost Tools

Simple zero-infrastructure tools for safety managing large collections of your digital files, inspired by git and related tools.

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





