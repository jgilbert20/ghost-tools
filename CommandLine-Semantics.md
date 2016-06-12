# Command Line Semantics and Use Cases

Use case: Keeping in progress backups locally for reverting

Scenario: While on the road, you routinely work on a set of files, some of them very large. The baseline copies of these files are backed up at home, but sometimes in the field you’ll make changes and want to be able to go back and look at previous versions. These files are together 1 GB, so your backup directory will start at 1 GB.

First, you’ll want to create a local baseline

	gfs vol create ~/backups/ vol:local-backups:
	# Sets up ~/backups as a place to hold local backups

The syntax above with vol:local-backups: is interesting. The tripath you see above is the way that GFS refers to virtual filesystems.

The tripaths act as logical file names (LFNs), and let you refer to bigger concepts of time, space, place etc with file-name like consistency.

In this case, the LFN

	vol:local-backups

Tells GFS you are referring to a volume. Volumes are nice because they can be relocated to other places, split, virtualized, etc and you don’t need to worry about them.

The syntax is:

	[handler] : [root] : [path]

Where handler refers to the object type, root describes the name of the object, and path refers to the subpath inside the object.

There is no obligation to use this syntax. GFS works just fine if you would like everything just to have a plain old UNIX path. But the special syntax enables you to do things that aren’t normally possible. In the case of a volume, like we are about to use, the volume keeps track of its contents even when they are intentionally (or unintentionally) moved around, or even when the mount point is different. 

So bear with me for a second and just try this:

	gfs vol create ~/backups/ vol:local-backups:

We’ve now established “vol:local-backups” as a writeable place, almost like a pointer, to the directory ~/backups/.

Now, tell GFS you want to use the new volume local-backups as the default location for backups.

	gfs backup default vol:local-backups

GFS is all set to run backups! THat simple!

Now on a periodic basis, you’ll want to run an incremental backup. When you run this next command, anything in ~/work-dir that is not present in the local-backup store will be saved away. 

	gfs backup run ~/work-dir
	Created ~/backups/backup-20160502/work-dir.snapshot

Backups are fast. GFS is doing all of the usual things to check for modified files. And it never stores duplicate files. If you had 10 files with the same contents in ~/work-dir, it will copy exactly ONE file over into ~/backups. 

How do you get back the older version of your files?

Whats nice about GFS is that at no point does it create byzantine proprietary files. (Rule #1: You will still love GFS even if you decide to stop using it.) The simplest way is just to open ~/backups using your file exploring tools of choice, and grab the files you need.

Inside, ~/backups, you’ll find directories like this:

	~/backups/backup-20160502/work-dir/.... 
	~/backups/backup-20160503/work-dir/.... 

Inside each folder is everything that had changed on that particular date. The only think you have to watch out for is that a duplicated file is never stored twice, so you might not find every file necessarily where you’d expect it. But GFS has tools to reconstruct those missing files if you want them. 

However, simple file access to the “delta” directories is not particularly satisfying. We might prefer to actually reconstruct a copy of work-dir on a particular date, duplicates and all. Or we might want to do a diff between two arbitrary versions of your file tree, sort of like git. Or maybe we just want to look at the version history of one file.

Lets take each of these scenarios in turn. Lets say we want to get everything that existed in work-dir on May 3, 2016 as a copy. First, we invoke a tool to check what backups exist for this path

	gfs backup lshistory ~/work-dir
	Using [vol:local-backups] as backup repository
	Mon May 03 11:55:39 - vol:local-backups:snapshots s20160503.snapshot
	Mon May 02 08:12:01 - vol:local-backups:snapshots/20160502.snapshot

Now we can tell GFS to make a copy of the earlier snapshot in its entirety.

	gfs backup restore vol:local-backups:20160502.snapshot ~/dir

This will make a COPY of the working directory, exactly how it existed on May 2. Performing this operation will consume about 1GB of extra storage space, because we are copying things out of the backup repository, and the working directory was 1GB. In the process, duplicate files will be recreated, because, well, user expectations! 

Here is a more advanced use case -- maybe you just want a directory that has symbolic links with the 21060502 snapshot. This will use no extra disk space, instead you’ll get a tree that looks like you’re working directory, but every file is actually a symbolic link.

	gfs backup restore --symbolic vol:local-backups 20160502.snapshot ~/dir

Of course, you’ll want to be extremely careful not to modify those files in ~/dir, since you’d be changing the actual files of your backup.

# Voodoo

Now, try something more fancy. Lets say your hard drive is filling up with all of these backups. How can we move some or all of them to another location?

Most backup tools don’t really like splitting up archives, but it turns out that GFS doesn’t really care. You can move its backup files around completely at will. As long at the data is still physically somewhere, it continues to work. 

For instance, lets mount a USB flash drive to /Volumes/usbstick. You could literally just use rsync(1) or cp(1) or even the OSX finder to drag some of those backup directories out to another disk. 

As long as you don’t screw with the files in ~/.gfs (where GFS keeps lightweight metadata), it doesn’t matter what you do, GFS is going to remember the things it backed up and what they looked like and undo your chaos later.

Lets take a baby example. Lets say you’ve got a 100MB photoshop file here:

	~/work-dir/big-file.PSD

And lets say that every so often you back it up to some new location using GFS.

	gfs backup run ~/work-dir/big-file.PSD
	Created [snapshot-20160502-fce2e2]

All of your old versions are just sitting on disk in directories like these

	~/backups/20160502/work-dir/big-file.PSD
	~/backups/20160503/work-dir/big-file.PSD
	~/backups/20160504/work-dir/big-file.PSD

You can get a report on this situation easily. Just give the lshistory command.

	gfs backup lshistory ~/work-dir/big-file.PSD
	# stuff here

If you want, you can move these backed up files to some new location, like a flash drive. You don’t have to use any special tools. Just move them somewhere else and delete the originals that GFS made for you.

	mv ~/backups/201605*/work-dir/big-file.PSD /Volumes/usbstick

Belive it or not, GFS is perfectly fine with this. YOu still have valid backups, just in a different place. The key is that GFS needs to learn you’ve copied those files. Its philosophy is “do what you want, I’ll catch up later”

Of course, GFS isn’t a mindreader. It doesn’t run a damon or a background process. It has no idea you’ve just done this henious act. As far as its concerned, those backups don’t exist anymore. 

But if you tell it to index things, it will figure it out. Index is a very fast operation.

	gfs index /Volumes/usbstick
	# Takes 0.2 seconds

Now GFS knows that backups of big-file.PSD are present on your USB stick. Yes, it’s that simple. That’s all it took! You’ve just created a distributed backup system!

	gfs backup lshistory ~/work-dir/big-file.PSD
	# WOw, loook, it found it!

GFS accomplishes all of this magic by keeping track of file checksums in a very efficient way. The checksums themselves are almost never performed redunantly, so the actual time spent is very low and mostly on “up front” when a file is first seen. Incidentally, this penalty is no different from git, git-annex or any other tool.

If you used ‘gfs mv’ rather than ‘mv’, you wouldn’t even need index anything. If you use GFS’s move commands, GFS would see you were moving around your backup files and update everything behind the scenes.

Of course, GFS’s awesomeness is that you don’t need to make repositories in order to get the magic. It just always happens. GFS can figure out what you’ve done and recover very easily. In fact, that is a essential design criteria of the software.

And it catches up exceptionally fast. GFS rarely checksums a file twice, even if you move directories around. So indexing things is radically fast, so fast that in many cases, it doesn’t do much more than stat each file.

The “penalty” for this extra speedy awesomeness is that GFS likes reliable places to store metadata. Its default location is ~/.gfs. This directory contains a ton of pointers to your files.

This meta data is exceptionally low overhead. Its on the order of 100 bytes per file, uncompressed. Which means that for a system of a million files, you’re wasting about 100 MB. Which is really not very much at all. And that 100MB compresses typically 1:10.

But the metadata isn’t even all that critical. In most cases, GFS can reconstruct it even when it goes missing. Which brings us to the next question. 

# Is metadata evil?

“Wait! Metadata! I don’t like that,” you say. “I want the freedom to not have extra sticky metadata directories that i have to back up!”

Bear in mind you don't ever need metadata. File backups always look like real files on your file system. So throwing away GFS is never going to cause you to loose data or backups. 

But some people don't like metadata at all. More stuff to keep track of.

We feel your pain, brother. 

So here is the real miracle: GFS can *recover* its metadata automatically. Even if you blow away ~/.gfs, it can usually reconstruct the history of your digital life. It does this by spreading your metadata around in different places. It writes spare copies of its metadata into little files called ‘.gfs_ip’ in each of your directories. It also pushes copies of its master metadata repositories into other places, including backup locations. 

This generally means you don’t need to worry about preserving or backing up anything in ~/.gfs, the magic directory where GFS keeps track of things. Even if you take away its special working directory, GFS is generally smart enough to look for other places (like your backup drives), and will start to rebuild its collection of meta information.

Indeed GFS is super resiliant to deletion of its metadata. But what happens if you remove everything? What if you blow away EVERY SINGLE .GFS file? What happens then?

Well, even if you deleted all of that metadata, GFS still works in a degraded way. Its kind of like the T1000 robot from Terminator II, or the velociraptors in Jurrasic Park. It starts to reassemble itself from its broken peices.

# The man behind the green curtain

How is this zero-metadata voodoo accomplished?

Lets go back to the earlier example. YOu have a file on disk here:

	~/work-dir/big-file.PSD

If you’ve been using GFS to back it up, all of your old versions of that file are just sitting on disk in directories like these

	~/backups/20160502/work-dir/big-file.PSD
	~/backups/20160503/work-dir/big-file.PSD
	~/backups/20160504/work-dir/big-file.PSD

Of course, you can move those file around as we just discovered. GFS has no “sacrosanct” backup format. It happens to organize backups into these nice names because it thinks you’d like it  better seeing your backups look that way, but it actually doesn’t care about filenames. Everything under the hood is content-addressable storage.

So say you went in and purged all of the tiny metadata files. Can GFS reconstruct them?

YES!

How?

Say you issue a backup command like this one after removing all of its pointers and dotfiles. What would happen?

	gfs backup run ~/work-dir/big-file.PSD ~/backups

GFS doesn’t actually need metadata to track backups its made. It can infer the correct behavior without knowing what happened earlier. 

In the case of big-file.PSD, GFS is smart enough to go back and look to see if had a hash duplicate of big-file.PSD inside of the ~/backups directory. If it doesn’t, a copy is initiated. If it does have a copy, it will will avoid backing up. It doesn’t need meta-data to tell us if a backup has been made. Of course, it doesn’t know what is in the backups directory anymore because you destroyed its tracking pointers. So GFS will simply reconstruct the information it formally had “on demand” by reading the source files.

The principal GFS always uses is: *The correctness of the operation is not defined by the metadata, its defined by the actual files on the disk*.

Now here is the part you’re either going to love or date. This is the T-1000-like-thing about GFS. 

GFS will slowly start rebuilding close to its original metadata directory as soon as you start using it.

When you run the command above to make a backup into a virgin directory, GFS may have to start checksumming files. When it checksums those files, it will store the metadata about the checksums. Why? Well, GFS hates duplication of effort. Inside a backup directory or repository, GFS does not want a file to exist more than once. So if you load a new file during a backup operation, GFS automatically looks to make sure it doesn’t have a copy of that file already. 

That doesn’t mean GFS has to checksum your file to back it up, at least not necessarily. Lets say your photoshop file big-file.PSD is 230043593 bytes. GFS first is going to check to make sure there isn’t some other file that is exactly 230043593 bytes in the backup directory. If there isn’t, it will copy the new file over and call it a day. Why? Because GFS knows that the only way to have a copy of a 230043593 byte file it to have another file that is 230043593 bytes exactly. No need to checksum anything if you don’t have a file that is 230043593 already. A simple enumeration of all of file sizes in your backup directory proves that a backup is needed.

But lets say there does happen to be a file that is 230043593 bytes already in the directory. Now we have to checksum both files to see if they are duplicates. If they are duplicates, the file is already backed up and no further action is taken. 

In the process of all of this size checking and (rare) checksum operations, GFS is going to learn new things about the files on your disk. It’s going to want a place to store that new information so it doesn’t have to be expensive the next time you ask it to do things. So it going to start writing bits of metadata to prevent having to do the work again in the future.

Eventually, almost all of the source-record verifiable information in your backup directory will be reacquired, and GFS’s metadata will return to it’s (mostly) former state! The major thing that will be lost is lshistory. GFS will loose its understanding that all of these backed up files used to live in the same place. But that has very few implications to the user. 

Another way of looking at this is that most of the metadata that gives GFS its magic is mostly just a cache of things that can be verified on disk. There are only really 3 reasons for this metadata in the first place:

1. Caching data means that GFS is lightning fast - in most cases, it has ZERO unnecessary disk access 
2. Metadata helps when your archives are slow or offline - your source archives might be backups in your sock drawer or on a remote server while you are on a trans-oceantic flight
3. Meta-data helps track certain peices of information such as lineage, splits and pools, which greatly increases it ease of use.

But GFS is designed to be able to start from the ground up in any and all cases. In fact, its test cases routinely blow away its own metadata during the test runs, to ensure it doesn’t rely on anything. 


# Rules syntax in progress


Make /src back up to two different volumes 

    gfs rule add /src/ vol:abc,vol:yzx
    [4] /src/ -> vol:abc,vol:yzx [2 copies]

    gfs run







# The Rules of GFS

Rule #1: GFS is resiliant to file renaming and moving chaos - as long as a backed up  or archived file still exists, GFS doesn’t care WHERE. 

Rule 1b: Being unconventional means hating convention: GFS gives names to things but you’re free at ANY time to change those names with (or without) notice to GFS. 

Rule #2: GFS leaves everything in a familiar state - you can stop using it at any time and still find your stuff

Rule #3: GFS metadata is maximally recoverable - it loves you even if your screw with or delete some or all of its private metadata structures.

Rule #4 (The T1000 rule): GFS’s ultimate source of truth is what it finds on disk - it is immune to metadata deletion. Nearly everything can be reconstructed. 