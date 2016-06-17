

# Cool tools to check



http://superuser.com/questions/730592/compressing-many-similar-big-files/730598
https://git-annex.branchable.com/future_proofing/
http://ck.kolivas.org/apps/lrzip/README

## git annex

git annex is cool! i want it to do what i want! but it doesn.t its stuck doign
everything like a big git repository that knows how to shuffle files around.
you have to manually add things into the annex and commit them. My style is
more to just put files around in any place I want them and then worry about
redudancy later. Annex is nice though becuase it seems to provide closure
around a set of "managed" files so there is no confusion as to what your
archive looks like no matter where the peices are scattered.

## bup

and bup is way cool too, but also seemingly tied to being a backup tool. But
its got way awesome large file saving and incremental backup support.

https://github.com/bup/bup
https://raw.githubusercontent.com/bup/bup/master/DESIGN
http://blog.wrouesnel.com/articles/bup%20-%20towards%20the%20perfect%20backup/
https://news.ycombinator.com/item?id=8620236

## Attic

https://attic-backup.org





## brackup

written in perl
nice tool, chunks things

    Path: 2015-04-18/jag-150418-16062.CR2
    Size: 22777072
    Digest: sha1:ef565146db40252e51e2cf285fe7aa334a90ec77
    Chunks: 0;5242880;5242880;sha1:7334730d2ac5326005b57b64ee15818957d80ec6
     5242880;5242880;5242880;sha1:7d19536ec4d93f40833d74599d318dfe9794a6f8
     10485760;5242880;5242880;sha1:08ca60a1cc2fcc12809e361cd6b9f94900b9ef05
     15728640;5242880;5242880;sha1:6a1d1b7cefe2f96743ef71e8d67ff5b0f49a52e8
     20971520;1805552;1805552;sha1:b1f665601efdf64354eb7541a3f07e38066c9762
    Mtime: 1429361654
    Atime: 1465732163


