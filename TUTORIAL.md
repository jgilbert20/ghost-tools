# Starting out with GFS

This tutorial will show you how to get started using GFS to examine, backup, archive and manage your files.

Lets start with getting the conceptual understanding of how GFS works. First, lets start with a simple command called "ls". This command prints out a report about the files you give it, telling you what, if anything, GFS knows about them.

Lets say you have a working document of important files:

    jeremy/work
    jeremy/work/presentation-to-merck
    jeremy/work/presentation-to-merck/rounds-v1.ppt
    jeremy/work/presentation-to-merck/rounds-v2.ppt
    jeremy/personal/
    jeremy/personal/wedding-photos/photo1.jpg    
    jeremy/personal/wedding-photos/photo2.jpg  [...]

Lets go into the jeremy directory and we'll start by issuing a simple GFS command. (This example assumes GFS is a fresh install, no changed config settings or cached data loaded.)

    cd jeremy
    gfs ls
    333
    333
    333

Interesting, isn't it? LS is showing us NOTHING about these files. Just so we are all clear, that is CORRECT. We can count on ls to avoid most disk operations and give you a picture of what GFS knows. In this case, GFS has never seen these files before, so this report is correct. Ls, by default, is lazy.

So how do we start doing content-aware operations? GFS has to learn about the directories you work with. There are two ways for it to learn. First, it can learn incrementally. As you work with GFS and touch new directories and files, it is constantly taking in new information and caching it into its information store(s).

There is also a way to do this all in one go. You can checkpoint your files in one go using the checkpoint command.

    gfs checkpoint .

Checkpoint scans the entire directory tree, making sure everything it finds there is incorporated into the virtual, content-aware filesystem. None of your files move, but now they fully characterized for future operations.

I want to emphasize that unlike tools like bup and git, checkpointing is NOT required for correctness. GFS is really smart. If you perform an operation that requires data to be checkpointed, hashed, etc, than it will automatically do this step. 

Now when we do an LS, we suddenly see new things
    
    gfs ls

By default, GFS does the bare minimum of work to get the answer you're looking for. And "ls" is especially lazy - it deliberately doesn't force anything to be calculated or understood.

# Checksums and Caching

Lets try another simple command. The "checksum" command tells you the SHA1 hash of any file you point it at.

    gfs checksum jeremy/work/presentation-to-merck/rounds-v1.ppt --stats
    333sdf
    0 files reads

In this case, we've also asked GFS to tell us the statistics of the files its read and written. Here you can see that the tool has actually not read anything from disk. The previous operation to checkpoint the file has cached the information. GFS, being the lazy thing that it is, has refused to even stat the file again.

This is considered a virtue of GFS. When you are performing command line operations on extremely large files and directories, you don't want things to be repeated. GFS is optimized for interactive use - cases like you are checking around your file system and looking for information, finding duplicates etc.

GFS has extremely strong integrity checking on top of its cacheing. It stores WHEN a checksum or stat was performed. You tell GFS what amount of recency you prefer. The easiest way to do this is with the -G option. The guard time option basically says how many seconds old the information that GFS relies on can be.

    gfs checksum jeremy/work/presentation-to-merck/rounds-v1.ppt --stats -G 0
    1 full checksum

Now you can see that despite its caching, GFS has forced a read from the file and recaclulated the hash.

GFS also has a concept of revocation of invalid data, which results in escalation of cache voids. For instance, lets change the size of a file.

    date > simple.txt
    gfs checkpoint simple.txt
    gfs ls simple.txt
        size and hash

Now, lets perturb things

    date >> simple.txt

Normally, GFS stats any file that it has not statted in the last 60 seconds. If we are quick enough, we should be able to pull stale information out of the stat cache. As long as you perform this next command within a minute, you'll actually get WRONG data.

    gfs size simple.txt
        WRONG data

Wow! The data was wrong! Why? Because GFS trusts any stat to remain contant for 60 seconds.

Why is this a virtue? Well, not all disks are fast. I'm writing this text file now on MacBook Pro with a 1TB SSD. File operations are blazing fast. But if it were a slow ssh connection or a slow old disk, I would want my file scanning tools to be more conservative. And frequently we back things up to old drives. Furthermore, GFS is intended to work on millions of files at once. You wouldn't want it unnecessarily stating a million files.

Using the -G flag, we specify we don't want lazyness:

    gfs size simple.txt -G0

# Revocation

GFS is smart about consistency of its caches. When stat(2) information (filename, size, modification date) is older than 60 seconds, it loads it again from disk to make sure nothing has changed. Usually, files don't change and GFS will notice that its data is still correct. 

However, when things are changed, GFS is careful to revoke information htat is now wrong. If you stat a file and the size has changed, it knows that any checksums have changed too. It will very quickly revoke all of the information and avoid showing you anything wrong.

Now, GFS has been forced to get a new size for simple.txt from disk. In the process, it discovers that the file has actually changed (its size is different.) Now, GFS will toss out all of the information that relied on simple.txt being unchanged.

If we do an LS, we can see that the hash has disappeared:

    gfs ls simple.txt
        size and no hashed

# Consistency

Because GFS will always take in new lightweight stat data every 60 seconds, and it revokes data, it means that you generally can count on file level data to be correct within a 60 second window.


# Simple archival and bitrot protection

Lets take on a very simple archival case. We want to be sure we don't experience bitrot or changes to our precious wedding photos. 

    gfs snapshot personal/wedding-photos/ -n "wedding"
    wrote wedding.snapshot

wedding.snapshot is now a text file that contains the exact state of the wedding-photos directory. This contains all checksums, file sizes, mdoification dates, etc. Its not a backup -- its a verification file that you can use later to see if anything has changed.

How can we use this? The simplest way is the "recon" verb.

    gfs recon-test personal/wedding-photos snap:wedding.snapshot
    OK

The "recon-test" verb checks to make sure that the exact state of your snapshot can be reconstructed from the directory. If it returns "OK," it means that your data is safe. The exact set of bits and bytes that were available at the time you made the snapshot can be rebuilt by the files you have.

Bingo! You've just checked that there is no bit-rot! Your photos are safe.

One interesting thing is that the "recon-test" doesn't care about fungible changes. Maybe you renamed files in "personal/wedding-photos" or moves them to different places. To a content aware virtual filesystem, these changes don't matter. Reconstruction of the original directory is possible as long as the metadata and the blobs behind the files are still present. In theory, you could rename the files back, and set the modification dates and permissions back to the way they were in the snapshot. No data has been lost.

# Rolling back time to a particular snapshot

GFS in fact can perform this reconstruction at will. It can take any directory of content, and reconstruct the snapshot exactly the same way it used to be, using the original directory as a set of blobs.

    gfs reconstruct snap:wedding.snapshot personal/wedding-photos dest/

Now the dest subdirectory contains exactly the same files from the snapshot. GFS takes the first thing you give it as a pattern, and the last thing you give it as a destination. It will now use the files listed as sources, reconstructing your original directory.

In essence, snapshots give us a very primordial backup and restore capability. They can return a file system to a previous state, as long as the underlying data is still around somewhere.

When reconstructing snapshots, GFS actually looks for any proximal data. It starts with the places you've provided to "source" the files. But it is happy to find any other checkpointed file. 

As long as you frequently checkpoint your working directories, GFS can often reconstruct old snapshots without any fuss. Unlike most other tools, GFS doesn't care where the blobs to reconstruct a snapshot might come from.

# Backing things up

Snapshotting and reconstruction provide some primitive operations but don't actually manage backups. Backing up typically means you take files and push (or pull them) to a remote location. There are many tools like that. In fact, rsync() is a good start, and tools like duplicity can even create very credible incremental backups doing nothing more than rsync. 

A de-duplicated backup system, like GFS, takes things one step further. A blob only needs to exist once on the backup repository to count as a backup. The GFS backup tool operates with that rule. 





