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





From this point on, any directory you checksum 


