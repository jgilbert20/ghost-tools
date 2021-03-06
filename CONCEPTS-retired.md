# retired concepts

# retired concepts




# questions asked earlier that don’t make sense anymore in the current architecture


+++++

# what is the means by whcih deleted files are recongized?
# in cases of a full scan, someone has to be smart enough
# to know we are doing a scan that is "full" enough to count,
# and to mark ghosts for deletion

# right now, there is no stat verification when writing a snapshot
# snapshot -verify-stat (can find it, size and lastmod match) --verify-
# hash and when those things are in place, anthing that hasn't been
# checked in the last 24 hours are rejected, unless you say --hit-disk
# which means basically ignore all of the  tools and recheck everything











# how do we stick virtual things into the hash?
# for instance, where or how do we store the idea that a .tar file has certain contents
# or that an rsync directory (jgilbert@auspice.net:a/b/c) contains a file?
# almost like a normal path a/b/c is basically an implicit "local:a/b/c"..
# the local handler knows to write .gfs_info files
# whereas the remote one knows not to
# each FS type offeres a get, move, copy, retrieve full checksum, stat, etc function
# so now we can handle S3, Rsync, etc

# how does that tie in with tags?
# how do I say anything under /Volume/Foo should be addressed as FOODRIVE:
# clearly this only matters during AFN conversion
# disktag:FOODRIVE|/one/two/three
#
# if that is the AFN recorded, then the system will think that is there
# we just need to write a record like /Volume/Foo -> FOODRIVE
# and also write a /Volume/Foo/GFS_ID.txt file
# then a dereference just requires that we verify that the /Volume/Foo/GFS_ID can still be reached
# maybe there is a tool like "bless" which does this for you
# so the rel->AFN conversion would either need to rely on this mapping
# or traverse back up to make sure that all ../GFS_ID.txt files were registered

# is a snapshot treated as a file collection as a target? if it is, we have to emphasize 
# that it is a ghost. e.g. you can enumerate, but the fact that snapshot:4444/sdf contains
# this file version could never be used as a proxy.
# so when you "get all AFNs for hash", you'd have to pass to a FShandler for the snapshot



# Database layer use cases
# 1) quickly give me ghost information for a path
#      AFN -> GhostSerialized as absolute
# 2) quickly give me the AFNs associated with a hash or size
#      hash -> AFN 0..n   (needed for LSDup)
#      size -> AFN 0..n   (also possibly needed for LSDup)

# so when something is stored in location X or found in location X, 
# what permanent record is created?
# maybe we create spontaneous snapshot records that can be loaded back in?
#
#    gfs rsync /source jgilbert@asdf:foobar/
# 
# All files written to foobar drops a file in .gfs/receipts/rsync-$$-date.snapshot
# With the note: "This snapshot record was written by GFS on date XXXX"



# References and tags (old ideas)


GFS objects

What about the remote case?

normally i'd talk about files somewhere else like this:

    jgilbert@auspice.net:backups/file-a

    (which is really:)

    jgilbert@auspice.net:/home/jgilbert/backups/file-a

    or

    jgilbert@auspice.net:/tmp/file-b

from the perspective of the local cache on the sending side, the AFN is what?

Ideally, you'd want to know 

    jgilbert@auspice.net:. 

        is the same as 

    jgilbert@auspice.net:/home/jgilbert

but how would i learn anything about that? feels like there is no clean way,
and the system can deal with discrepencies even if it doesn't know this
information. would be nice if there was a way to tell GFS that in the future

ideally when i have a volume tagging process i can say

    gfs volume.add.remote @GRYPHON jgilbert@auspice.net:myvolume

    gfs init GRYPHON

    gfs rsync my-photos/test @GRYPHON

So if we use the GIT strategy of tagging, there is some symbolic link between GRYPHON 
and a data structure with the connection information

@GRYPHON becomes its own virtual path. Assuming that there are no overlaps
@with other paths, or other volumes, the FQAFN would be remote:GRYPHON/my-
@photos/test

technically, the "remote" in this case is superflous because it can be
instantly known that this is a remote path by decoding @GRYPHON. Also we'd
want a way in the future to relocate @GRYPHON to a local path.

so these alias constructs know
    The handler (local, remote, volume, snapshot, etc)
    The remote location if there is one

technically, that means that every alias is a pointer to some object sitting in
an object store (Either one that is "real and dedicate", or one that is "found" )

this implies i can do the following

    gfs alias snap:foo.snap @FOO

    # creates a link saying that FOO -> 423fe15a4757767bb4d2a04702f8572d3421179d
    # and also remembers that 423fe15...21179d is a snapshot

    gfs ls @FOO/bar

    # which as long as I can find 423fe15...21179d, i'll just reach into that snapshot

But my stored FQAFNs.... Should they be:

    snap:423fe15...21179d/a/b/c

Or should they be

    alias:@FOO/bar ?

What if FOO changes in the future? The data that is closest to the "truth" is
the first one. That is also the information that is most reusable. But the second
strategy is probably better because @FOO presumably means something to the user
and the user has some intent to refer to @FOO/bar in the future

this raises another question even more fundamental. what if I do this?

    gfs alias /home/jgilbert/backups @LOCAL_BAKS

and say the content is like this

    /home/jgilbert/backups/file1
    /home/jgilbert/backups/file2
    /home/jgilbert/backups/file3

now I do a scan

    gfs snapshot @LOCAL_BAKS

Technically, I now have three universal names for these three files

    fs:/home/jgilbert/backups/file1 -> checksum 123
    snap:423fe15...21179d/file1     -> checksum 123
    @LOCAL_BAKS/file1               -> checksum 123

I think for the moment, all of these things are OK

A snapshot basically acts like a local alias. and the snapshot is the most permanent thing

if i were to list who has checksum 123, 

I'd want to see 

    snap:423fe15...21179d/file1
    @LOCAL_BAKS/file1

The first one I'd rather NOT see because it's expansion will have an identical expansion

How does this work with objects? Lets assume that when an blob is stored, its checksum is identical to a file that was also stored with that hash.

Implications

    gfs checksum file1
    -> 623fe15...24479d
    # now we know this hash

Next I can copy by hash

    gfs copy blob:423fe15...21179d foo.txt

I go through and find obj: anywhere I can, and copy it as a blob. 

Another form of this:

    gfs copy blob:423fe15...21179d foo/

    # copies into foo/423fe15...21179d.blob

But what is this idea of an object store? The object store theoretically has a name
and a location on disk. And the contract is that anytime you copy a file there, its name is lost and it gets absorbed.

    gfs copy foo.txt objstore:path/to/store
    # takes foo.txt and places it in objstore:path/to/store/obj/42/423fe15...21179d

    # the AFNs involved
    # objstore:/home/jgilbert/path/to/store/obj/42/423fe15...21179d --> 423fe15...21179d

Wait, so what is the point of storing objstore as the "domain" here? what
information does it add to the system?

Interesting pattern - in which cases does the AFN actually "need" a domain?

    Doesn't matter for aliases, regular files (fs:)
    but it does matter for remote
    and it matters for snapshots (i think)
    and its confusing for object stores

for instance, what is the difference between 

    objstore:/home/jgilbert/path/to/store/obj/42/423fe15...21179d
          fs:/home/jgilbert/path/to/store/obj/42/423fe15...21179d

What data is gained by calling something an objectstore in the AFN?

two obvservations now that i've written this out:

1) AFNs really have only three core types:

    fs paths: /home/jgilbert/la-la-la
    containers: tarballs, snapshots, things that contain files and have an addressing scheme
    aliases: which can theoretically cover the earlier two cases

2) The meta-data requirements are different

    fs paths: don't need to know anything
    containers: need to know what expansion formula to use
    aliases: may need to point to either fs, or containers, or remote things

An alias is basically a container. But i need a level of indirection. The data in an alias
could change (hostname updates, path updates), and the namespace has to be valid.

so maybe the best term for a domain is a namespace

a. one name space is limited by files on your computer - world of absolute pathnames
b. another name space is basically ghosts that live under a single file-like-thing
    theoretically that single file like thing could have different "views"
c. another name space is aliases, which are ghosts that are referenced through a LUT

the prefixes used on the command line could be better thought of as access strategies?

consider a case of (b.) above.

AFN: snap:423fe15...21179d/file1 

but i might also have derrivative things

AFN: gunzip:423fe15...21179d     -> whatever the hash is of the thing unziped
AFN: gzip:423fe15...21179d    -> has of the thing zipped

what if the gzip algorithim changes?

well, each row is a ghost. so technically, each ghost has a lastFullHash date, 
and if that gets too old, you'd have to repeat the operation.

An alias can be said to always start with an AT

@FOO/bar

so lets consider a simple case of LS

    gfs ls @FOO/bar

    first, we make @FOO/bar a ghost G1
    then user calls G1->getHash();
    G1 is going to say, "wait, don't ask me, dereference me first!"
    so G2 = G1->dereference();
    two cases now
        case 1: fs alias
            G2 is going to be /home/foo/bar
            so i give the information about G2 and ascribe it to G1
        case 2: snapshot alias
            G2 is going to be snap:423fe15...21179d/file1
    Either way, G2 is something that i can ask a meaningful question about
    there is actually a question if G1 should even get "stored" anywhere

    in the case that i have no idea what snap:423fe15...21179d/file1 resolves as a hash

    when i call obtain "snap:423fe15...21179d/file1"

    the TK handler makes a virtual ghost with that name
    G2->getHash()

    will get passed to the "snap" FSE 

    SNAP FSE says fine, get me an object named 423fe15...21179d
    presumably this thing appears
    and i slurp it in, and then i'll have my LFN cache, and i look up file1

great lets take another case

    gfs ls snap:foo.snap/file1
    I obtain a ghost G1 for "snap:foo.snap/file1".
    G1->getHash();
        this will be a cache miss (obviously!)
        needs to be some logic to deference rather than giving up early
        so next I call G1->getFSE()
        and the FSE call will make me a new explorer connected to foo.snap
        if one doesn't exist already
        this implies some sort of FSE cache or factory
        Now I call FSE->getFullHash()
            well, if i have an AFN cache, a forced conversion to AFN will give me an
                opportunity to remember this answer since it AFN is snap:2343...43432/file1
            but if not..
            this routine will force enumeration of foo.snap, populating a LFN cache
            all of the ghosts will be stored in the FSE maybe for use later
            these ghosts should be marked as "immutable" somehow
                i'll pull out the ghost, and extract the hash from it, and return to caller

great, lets take another case

    gfs checksum tar:foo.tar/file1

    I obtain a ghost for G1 for "tar:foo.tar/file1"
    I ask, is tar:foo.tar/file1 a directory?
    the ghost code won't know yet,
    so it will ask for a stat on tar:foo.tar/file1
    get me a FSE for tar:foo.tar/file1
    this will navigate to a cahce for tar:foo.tar FSE controller
        i'll now generate an AFN and check it
        but assuming that fails
        I untar the file somewhere
        i'll be generating LFNs that look like this
            tar:foo.tar/a 
            tar:foo.tar/file1
        and i'll get the checksum of that file
            i'll make ghosts for the things i find there 
                and the call to checksum it will pass through to that ghost
                    ghost will call me asking for the checksu
                    i'll construct a new path with the temporary directory
                        and hash that file
        if anyone asks for the AFN
        ghost will call the FSE to get the AFN

THis implies that canonocalization and rel2abs, cwd stuff all should live with the 
FSE, not the ghost code!!!

Lets revisit the alais case in more detail

    gfs checksum @mainbackup/neepfile.txt

    I obtain a ghost for "@mainbackup/neepfile.txt"
    initially marked as virtual, b/c i don't know what it is.
    first, i ask "are you an alias"?
        ghost getFSE returns the alias handling FSE
            fse->getFileType("@mainbackup/neepfile.txt")
                FSE will say yes, you are an alias
                    not clear if FSE is a singleton thingy or one per alias
                    might be cleaner if each alias consiered to be its own FSE
    now i say, G2 = G1->dereference();
        ghost will call getFSE->dereference("@mainbackup/neepfile.txt")
            FSE will go to its reference database
                learn that "mainbackup" points to object 423fe15...21179d
                    so now I pull this object from store
                            TK->getGhostForHash( )
                        the object is going to say mainbackup = /Volumes/blah/bu
                        now FSE calls TK->obtainGhostFor "/Volumes/blah/bu/neepfile.txt"
                        return
    great, now I have G2 = "/Volumes/blah/bu/neepfile.txt"
    That clearly is just a regular FS path
    so handled the usual way, (Generate pcip files, etc.)
    not clear how the G2 information gets "absorbed into G1" and committed to disk

another way to think about this is that maybe there is no explicit "dereference"
instead, a ghost that knows it is an alias just passes all questions about itself
to the other thing?
but that would require a ton of proliferated code

maybe the TK just gets aliases and dereferences them after the fact to keep things clean

or there is a function on a ghost called g->absorbAttributesFromReferenceGhost()
and this moves over the hash, size, stat and other information

big question - is the file type of an alias always an alias?
like a symlink?
where you don't know if the symlink is a directory or a file?
maybe its best if we do that
unix seems to cope with that type of situation just fine


we need to figure out semantics for the reference database
we need to store two things permanently:

obj and ref

just like git

we can do this in our working directory
but it would be nice if this was all managed

this also should be some sort of FSE

objectstore

and there is a $coreOBS that is my working directory of shit
and i have an FSE complaint thing that manages it

so if I construct a hash like blob:423fe15,
G1 = tk->getGhostWithHash( "423fe15" )
    this routine is going to get you the most local version it can
    this may involve scanning various object stores it knows about
        its going to eventually end up saying
            $coreOBS->getGhostWithHash( "423fe15")
            the core OBS is special
            when you ask a object store FSE for a hashed thing
            it doesn't have to scan its known ghosts
            instead, it can actually just pick up the file
            it turns that into a ghost
            and someone should call "isAcceptaleCopy" to do a guard test on it

if i construct an lfn like blob:423fe15
G1  = TK->obtainGhostFromLFN( "blob:423fe15" );
    I'll get the ghost right away (its virtual)
    but the second anyone asks me to do something with it,
    i'll get delegated to to a blob FSE
    and the blob FSE knows to go back to truthkeeper
    maybe blobs are ocnsidered aliases (e.g. caller must dereference them)

thats sort of a nice way to handle it

in fact, yes, lets do it that way

maybe truthkeeper has a way when asked to do a getGhistWithHash
of passing that work down to FSEs?

there is also a ghostFromLFN strategy to think about
TK first checks its cache
if it can't find it there, it asks the relevant FSM
which implies that a global function to find FSM for LFN
this lives at the factory level


# early object store design


virtual ghosts
==============

gfs ls a b c 

will try to obtain equivilent of the following

gfs ls fs:a fs:b fs:c

but a b and c may not exist!

until a successful stat, they should be marked as a status = v
virtual should not get stored anywhere

whereas if you do a gfs ls . in a directory that has a b and c

in this case, a b c come from a scan, which by definition is not virtual
see?




so we create some virtual ghosts along the way
every time the operation is done, we should commit them (take off their virtual flag?)

maybe we just have a notion of copies or moves, and later we decide if they are local or remote?

need to check $g->isPathUnder($y);

next steps could occur in parallel
    begin copying over files we think are net new in reverse mtime order
    begin verification download (basically, get a snapshot of the remote FS)


PUSH looks different.. Push just says that we're going to move over things we think weren't there
into a "pushed-content" subdirectory
we could get things wrong but better for an offline situation

gsf accept pushed-content-1234 is used to rearrange the tree

maybe we have concept of a library
technically snapshots, orphans, pushes, etc could all be objects

GFS-library/objects
GFS-library/primay
GFS-library/orphans
GFS-library/pushes
GFS-library/snapshots
GFS-library/snapshots/Latest->xxx

need a flag on a blob if is is a moved file, or just some junk that GFS maintained
otherwise, cleanup on aisle three situation is shitty

maybe object syntax is as simple as foo/bar/this@234abc32a993, 
this syntax always means a file with this hash under that tree

how do we create definitive names for things and spread them around?
seems like git just stores things in namespace/xxx format

refs/tags/xyz



# design ideas for object stores
object store concepts

a blob always has the hash of the file it saves - the header (everything up to the first null) can be anything


# Revocation rules and timing


we need to not be in a situation where the fullhash has changed but
the stat stays the same
scenario - two ghosts pointing to same file
ghost A has Ha and Sa (hash and stat)
ghost B has Hb and Sa

if Hb is later that Ha, we'd be confused - how could both stats be identical but the hash has changed?
contract should be
getStat
getHash
getStat

if stat is the same both times, than take the timestamp and report it for both

also implies the "learn" semantic should always be accompanied by a date
and not generated inside the routines'

(merge case)

A new stat does not mean that a hash is out of date. The stat date should be used for resolving which of two stats is most accurate. 

stat fields should be taken all or nothing. 

The only cause for revoking a hash is when the size has changed or last modification date has changed. 

The revocation should not be immediate. It can be lazy. Maybe an optional warning. But the next time anyone asks for a hash on that file, we'd be forced to get one. 

Ghosts should never revoke other ghosts. Let that semantic rest with truth keeper. TK will just ask for incorporation in direction it wants. 

If you ask for a ghost that has no authoritative record is should be marked as virtual. 

One sanity check is that the virtual ghost should not be written to disk under normal cases. 




+++++++++++





+++++++++++++++++


seems like the best separation of concerns is a new idea:

FSE is really about the interface to a collection of files
TruthKeeper is the module that generates ghosts
it also tracks when ghost records are stale and may need to be pushed out to new places


+++++++++++++++++




Definitions:

    Source: A place to take things from, but must remain untouched
    Auxiliary|Pillage: A place we don't give a shit about and okay to move things

+++++




Thats sort of annoying, right? 

Everything is a ghost refactor. In this design, ghosts are interachanable with filenames
and FSE is really stripped down

M: main
FSE: Filesystem manager
    getStat
    getFullHash
    listDirectory
    copy
    move

G: Ghost
    getSize()
    learnSize() - accomodate potentially new information
    learnStat
TK: Truthkeeper
GS: Ghoststore


1-How do .gcpip files get populated initially?
2-How does .gcip file change when contents is changed?
3-How does a snapshot work?

    gfs snapshot filea fileb

    main: gs = TK->makeGhostsFromCommandline( @ARGV );
            obtainGhostForLFN( shift @ARGV )
                look inside our local run cache for these ghosts first
                if not, pull in the ghoststore of the parent using existing functions
        i = gs->getDescendingIterator(); - populate this GhostIterator with a queue of the roots
        g = i->next();
            pop queue 
            if( g->hasChildren() )
                g->getChildren()
                    getChildren finds out the last time dirent was populated
                        if ok, then TK->getCachedChildren( g );
                            load the .gcip file
                        if not ok, children = FSM->getImmediateChildren( g )
                                this function opens the directory, does a full scan,
                                and creates a ghoststore 
                                calls TK->makeGhostForLFN( "fs:" . thingWeFindInDir)
                            g->learnChildrenPresent( children );
                                setChildrenMarkPresent now scans the known children
                                marking those that aren't found as "not present"
                                and setting the lastDirEnt date
            onThunk
                g->finalizeChildrenPresent()
                    computes the hashes of names and subhashes
                    populates lastHashDate() 
                    g->learnFullHash(X);
                        if different, sets dirty flag
                        if date different, sets it, and sets dirty flag
                    g->flush();
                        Tk->flush() - called on thunk only for directory case
                        writes out the gcip file and cleans the dirty flag
        g->forceCharacterize();
            checks all of its fields. For those that don't seem right, 
            we call the FSM -> retrieveFullHashHard
                            -> retrieveStatHard




    gfs lsdup dirA dirB
    $sourceGS = TK->makeFromCommand( @argv sources)
    $targetGS = TK->makeFromCommand( @argv dest)

    $targetGS->getDescendingIterator();
    foreach( my $g = getNext() )
        $sourceGS->findAllDuplicates( $g );
            my @sizeMatches = $gs->findFilesWithSize( $size );
                @matches->isDuplicateOf( $g )



###


++++

RSYNC implementation ideas

gs = ghoststore of source
for each file in ghoststore
    get a full snapshot of the remote directory + its pillage/spare places + aux places
    get a full snapshot of the source directory
    identify corresponding path in target to match rsync exactly
        A. There is a file fT already sitting at TARGET at this filename
            A1. The hash of fT is correct - its what we want
                -- Mark TARGET fT as claimed, no action needed, go to next file
            A2. The hash is not correct
                -- Mark as orphaned, select new orphaning location - during clearance MOVE phase (will be moved out first)
                -- Flow down to Case B
        B. There is no file at the remote location in TARGET tree at that filename, or the file in TARGET is marked orphaned
            A1. There is no hash ANYWHERE at TARGET, PILLAGE, AUX, COPY or ORPHAN
                -- Queue for remote copy, function done
            A2. There is a hash available in PILLAGE that is not claimed
                -- Queue move PILLAGE -> fT, claim it
            A2. There is a hash availble in ORPHAN that is not claimed
                -- Queue clearance move ORPHAN -> fT (assign new location) - DONE  (What about case of a swap..?)
            A3. There is a hash available in TARGET that is claimed
                -- Queue local copy, done
            A4. There is a hash available in AUX
                -- Queue local copy, done
            A5. There is a hash available in PILLAGE that is claimed
                -- Queue local copy, done
            A6. There is a hash available in COPYLIST
                -- Queue local copy, done

First do all clearance moves
Then do all remote copies
Then do all local copies

Internal commands we need:

action_clearanceMove( $s, $d )
    mark $s as empty, $s->deleted()
    mark $d with new ghost by cloning and assigning new LFN

action_remoteCopy( $s, $d )
    mark $d 

action_localCopy


FileOpPlan
    knows how to record activities
    knows how to check for conflicts
    knows how to start work and verify its proceeding as planned
    fires actual actions over to the FSE in sequence
    copy
    move
    copyIntoObjectFormat
    
We need to insist on a contract that STATs are repeated during full hash
this is going to be done anyway by the OS
also a guard if the file changes and the original stat and hash fall out of sync





# too much caching

there is a good reason to really restat the entire destination directory
before developing a workplan otherwise operations will start to go wrong when
things aren't where expected. I think its fair to not rely on caching 
when you are actually about to write new stuff.. 

do this at the start, don't wait for some forced cache



now another thing has occured to me -what do we do about FSE proliferation?

technically, more than one FSE can be operative on a local file system at a time.

does it matter who or what finalizes these things?

we need to recheck the LSDUP semantics and make sure getFSE() isn't going to get 
completely confused. my guess is that because the ::FS is so thin, it probably doesn't 
matter.  it stores very little state relateive to the ::Snapshiot

but this is an issue when it comes to moving the ghost descriptor code. 

i think we need to make sure that all fs:root ghosts always get the global FS when they ask questions about themselves. And I think this is what the code does now.



# determinis and order

ghost iterators now sort their contents why? well, turns out there is some
non-deterministic behavior going on depending on how the values get scrabled
into the cache and i need consistency. ideally though, we need a way of
iterating through the ghost store in an order similiar to the one the user
intendend.



# Original notes from duplicateUnderPath

We have to decide at this point if the universal cache is up to date. 
Each FSE might handle this differently.

I think the check would recurively look at subdirectories? Or maybe just
simple time based? I'd like to find some way to prevent the need for the
user to be smart enough to know that a checkpoint is needed.

One way to handle this is there is some kind of
"hasAcceptableDirscan()". If dirscan is popualted for an AFN, we could avoid
the full traversal. But an acceptable dirscan on the .gfs_ip doesn't mean the
one inside the cache is up to date.

you'd want two fields (most recent enumer, and oldest enum.) 

maybe there is a recursive way. I can "test my own" understanding.
first, i get the GFS_IP for the current directory
its going to have some stats in it about sizes and file counts and whatnot.
and i compare that to my own db enumeration.

BDB may or may not be quick enough to allow this..?
but doesn't its Btree have some sort of "# records between?"
that would be optimal
that is the check?

the problem of a cache of a cahce of this kind it hat the "full deal" is what matters.
it seems to take about 1.4 user seconds to enumerate 1.36M rows in BDB
that tells me we've got a lot of room to grow here






