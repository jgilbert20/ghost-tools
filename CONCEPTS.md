
# Universality and metadata

Less common ghost-level information i think has two different forms. The first is things that are part of the stat. This would include all of the stat variables, plus things that are right off the filesystem, such as the place a symbolic link is pointed. But the rule for this type of information is that it always is in lock step with the stat date. Lets call this the statMisc field. It should follow all places that the mtime, size and mode are carried. We can work on getting the data in their later but I'm worried that as code complexity rises, the number of places in the code that can "ignore" this stat information is rising too.

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

SO dereferencing/retreivals are not normally done. The only time you really do
them is when you copy something or have to make the file.

    gfs copy vol:23432:test.txt bar
    # this will dereference vol:23432:test.txt -> to a ghost on disk
    that is then handled like any other file. Dereferencing could be quite 
    expensive in the case of a file retrieval.

That implies that a bucket manager like S3 needs to maintain a map of what is
stored at its location and where a copy exists internally when its pulled down.

This "dereferencing" needs to be maximially lazy - you'd never do it unless you had to.

An ls operation wouldn't actually derference.

    gfs ls vol:2343:tst.txt
    # shows what we know about that entry in the latest checkpoint

asking one of these things to verify itself is pointless. They are immutable. 

Verification happens explicity when you force these things to be dereferenced
and then you can learn the checksum or data from the dereferenced thing. Or you
have the server retrieve that file or perform some task, and then send you new
ghosts.

So learn() needs to work on immutable things but it should never occur from a
getSize() or getMode(). hasAcceptable() must always be true for immutable
things.

To do all of this, i need a super fundamental tiny layer above the fs:root. I suspect this thing will be like an FSE.

    my $OJS = FSE::Objectstore( ~/.gfs/obj );
    my $file = $obj->getTempGhost()
    write into that file
    my $g = $OJS->store( $file );
    # g = fs:root:~/.gfs/obj/22/22fc22
    $OJS->findHash( 22fc22 );
    # gives me back the ghost that points to that thing
    $OJS->setPointer( 'volumes/my-usb-stick','234df355562dd');

Does the object store have its own LFNs? I have no idea. I think its just a
naming service. But lets plauy around with the idea that its smarter than
that and exposes some type of LFN.

    gfs ls obj:CORE:^234df355562dd
    gfs ls obj:CORE:my-usb-stick

A "path" in this name space is a named object. And it always resolves to a
ghost sitting in the object store.

Basically, its a thing that can always generate FS ghostnames for you. It can be super limited. Doenstn' have to support fancy aliases or checkpoints. Just translate a tag -> data.

Its any fancy. It basically something almost as primitive as the fs: guys. No
caching needed No AFNS needed. I'm not even sure it strictly needs a ghost-
like interface.

So now, to get our volume behavior, there is a more advanced FSE::Volume. Say
this tool is given something like vol:lacie-drive:foo.txt and asks to
derefernece it. The API would call

    $g = $OJS->getPointer( 'lacie-drive' );
    now, slurp the ghost $g into a ghost store and save it
    hit my ghost store for foo.txt 
    and now i have my primary record of that thing.

Lets make the objectstores technically be rooted in different places. But
there will be a special one called CORE. SetPointer and GetPointer are super
flexible and cache their answers. Also the value they point to can be any
string, not just a hash. But as a hack, setPointerHash will check that Hash is
actually stored by OJS.

OJS doesn't have to hang onto any ghosts but it will definitely make rich use
of the global ghost cache so that its super speedy.

Interesting implication of this design: it means that LFNS like "obj:CORE:my-
usb-stick" are technically pointers. Say we make a ghost and call getSize().
What is the getSize() of such a thing? Shpuld it automatically follow down to
the FS? I'm not sure it really matters. I'm fine with that bombing out the
system. Or maybe delegating back to the ghost of the file it points to is just
fine - they will always be filesystem objects, special cases don't need to be
considered since this is really just a light layer over the filesystem. Only
trick is in the case where the Pointer goes to something that is not a hash.

OJS can have a rule - pointers may not end in a slash!


