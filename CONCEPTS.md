
# About this file

This is a working set of design concepts. There is a retired, scratch version of this file also around. But everything in the code that has been cleaned up as of June 10, 2016 is here.


# Ghosts

The smallest unit of metadata is a ghost. A ghost is an imprint of a file or object. It says that name, size, location, checksums, permissions, etc of a file that either is on disk or once was on disk.

Every GFS operation (move, backup, etc) works by making ghosts of your files, and then applying formal logic rules to those ghosts. Making a ghost is extremely low-cost; basically just allocation of a small amount of memory. Data such as size, mode, file checksums, etc is added to the ghost on-demand.

Ghosts can represent files or objects on your local computer as well as remote one. They are entirely generalized to point to any kind of data asset that can have a checksum.


# Ghost contracts: What is truth? What is the single source of truth?

[Current descriptions as of 6/22/2016]

Ghost is a description of the potential state of a file.

Currently, contracts look like this

- No obligation for a specific ghost to be up to date
- Only the ghosts from the TK are guarnteed to be single source of truth
- Ghosts find their FSE by looking at their handler value
- Ghosts can be in any number of ghost stores
- TruthKeeper should make all ghosts that are intended to hold present state
- The only reason TK is not used to make ghosts is in cases where you are deliberatly constructing older things
- TK holds all ghost pointers and knows if they are dirty or not
- Generally, the FSE doesn't hold links back to ghosts except in cases where a ghost is a root (like a snapshot)

Its canonical that the LFN of a ghost once created must never change. If you need to reference a new thing, make a new ghost and copy over the data.


# TruthKeeper evolved: Everything is learned

[Current design]

You wnat multiple directories of ghost information -
  e.g. a backing store using BDB, .gfs_ip

THe truthkeeper ALWAYS makes the ghost first during obtain. Then state is gathered from the others each in turn via new learn operations.

When a ->save() is issued, the ghost is dropped out of the caches, and the ->saveGhost() message is given to each GDB

We need some mechanism to be sure the ghosts are actually finalized on save, otherwise operations that create millions of ghost are going to drive crazy amounts of memory use.




# What is a managed object exactly?

[Current as of 6/22/2016]

A managed object is a blob of information and metadata created by GFS, typically stored as a file on disk. They are typed and can be identified with a magic tag at the start of the file. The first use case of MOBs is a deep tree snapshot. But other uses are anticipated, for instance as “commits” (named backups).

MOBs may refer to zero or more sub-objects as hash pointers. This means that a GC operation can be performed to make sure that the MOB’s metadata is reconstructable, and also to be able to purge dangling references. For instance a list of known backups might be stored as a MOB, where each backup is another MOB, a pointer to the head directory of the backup.

MOBs are frequently carriers of ghost information, typically information locked down in place in time. Normally this ghost information is returned as immutable objects into the GFS layer. 

My design intent is that managed objects can live anywhere the user wants them to live. Also I think they are effectively immutable. They are identified by their hash. 



# effectiveDate

The contract with a ghost is that the ghost is always a picture of the best known state of the world. If new infromation comes in, the ghost is updated and saved. 

Information like stat() and hash() fields always come with the date of their most recent effective operation. This date is used to ensure that if conflicting information is available (say from different caches), we can always select the best data to use.

But other things could potentially change about a ghost, for instance attributes. These are things that are potentially not source verifiable. Each ghost maintains an effectiveDate, which is basically the date at which the information in the ghost overall is thought to be correct. For non-source verifiable information, the data with the latest effectiveDate is generally taken. 


### An ernest warning about strange but correct behavior
Some very innocent updates will not force a ghost to be dirty. For instance, a stat alone where no new information will not force a flush. This can cause some strange behavior. For instance, if you checksum a ghost that has changed, the new stat information will flush out. New stats will not be required for the period of the guard time. But then future checksum operations on an unchanged will force a new stat to occur every time, beucase the fact of the last stat will not have been persisted. It will suddenly look like the system lost its ability to cache stats.


## Ghosts for directories

Directories are also represnted as ghosts. A fullHash() on a ghost means get the full hash of its children, just like git(1). 

In an ideal backup scenario, I obtain a hash representing a particular directory. Primary source verification occurs by getting the hashes of all directories underneath. Therefore “has hash duplicate” still works. As long as the directory is EXACTLY the way it was, we are all good.

so now, calling getFullHash() on a directory object has a meaning. As a container object, it means getGhostSig() on each child (which forces stat and hash).. You want multiple calls to serializeImmediate to always return the same thing.

GIT would handle that differently. It would make a tree object, and the hash of the tree object would be the hash of the directory.

Now it is semantically meaningful to ask if a whole directory is available, exactly as-is at a remote location. You simply make sure that the getHash() on it is up to date.

So maybe you make a new ghost: cloneAsChildSignature. And you use newlines as the delimiters and escape what you’ve got.

As an optimization later, we can pack the structure so we don’t have a shit ton of ::::: 

This table shows which fields factor into the hash. 

    my @s = ( $lfn,                  YES, but must be relative
              $self->effectiveDate(),    NO
              $self->sequence(),         NO
              $self->uuid(),             NO
              $self->lastStat(),         NO
              $self->lastFullHash(),     NO
              $self->size(),             YES
              $self->mode(),             YES
              $self->mtime(),            YES
              $self->fullHash(),         YES
              $self->lastDirScan(),
              $self->lastEnumerated(),
              $self->firstStat(),
              $self->fileStatus(),
              $self->attributes(),
              $statStructSerialized
              );



# Dates and scanning

Each directory contains a lastEnumerated date. This date is the last time that the system performed a source verification on the LFNs directly underneath it. This field is consulted to decided if cached data about the contents of a directory is valid. If you call getChildrenLFNs() and the system finds a lastEnumerated inside the time window, no directory read is performed. 

Right now there is no direct provision for recalling the contents of the directory, short of a directory scan, but there is a clean design for how it should work. Basically, when getChildrenLFNs is called, we could either do a prefix search in the global LFN database, or read from a container list object (such as a gfs_ip files). The container list would register the children using addChild(). Then a call to verify the checksum would occur, and if the fullHashes matched, the lastEnumerated field would be accepted (if not, the hash and the lastEnumerated date would be dropped.)

In the current implementation, lastEnumerated is not passed along during learnFromGhost() operationes. It is only populated when learnChildrenByLFN is called with the contents from the upstream source. It also questionable if the fulHash should be learned for container objects during learnGhost either.

Consider two information sources A and B, with their picture of what is in a directory DIR.

    A:   DIR = {1, 2, 3 }
    B:   DIR = {   2, 3, 4 }

If A learns its children from B, there should be a check. Only one of these two versions of truth should be accepted. Whoever's picture of the kids has a more recent lastEnumerated date should be taken.

Directories also have hashes, but they operate in a funny way during learn operations. The full hash of a tree's age is the oldest of all of the contributing hashes, since it is effecitvely a compound hash. 

Directories also have a lastChanged field. This is used to mark events where we come to learn that the contents of a file have been modified. Of course, for regilar files, this is probably no different than the mtime(), but the mtime comes from the system and says in lockstep with the stat(2) operation, whereas lastChanged is based on changes we've either observed or caused. 

The lastChanged() field is updated whenever the hash or size of a fundamental object is observed. It is also updated when a directory is created, or when a file is written or moved. In short, anything that might feed into the signature of an object (hash, mtime, size) if it changes, will force a lastChange().

The semantics of lastChange() are recursive. Effort is made to propogate lastChanged to parent containers, namely directories. The parent of the ghost is located. If its lastChanged is more recent, than no action is taken. But if it is less recent, the ghost updates its field, and revokes the hash and deepSize field. This in turn forces a check on the parent, etc. This is primarily the reason the field is needed - it helps us know if we've already marked an object inconsistent so we can avoid doing it multiple times.

The net result is that a change deep inside the tree will propogate out to the ghost of its parent nodes, and prevent reliance on an outdated hash.

The lastChange() of directory is always the max() of its children.

This extra legwork is necessary because GFS does not want to overly rely on the user snapshoting or indexing the state of a filesystem to get correct information. Its data structures need to remain up to date as the file system changes, and some of those changes will happen inside the GFS tool during its operation.

Another field of note is getDeepSize() and getDeepCount(). This is basically the number of files and their total size under the tree. These fields too are invalidated when a change is mentioned, meaning that the next call to getDeepSize() will force recursion. 

To ensure that the expense of deep directory enumeration is not overly burdensome, there is a general purpose function called calculateDeepFields. This takes three arguments: forceFunctionHash, forceFunctionStat and forceFunctionEnum. If the forcing function is a time, those fields are brought to date to that granularity of time. If they are not defined, nothing is forced, but mergable information is located. Note that the user's guard times always override these values since the forcing is actually done by getChildrenLFNs(), getFullHash(), etc which will always tilt in favor of the guard values.

lastDeepScan() is the last time that the "deep fields" (deepSize, deepCount and hash) has been brought up to date. It is explicitly revokeed by learnContentsChanged(). lastDeepScan differs from lastFullHash because it can be updated without the hash being calculated. However, like its breatheren, calling it on a directory tree will always return the "minimum" time inside that tree. 

The verb to update the deepScan is "checkpoint", and has been implemented. 




# Painful lessons on state 

## Night in Woodbrige - disaster and response

Ran into a big glitch today. Ghost were getting generated for files that were not there. Those spurious ghosts were mucking up several things. First, the real_path function doesn't work if it can't check the file system. So it was returning undef silently when the paths were converted to absolute during the end of script flush. That foiled the AFN cache completely.

Secondly, even on fixing that, the code was trying to flush out .gfs_ip files to locations that didn't exist (to record the fact that files don't exist there.)

This illustrates the massive reliance on caching and cache consistency.  This illustrates a broad set of design decisions I need to address, that git and bup seem to have a nice way of dodging (becuase the index operation is always top down.)

1) All new state has to hit the disk as soon as possible. Given that my tool is trying to not have seprate indexing and staging phases, its all the more important. Cache consistency while the program is running has to be inviolate. Every find or obtain opperation should immediately flush all ghosts (and even raise an error).. We should not have routine cycles where dirty ghosts are kept in memory for more than a few operations. I'd even consider making "mark dirty" do an immediate flush but that is probably overkill.

*DONE: the way this is handled is now that flush is called in different places. And the ghosts stored in the system are now "pure", and never pulled from serialization (we use our now robust learning tools to handle it*.)

2) Mkdir's must revoke directory state

*On TODO list, but now fairly trivial given other implementations today*

3) Any learn operation that discovers a new hash or anything that changes the signature of a ghost must also revoke all known hashes.

*DONE: New logic takes care of this*
    
4) Knowledge that something isn't there (failed stat) needs to do way more that it does currently. It should revoke directories along the path. There is no sense in writing state for objects that don't exist either. The second we find something in a place we didn't expect it, or not see something in a place we did expect it, some mechanism needs to revoke things properly.

*DONE: If a stat has any meaningful signature change, it will invaldiate upstairs directories*.

5) GetFullHash on a directory has to respect both the hash guard time and the enumeration guard time. If I ask for a hash on a directory, and its enumeration time guard is expired, i must do a full descent. On a containerized thing like a tarfile, I'm willing to accept a different guard time on the expansion.

*Done. There is a deep scan guard now*. 

6) The GFS_IP functionality needs to gracefully degrade. If I can't write the .gfsip file, just issue a warning. Furthermore, its possibly dangerous having things in there. I can't just flush it out all the time. Say we learn about a deleted file. findAFN for that file, and a getStat fail. Lets say the stat was accidental, not forced by guardtime. the program is going to continue to get successful findAFNs over and over again while it is running, and not pick up on this change, so the accidental stat won't spread knowledge to invalidate our cache. How to solve that? OTOH, inside the system aren't we just caching what is supposed to be in those gfs_ip files anyhow? I think we'd be covered by that.

*DONE - its now fully disabled and new implementations can be cleaner becuase they will operate as "peers" to the AFN cache. The old ghost directory stuff will go away.*

## So what was the root cause and solution


Something is funky. Not picking up changes to a directory the way i'd expect. move large things in and out of a directory, and not even the most basic test to see if the checksum is still valid. (Not even a warning) I think whats happening is "hasAcceptableEnumeration()" may need to descent into the subtrees. 

Also a problem for STORE - the store operation is not checkpointing and revoking the parent hashes. 

Taint a directory taints parent, etc. But can we rely on that to avoid tainting all parents? No. What if the parent directory has never been incorporated? some criteria that says "hey, this directory ands its parents have already been revoked? 

So if I write into /a/b/c/d/filex, i should walk up to my parents to the root, making sure each gets a dirChange event. But as an optimization, i can also just see if a parent has a revoked flag and not have to revoke everything up the chain over again. 

*UPDATE: This is how it works now*

# File Versions

(How I think they should work.)

GFS generates a ghost object for every file or object it touches. In some cases, these files or objects have been seen before, and the GFS infrasture is fairly efficient at loading the old ghost record. As long as the file has the same name, GFS will see it as continuing the history of previous files of that name. 

When new ghosts are discovered, they are assigned a random bit of data called a UUID. The UUID means that even as that file might get moved from place to place, it can be linked back to the ghosts representing it’s previous places.

Ghosts are stores in several places. There is always a pointer to the “latest”, most up-to-date ghost for a given absolute location. This is the %afnToGhost cache. Every time GFS is invoked, it will attempt to store new state back into this cache. If that data has changed, GFS will also insert records into an ongoing multi-value hash called the %afnToFileHistoryGhost table. This table’s key is the AFN + effective date of the ghost.

To reconstruct the history and locations of a ghost, one simply traverses this afnToFileHistoryGhost() for the AFN of interest. One might also identify other AFNS with the same UUID and add them. Then sort the records and you have the output.

On flush, ghosts with an effective date on or after the time the script started are checkpointed out to journals. 
 



 
# Universality and metadata

## Stat meta-data
Status: Implemented

Less common ghost-level information i think has two different forms. The first is things that are part of the stat. This would include all of the stat variables, plus things that are right off the filesystem, such as the place a symbolic link is pointed. But the rule for this type of information is that it always is in lock step with the stat date. Lets call this the statMisc field. It should follow all places that the mtime, size and mode are carried. We can work on getting the data in their later but I'm worried that as code complexity rises, the number of places in the code that can "ignore" this stat information is rising too.

## Attributes
Status: Not yet implemented

The second we can imagine is arbitrary n/v pairs that get passed along with a ghost. For instance, there might be a pointer to a hash that contains an ACL. Or a pointer to the result of a compression of a file. In these cases, a name is associated with a value and a assignment date. And the merger operation should be pretty simple.

I'd rather get this facility loaded into the system earlier so that it breaks fewer things than it might if adding it in later.

# What should an AFN mean?

As long as an AFN is more general and not wrong, I think it is OK to use it. We don't need to be perfect. Things can be renamed, etc. but as long as the AFN is storing something that has a fighting chance to be right and is easily repudiated as wrong, we should use it. Anything based on the local path or the contents of a file is wrong. But volume names feels safe to be an AFN. We can always change the semantics later.

# Conceptual cleanup of Immutability, Overlays and Fundamentals

We are starting to see a design pattern where one mapping system passes down actual state to another object rather than being a fundamental thing. This suggests we need some clean ways of thinking about ghosts and ghost state.

* *Fundamental*: The ghost is a thing that can be source verified. This means that theoretically, some operation is available that can go back to disk and check that something is there. Or can go to web site and pull the headers.  
* *Immutable*: This is a flag on a ghost that indicates the ghost’s values cannot change. It may be either fundamental or an overlay. Either way, getSize() or getFullHash() will always return what already exist.
* *Overlay*: An overlay ghost inherits (learns) its properties from some other ghost. Calls to things like getSize() or getFullHash() will pass down to some other object. The FSE is called to decide what that other object should be. 

I think this design will support volumes and other virtualized things. For instance, in the end state, there is a VFS. The VFS is a registration of a path like 

	vfs:jeremy:/work/file1.txt	

Depending on the allocation and mappings, this object might be mapped into a object store, or an actual file on disk, or something remote. We might want to decide that a change to the underlying file also means a change to the VFS. The overlay mechanism will give us that opportunity.






# Volume Design

## Volumes revisited

A volume has a name which I think is presumed to be universal in the setup of
the users stuff. However, its contents can change. So you look up its current
snapshot by looking at a pointer from the volumes name. I don't see any reason
for the AFN logic to be any more sophisticated (e.g. using volume permanent
names.)

There is some kind of action that consolidates a volumes content in case the underlying more fundamental things have shifted. Each of a volume's entries is backed by something more fundamental underneath, probably an fs entry. In the earlier design notes, I thought maybe this was a "checkpoint" operation.

How could we make this more automatic? Maybe there is some kind of dirty flag that can be processed. The challenge I've had with dirty flags is the concept of dirtiness seems to have mutliple potential stakeholders.

Consider the case where there is a fundamental LFN

    fs:root:/Volumes/stick/beta

The file "beta" could part of a volume. That volume may or may not be even loaded inside the interpreter at the time of the change.

One way to handle this could be that every ghost has a state-change timestamp. We can take it as canonical that any ghost that is updated can be easily identified by this timestamp being advanced.

Then during cleanup of a run (or even at periodic intervals), we can always compare an upstream entity (The volume object) with the fundamental entity (the fs object) and do a "learn" operation.

## Original Volume Design

# Aliases and volumes

*Update*: I decided not to handle volumes with aliases. There seem to be better ways to manage volumes using this overlay concept, at least for the time being. 

We need a definitive way to store volume data.

    @volume/a/b/c

which really means

    fs:root:/VOLPATH/a/b/c

The rule is that isAlias() tells you that the thing is really an alias. And you can dereference it.

how? well, we have a LUT. Simplest thing is a file named volume in a particular directory that has the path.

when you do a global save to a ghost, it pulls open any aliases and updates
their ghosts as well.

that update should theoretically be done with a LEARN operation. So we need
learn() to be able to handle the alias type.

At certain times (when..?) we may want to snapshot the contents of an alias to something more permanent than the global working store. Sort of like a filesystem saying "make a registered snapshot of this thing". Like a checkpoint. The system can be pushed new data in a big bulk about what is that thing.

So i can get a download from a server saying "here is what i have right now", and this should flow into my understanding of what i can ask it for. Basically, the server can say, here is a list of LFNs i can respond to. Or a volume can say that. And I need to store that list so later I can decide what to do with them.

The trick with a volume is that it could move from place to place. So does the checkpoint behavior occur over the alias, or under it?

In concept, my server can say

    remote:gryphon:/foo/bar/bat
    remote:gryphon:/foo/bar/file2
    remote:gryphon:/foo/bar/file3

Meaning, any of these paths are valid. You're allowed to punch them back into the system and get your blob or object or whatever.

A volume similiarly could say

    vol:234df355562dd:/foo/bar/bar
    vol:234df355562dd:/foo/bar/file2
    vol:234df355562dd:/foo/bar/file3

Which means that as long as I can find '234df355562dd', my volume code can
find this thing.

When a volume is locally mounted, that will mean the following ghosts

    vol:234df355562dd:/foo/bar/bar
    vol:234df355562dd:/foo/bar/file2
    vol:234df355562dd:/foo/bar/file3
    fs:root:/Volume/mnt/DRIVE/foo/bar/bar
    fs:root:/Volume/mnt/DRIVE/foo/bar/file2
    fs:root:/Volume/mnt/DRIVE/foo/bar/file3

A relative path points into this object.

    vol:my-usb-stick:/foo/bar/bar

That is totally kosher -- when you as to translate it to an underlying ghost, it will hit the local repositories.

All all present. But I don't think I want fs:root:/Volume/mnt/DRIVE/foo/bar/bar to normally be displayed to a user.

That way, volume 234df355562dd can be found in different places.

Maybe a volume is best thought of as a snapshot FSE that maintains a link to real files.

So the 234df355562dd object needs to contain a pointer to the current mountpoint, if any. And it needs to have a snapshot location to use when things are offline.

The snapshot itself can be a blob. 

So a very basic dead simple approach here (w/o aliases for the time being?) is this syntax.

    gfs init-vol /Volumes/stick my-usb-stick
    > created 234df355562dd my-usb-stick

    .gfs/refs/volumes/my-usb-stick -> 234df355562dd
    write text file /Volume/stick/.gfs-id -> 234df355562dd

    gfs checkpoint /Volumes/stick
    <reads all of those files, writes them to a snapshot of some kind>
    > created checkpoint 33fc12 
    .gfs/objects/33/33fc12 - holds this snapshot thing
    .gfs/refs/latest-checkpoint/volume/stick -> 33fc12

Basically, volumes are an overlay somehow over ghosts in that part of the tree. That means that a volume is empty until it is checkpointed. and maybe it also means that the volume state is invarient, so changes to the underlaying ghosts don't change the volume necessarily.

So if we work inside of /Volumes/stick/testfile.txt, and make some file changes with gfs, the contents of the volume have not technically changed yet. I don't automatically get vol:234df355562dd:/testfile.txt.

I think adding the automated mapping case is easy once it works manually.

The system is going to need to know that volumes have to be dereferenced. the
dereference call gets passed down to the FSE which returns you a more basic
filesystem ghosts. Or maybe its better thought of as a "retreival", rather
than a dereference. If it were a remote file, we'd want to pull it down when
the dereference occurs.

SO dereferencing/retreivals are not normally done. The only time you really do them is when you copy something or have to make the file.

    gfs copy vol:23432:test.txt bar
    # this will dereference vol:23432:test.txt -> to a ghost on disk
    that is then handled like any other file. Dereferencing could be quite 
    expensive in the case of a file retrieval.

That implies that a bucket manager like S3 needs to maintain a map of what is stored at its location and where a copy exists internally when its pulled down.

This "dereferencing" needs to be maximially lazy - you'd never do it unless you had to.

An ls operation wouldn't actually derference.

    gfs ls vol:2343:tst.txt
    # shows what we know about that entry in the latest checkpoint

asking one of these things to verify itself is pointless. They are immutable. 

Verification happens explicity when you force these things to be dereferenced and then you can learn the checksum or data from the dereferenced thing. Or you have the server retrieve that file or perform some task, and then send you new ghosts.

So learn() needs to work on immutable things but it should never occur from a getSize() or getMode(). hasAcceptable() must always be true for immutable things.

To do all of this, i need a super fundamental tiny layer above the fs:root. I suspect this thing will be like an FSE.

    my $OJS = FSE::Objectstore( ~/.gfs/obj );
    my $file = $obj->getTempGhost()
    write into that file
    my $g = $OJS->store( $file );
    # g = fs:root:~/.gfs/obj/22/22fc22
    $OJS->findHash( 22fc22 );
    # gives me back the ghost that points to that thing
    $OJS->setPointer( 'volumes/my-usb-stick','234df355562dd');

Does the object store have its own LFNs? I have no idea. I think its just a naming service. But lets plauy around with the idea that its smarter than that and exposes some type of LFN.

    gfs ls obj:CORE:^234df355562dd
    gfs ls obj:CORE:my-usb-stick

A "path" in this name space is a named object. And it always resolves to a ghost sitting in the object store.

Basically, its a thing that can always generate FS ghostnames for you. It can be super limited. Doenstn' have to support fancy aliases or checkpoints. Just translate a tag -> data.

Its any fancy. It basically something almost as primitive as the fs: guys. No caching needed No AFNS needed. I'm not even sure it strictly needs a ghost- like interface.

So now, to get our volume behavior, there is a more advanced FSE::Volume. Say this tool is given something like vol:lacie-drive:foo.txt and asks to derefernece it. The API would call

    $g = $OJS->getPointer( 'lacie-drive' );
    now, slurp the ghost $g into a ghost store and save it
    hit my ghost store for foo.txt 
    and now i have my primary record of that thing.

Lets make the objectstores technically be rooted in different places. But there will be a special one called CORE. SetPointer and GetPointer are super flexible and cache their answers. Also the value they point to can be any string, not just a hash. But as a hack, setPointerHash will check that Hash is actually stored by OJS.

OJS doesn't have to hang onto any ghosts but it will definitely make rich use of the global ghost cache so that its super speedy.

Interesting implication of this design: it means that LFNS like "obj:CORE:my- usb-stick" are technically pointers. Say we make a ghost and call getSize(). What is the getSize() of such a thing? Shpuld it automatically follow down to the FS? I'm not sure it really matters. I'm fine with that bombing out the system. Or maybe delegating back to the ghost of the file it points to is just fine - they will always be filesystem objects, special cases don't need to be considered since this is really just a light layer over the filesystem. Only trick is in the case where the Pointer goes to something that is not a hash.

OJS can have a rule - pointers may not end in a slash!



# Existence

STATUS: Implemented as described here. 

A stat on a file either has to tell you that the file has certain information, or that it DOESN't have this information (e.g. the file doesn't exists). A third option is that we cannot know (e.g permission issue.) How should this data be passed back up the chain?

Simplest way I can think of is a modification to the stat rules. One of the returned fields is basically a status, which can be considered a code of these three situations. THe other parameters should be undef'ed. But that raises a contradition - we'd have undefined data and a valid stat date.

Simplest Solution: Size is undef, mtime is undef, mode is "DELETED" or "NOACCESS"

## user verification of existence

I think if the user bothers to list something, we can take it as a given  that its OK to actually source verify it.  To not do that could cause a lot of problems -- for isntance, if there were a global cache, user could ask to LS things that were long gone and never be the wiser not expected behavior... so we can't do that.

## Checking for a real file2

the question is how should the caller be aware of the deleted file in this subcase? i've got a ghost, and maybe i think its a deleted thing. i call exists() to make sure it really is deleted and instead i get a true. should my ghost now change? probably overthinking this case.

if you have a ghost and its marked as deleted. you soruce verify it, and its there. refresh its stat, and mark it found. if the hash changes, so be it. maybe the immutable flag plays a role here. if immutable, we can't source verify it.

Resolution: basically, exists() can source verify, and it source verifies by doing a stat. That stat will bring in new knowlege, and may change flags on things to bring the ghost to a point where it is consistent with reality.

So what if we have a ghost and our purpose is to test if it conforms to reality? well, the answer would be to create a new ghost with that LFN and verify that one. not sure why you'd want this, but it probably will come up. suggests a semantic where you can ask a ghost to "checkDiscrepencies" and you get a new ghost back with the actual state.

# RSYNC Emulation

STATUS: Not yet implemented this way, but heading in that direction.

## What does RSync Do?

Rsync has some quirks I'd like to recreate. First, it doesn't matter if target is there or not, it gets created. Secondly, the trailing slash seems to be irrelevant to the target.

    rsync -av /bar/foo quxx target
        target/foo
        target/quxx

    rsync -av /bar/foo quxx target/
        target/foo
        target/quxx

The -R switch means that the entire provided path is taken as the root for the transfer

    rsync -av -R /bar/foo quxx target
        target/bar/foo
        target/quxx

    rsync -r dirA dirB destination
        destination/dirA/..
        destination/dirB/..

A trailing slash basically means treat the passed argument as a collection of files

    rsync -r dir-a/ dir-b destination
        destination/filea1
        destination/filea2
        destination/filea3
        destination/dirB/fileb1 
        destination/dirB/fileb2 
        destination/dirB/fileb3

Using this -R switch turns off the trailing slash behavior too

    rsync -r -R dir-a/ dir-b destination
        destination/dirA/filea1
        destination/dirA/filea2
        destination/dirA/filea3
        destination/dirB/fileb1 
        destination/dirB/fileb2 
        destination/dirB/fileb3

This creates some counter-intuitive behavior when the user is thinking about true mirrors

    rsync -r -R dir-a dir-b destination/dir-a
        destination/dir-a/dir-a/filea1
        destination/dir-a/dir-a/filea2
        destination/dir-a/dir-a/filea3

Because presumably you wanted it to make dir-a "look the same".

So in short, a trailing slash on TARGET is ignored
And a trailing slash on a SOURCE is only relevant if -R is off

## Translation to the gfs(1) tool

Lets say that foodir is a directory.

    gfs ls foodir 
    -> one result, representing the entry of foodir itself

    gfs ls foodir/
    -> a result for every entry under foodir

My guess is that its better to emulate the rsync(1) usage than the ls(1) usage
since most of the time you're handling directories of things.

## Playing devil's advocate on this decision

Devil's advocate.. Lets say you do a sync operation

    gfs sync foodir bardir

What is more logical? Would you want to think that foodir and bardir should look the same afterwards? Or would you think that foodir should be synced underneath bardir? After all, both are entries, and a sync operation suggests that one should look like the other. More cognitive load to remember to place things under other things.

    gfs sync foodir bardir/

To my eyes, the above seems to clearly say "stick foodir under bardir".

So the question is do we create our own new convention?

What about 

    gfs store foodir bardir

This seems to clearly call out for foodir to be stored as a full entry under something else.

So in this alternative scenario, a trailing slash always means "take what is underneath here".

    gfs sync foodir bardir
    # make them the same
    # if bardir had stuff not in foodir, remove it

    gfs sync foodir/ bardir
    # take everything under foodir and make sure each is the same as bardir
    # probably should result in an error - what does it mean to sync 3 things to 1 thing?
    # or maybe we pretent bardir had a trailing slash
    # or maybe this says that bardir should have exactly the contents of foodir in it

    gfs sync foodir bardir/
    # take foodir as an entry and make sure its the same as bardir/foodir
    # don't remove anything from bardir

    gfs sync foodir/ bardir/
    # make sure the contents of foodir appear under bardir.

Decision: NOT MADE - appears we may want flexibility to emulate multiple
formats in the future




# Learn semantics: Time to get these right

Status: "Canon" - the program is forced to conformed to this

There are two different kinds of stat overrides. Some stat overrides imply
immediately that the file is different, or are such a warning sign that they
cannot be ignored.

    - The size has changed (hash by definition cannot be the same)

Other stat overrides merely suggest suspicion.

    - Modification date changes
    - Inode has changed
    - Creation date has changed

## Logical Truth Statements

Entity(truth name / time name)

statA/B - A=size B=everything else
hash - hash

A(statA/A/1) <- A(statB/B/1) = ERROR

    Error. Times are identical. 

A(statA/A/2) <- A(statB/B/1) = A(statA/A/2)

    Newer stat in learner remains over older one from teacher

A(statA/A/2) <- A(statB/B/1) = A(statA/A/2)

    Newer stat in learner remains over older one from teacher

A(statA/A/2) <- A(statB/B/1) = A(statA/A/2)

    Newer stat in learner remains over older one from teacher

A(statA/A/1) <- A(statB/B/2) = A(statB/B/2)

    Newer stat from teacher enters learner 

Simple cases, old replacing new (as a compound step)

    A(statA/A/1,hashA/1) <- A(statB/B/2,hashB/2) = A(statB/B/2,hashB/2)

Simple case, older data is is ignored

    A(statA/A/3,hashA/3) <- A(statB/B/2,hashB/2) = A(statA/A/3,hashA/3)
  
Stat B is different, hashB is different 

    A(statA/A/1,hashA/1) <- A(statB/B/2,hashB/1) = ERROR - two conflicting hashes w/ same timestamp

    A(statA/1,hashA/2) <- A(statB/B/2,hashB/1) = IMPOSSIBLE - 
        A would never have a stat older than the snapapshot

Harder case, B is more recently stated, but has older hash. Size same

    A(statA/X/2,hashA/2) <- A(statB/X/3,hashB/1) = A(statB/X/3,hashA/2)
    # take stat B as a refresh, allow the older hash to live on

    A(statA/X/2,hashA/2) <- A(statB/Y/3,hashB/1) = A(statB/Y/3,<revoked>)

Now break down into smaller logical steps

    A(statA/X/2,hashA/2) <- A(statB/X/3) = A(statB/X/3,hashA/2)
    # allow the B to come in

    A(statA/X/2,hashA/2) <- A(statB/Y/3) = A(statB/Y/3,<revoked>)
    # Hash "revocation" case

    A(statA/X/2,hashA/2) <- A(hashB/3) = A(statB/Y/3,hashB/3)
    # new has enters as long as its same or newer than the stat on A

Now try it without an explicit recovcation - can it work?

    A(statA/X/2,hashA/2) <- A(statB/Y/4) = A(statB/Y/4,hashA/2)**
    # Hash "revocation" case, A is now dirty

    A(statB/Y/4,hashA/2)** <- A(hashB/3) = A(statB/Y/4,hashA/2)**

    # Entry of a hash older that stat, but newer than hash. Allow it. 

The revocation plays a role here -- it conveys information.  In the case that the hash and the stat date are identical, we know the ghost is maximally correct. If the stat date is newer, we cannot distinguish between the case that innocent data came along and really corrupting data came aloing.

Theoretically, there is nothing "wrong" with keeping the old hash. Technically, the information is out of date and caller could decide. But we've lost the change to tell the caller just how badly out of date that information is.

I think the optimal policy then is to signal this case with recovcation.. If I ask a ghost for a hash, its the best knowledge of the ghost's current state. If I learn that my data is wrong, i'm failing in the contract even if I return the "best known."

Basiclaly, the best known hash of a ghost who has received a size change is no hash at all.

# The rules in summary now that we have them

RULE #1: A stat can never be older than the hash in a given ghost. Cause for immediate program termination. (However, It IS allowed in cases where the ghost represents a directory, as this is expected behaviors.)

    What is the benefit of this RULE?
    WHY?
    It doesn't hold up for directories, where the hash has nothing to do with the directory itself. 
    But why enforce it for regular files? Who benefits?
    Say I stat a file, and then get a hash later. I don't know if that new hash is consistent with the stat or not. That's why the software is careful
    to set these two values to the same when the hash is calculated.

    RULE #1-B Except for directories

RULE #2: Two peices of information with the same date must be the same, otherwise ERROR

RULE #3: The date corresponding to the size is lockstep with the date on rest of the stat

RULE #4: A new provided hash will be accepted into a learner as long as its date is newer than both the hash and the stat

RULE #5: A new stat that represents a change in size should invalidate the existing hash, unless the hash has the exact same date as the stat. (If the existing hash is newer than the incoming stat, but older than the existing stat, we have a logical violation of RULE #1)

RULE #6: A dead stat (ERR) always revokes the hash and blanks out size and XXX





# Snapshot Design Decisions

STATUS: All of this is current, or at least consistent with implementation

# AFNs and snapshots

technically the B file's only LFN inside the invocation is: 

    snap:testdir/snapshot.gfssnap/b

what is the AFN? One (Wrong) idea:

    /home/jgilbert/testdir/snapshot.gfssnap/b

but what if snapshot.gifssnap moves?

The funny thing in the case of a snapshot is that snapshot.gfssnap is 
probably not going to change.

So you'd be good actually saying the AFN is: 

    snap:423fe15a4757767bb4d2a04702f8572d3421179d/a/b


If the snapshot changes, you'd automatically get "revocation" of those AFNs



# Interesting hanging chad around ghosts

say you want to write a snapshot to a new place. you get a ghost for
[snap:snapshot-5419.out:] and get the FSE to write to it. You store some
objects in it, and finalize the FSE. But there is a significant gap - what is
the purpose of [snap:snapshot-5419.out:]? Does it need to be finalized?

it is technically a valid path. its a valid thing in the world. its the head
position of a snapshot. its even been read and written to a few times. the
ghost represents a real thing, and you could potentially even want to do an LS
on it later. So who is supposed to write it to disk?

I decided to handle this by a finalize() pass on the FSE. But its not clear how this interacts with the dirty flag so the hanging chad remains.

# root renames on snapshot loads

a strange corner case - you write out a snapshot, a bunch of ghosts.  what is
the proper root of the snapshot file? Its actually not clear what that should
be. for instance:

    gfs snapshot foo bar -> snapshot123.snap

    gfs ls snap:snapshot123.snap:foo snap:snapshot123.snap:bar

    cp snapshot123.snap jeremy.snap

    gfs ls snap:jeremy.snap:foo

Technically, the root here was just a transitory thing. Its basically an local
relative filename. 

Design decision: The written version will always just be "root". but on LOADs, we'll rewrite the LFNs/root so they make sense to the user.

# Dealing with the obnoxiousness of absolute paths and snapshots

We decided earlier that the permanent root of a snapshot is the hash of the file that stores it. This seems to work well for git(1). But what happens with the framework wants an absolute LFN before its written? Current design decision is this is not allowed.

this is a pretty wide ranging issue, turns out. Say I invoke the following

    gfs ls snap:snapshot-5803.out:quxx

Technically, "snap:snapshot-5803.out:quxx" will trigger creation of a FSE all of these ghosts are sort of not real.

Ghosts that are not real should never be attempted to convert to AFNs? Is that true? they certainly shouldn't find their way into a persistent store. If that were true, the truthkeeper has to learns that it is actualized, and thats when it does and AFN test.

Decision: Snapshot ghosts are by definition immutable(). But they can have AFNs.




# About LFNS

STATUS: All of this is current

LFN - local file name - whatever is colloquial to the USER to
describe something. LFNs can never presume to be universally unique between
invocations of GFS because they may be relative paths or tags/references may
change. But the user's maximal intent must be captured in the LFN.

    for instance fs:a depends on your CWD
    for a different PWD, fs:a could refer to a different file

However, there are many different handlers.

    /foo
    fs:/foo
    snap:foo.snap:file123
    remote:jgilbert@auspice.net:file123

In this case, the "LFN" is a composite of a handler, a root and a path. The
user may provide some or all of these peices on the command line. We need to
be be able to intelligently construct the rest.

LFN's can be fullqualified, which means fully specify the missing strings and
to canonicalized the path.

To determine if two LFNs are the same inside one running instance of GFS, the
only way to do it is to canonicalize that path portion both to remove . and
../ etc. Then you need you make sure to attach the ROOT and the HANDLER.

Across instances, the only way is to convert to an AFN (absolute file name)
The AFN is a unique permanent name for the location

FQAFN - fully qualified AFN: handler:root:afn
FQLFN - fully qualified LFN: handler:root:path (can be local)

Snapshots will generally represent LFNs, typically from the point of view of the PWD but their power is that they should NOT be foreced to contain AFNs - the snapshot could be reconstructed anywhere and we'll leave it to the user to either feed a snapshot the AFNs or the LFNS.

In the nested case, assume the following data is stored

    snap:testdir/snapshot.gfssnap:
    snap:testdir/snapshot.gfssnap:a
    snap:testdir/snapshot.gfssnap:b

in this case, the LFN of the snapshot file itself is testdir/snapshot.gfssnap

So LFNs are designed to hold three peices of information, and together forms a unique namespace.

# What is truth? What is the single source of truth?

STATUS: All of this is current

Ghost is a description of the potential state of a file.

Currently, contracts look like this

- No obligation for a specific ghost to be up to date
- Only the ghosts from the TK are guarnteed to be single source of truth
- Ghosts find their FSE by looking at their handler value
- Ghosts can be in any number of ghost stores
- TruthKeeper should make all ghosts that are intended to hold present state
- The only reason TK is not used to make ghosts is in cases where you are deliverably
- TK holds all ghost pointers and knows if they are dirty or not
- Generally, the FSE doesn't hold links back to ghosts except in cases where a ghost is a root (like a snapshot)


# Command Invocation Ideas

all of this is in the idea phase


    reconstruct-test A B C
      makes sure that everything in A and B be reconstructed in target C
      as long as there is at least one valid duplicate, we move on
              first, best approach is to look for hashed AFN
                     ask FSE if it "owns" that file
                     then call "hasIdenticalContent()"
                  if that doens't work, check the cache of sizes
                      call hasIdenticalContent
                     as soon as one returns "hasIdenticalContent", we return
                  finally, force the full cache of sizes

    copy-delta A B C
       Anything in A and B that is not in C is going to get copied over 
       into a "delta" directory

    rsync A B C
       C is reconfigured to look exactly like A and B. Files that no longer
       belong can be identified and removed as an option.
       this works by MOVING files around inside of C. depending on how
       slashes are used, we either merge or replace (rsync ffo/ behavior)
       $fse->move( a, b );
              the move tool has to move the ghosts too
              and that may invalidate certain descriptor files


    gfs snapshot .
    generates a snapshot file
    gfs condense /etc /var snap:barf.gs
    gfs condense snap:barf.gs snap:arf.gs snap:new.gs
    will reconstruct using known afns, keeps orphans
    gfs reconstruct --test snap:barf.gs foo/
    150 files reconstructed, 4 not located (see gfsrun.3234.orphaned.txt for details)

     make symlinks rather than copy files
     gfs reconstruct --symlink 
     uses xxx and yyy as hash sources
     gfs reconstruct --test snap:barf.gs foo/ --source=xxx --source=yyyy
     verify all descriptors in /etc
     gfs verify /etc
     
     creates a GFS entity that can be pushed to
     gfs gfs create /etc gfs:foo.gfs
     gfs gfs push /etc gfs:foo.gfs
     makes sure that /etc can be reconstructed by gfs:foo.gfs

     gfs gfs add gfs:goo.
     a .gfs file can contain storage and 




