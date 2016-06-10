
# Universality and metadata

Less common ghost-level information i think has two different forms. The first is things that are part of the stat. This would include all of the stat variables, plus things that are right off the filesystem, such as the place a symbolic link is pointed. But the rule for this type of information is that it always is in lock step with the stat date. Lets call this the statMisc field. It should follow all places that the mtime, size and mode are carried. We can work on getting the data in their later but I'm worried that as code complexity rises, the number of places in the code that can "ignore" this stat information is rising too.

The second we can imagine is arbitrary n/v pairs that get passed along with a ghost. For instance, there might be a pointer to a hash that contains an ACL. Or a pointer to the result of a compression of a file. In these cases, a name is associated with a value and a assignment date. And the merger operation should be pretty simple.

I'd rather get this facility loaded into the system earlier so that it breaks fewer things than it might if adding it in later.

# What should an AFN mean?

As long as an AFN is more general and not wrong, I think it is OK to use it. We don't need to be perfect. Things can be renamed, etc. but as long as the AFN is storing something that has a fighting chance to be right and is easily repudiated as wrong, we should use it. Anything based on the local path or the contents of a file is wrong. But volume names feels safe to be an AFN. We can always change the semantics later.

# Volumes revisited

A volume has a name which I think is presumed to be universal in the setup of
the users stuff. However, its contents can change. So you look up its current
snapshot by looking at a pointer from the volumes name. I don't see any reason
for the AFN logic to be any more sophisticated (e.g. using volume permanent
names.)

There is some kind of action that consolidates a volumes content in case the
underlying more fundamental things have shifted. Each of a volume's entries is
backed by something more fundamental underneath, probably an fs entry. In the
earlier design notes, I thought maybe this was a "checkpoint" operation.

How could we make this more automatic? Maybe there is some kind of dirty flag
that can be processed. The challenge I've had with dirty flags is the concept
of dirtiness seems to have mutliple potential stakeholders.

Consider the case where there is a fundamental LFN

    fs:root:/Volumes/stick/beta

The file "beta" could part of a volume. That volume may or may not be even
loaded inside the interpreter at the time of the change. 

One way to handle this could be that every ghost has a state-change timestamp.
We can take it as canonical that any ghost that is updated can be easily
identified by this timestamp being advanced.

Then during cleanup of a run (or even at periodic intervals), we can always
compare an upstream entity (The volume object) with the fundamental entity
(the fs object) and do a "learn" operation.
