
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

## Duplicate trees

This is really useful and really fast. Prints the % of duplicates both in count and bytes.

    apollo:~/Dropbox/work $ gfs duptree /Volumes/Drobo03/work . --report-depth=1
    D:GS: GS-Generating size cache: fs:root:/Volumes/Drobo03/work/
     99% files 100% bytes :fs:root:work - Retired From DB
      0% files 100% bytes :fs:root:work - WORKING
      5% files 100% bytes :fs:root:work Transition
      0% files 100% bytes :fs:root:work transition 2
     74% files 100% bytes :fs:root:.


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

