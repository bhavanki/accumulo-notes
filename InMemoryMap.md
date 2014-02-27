# A Tour of InMemoryMap

This document describes the `InMemoryMap` class used primarly by the
Accumulo `Tablet` class. An `InMemoryMap` holds sorted key/value pairs, and
includes support for dumping the data to disk and for providing consistent
iteration as new data is written.

## MemKey and MemValue

Before understanding `InMemoryMap`, it's important to understand `MemKey`
and `MemValue` classes, which extend the core Accumulo `Key` and `Value`
classes, respectively. Both classes add a key/value count field to the state
of their superclasses (which defaults to `Integer.MAX_VALUE`).

A `MemKey` can be compared to an ordinary key. If two keys are equal as ordinary
keys and both `MemKey` instances, then they are compared by their key/value
counts.

`MemValue` instances do not compare in any special way.

A `MemValue` object additionally remembers if it has been merged, and new
instances begin as unmerged. An instance becomes merged when it is written to
a data stream; at that point, the four-byte key/value count is tacked on to
the end of the underlying byte value. The data can then be transported and read
as an ordinary Accumulo value. The method `splitKVCount` can extract the
key/value count from such a value and restore its original data.

### The Key/Value Count

The purpose of the key/value count is to act as a signal for how many mutations
(column updates) have been performed so far, or a sort of logical timestamp.
As described below, `InMemoryMap` keeps track of the current key/value count,
and it is used in several ways to provide consistent behavior.

### The Point of Merging

`InMemoryMap` is capable of dumping key/value pairs to an RFile. While in the
map, each key is actually a `MemKey` with an associated key/value count.
However, the RFile format cannot support preserving that count with the key.
So, when each key/value pair is written out, the value is converted to a
`MemValue`, which can preserve the count through merging. The key/value count
on the key is lost upon writing.

When the dumped key/value pairs are read back in later, the
`MemKeyConversionIterator` described below reverses the process, splitting the
key/value counts from the values and restoring it to `MemKey` instances.

## Associated Classes

The source for `InMemoryMap` includes a set of associated classes.

### MemKeyComparator

A `MemKeyComparator` compares Accumulo keys. However, if the keys are equal
and either of the keys are actually `MemKey` objects, it can compare them by
their associated key/value counts. A `MemKey` instance always sorts after an
equivalent ordinary key.

Two keys with the same key value count are therefore ordered from the earliest
to the latest.

### PartialMutationSkippingIterator

A skipping iterator is an iterator over sorted key/value pairs that wraps
another iterator and can skip entries in that wrapped iterator. It is assumed
that the keys in the iterated key/value pairs are actually `MemKey` objects.
The iterator skips over any pairs where the key's key/value count exceeds a
maximum that is set when constructing the iterator.

The skipping behavior provides a consistent view of key/value pairs that are
iterated over. Each pair is denoted with the key/value count describing the
logical time when it was written. This iterator can therefore be made to
skip pairs that were created after a certain point in time, i.e., that are
in "the future" from the perspective of the iterator's creation.

### MemKeyConversionIterator

This iterator is another wrapping, sorted key/value pair iterator. The
iterator does two special things to iterated pairs.

1. If the key is a `MemKey` (or null), it passes the pair through; otherwise,
   it assumes that the value is merged `MemValue` data containing a key/value
   count, and it splits the count out to replace the key with a corresponding
   `MemKey` with that count. (The value data is restored to its unmerged state.)
2. When a seek is performed to a range starting with a `MemKey`, the seek is
   first performed normally, and then the iterator moves ahead to the first
   key/value pair whose key is at least equal to the range's start key.

The first item is how dumped key/value pair data is properly restored with
key/value counts upon iteration.

## SimpleMap

`SimpleMap` is an inner interface of `InMemoryMap`. It defines a simple
key/value map contract:

* get a value given a key
* get the map size
* get the memory usage of the map
* delete (free) the map
* get an ordinary iterator over key/value entries, starting from a key
* get a sorted key/value iterator
* perform a list of mutations, associated with a key/value count

Note that item removal is not defined.

Three implementations of `SimpleMap` are defined.

### DefaultMap

`DefaultMap` uses a `ConcurrentSkipListMap` for storing key/value pairs. This
is a sorted map, and keys are compared using a `MemKeyComparator` instance.

Getting values, the map size, and iterators is straightforward, as is deletion.
As pairs are added to the map, this implementation keeps track of the memory
used by the key and value bytes, plus the key/value counts to be applied; it
uses this information, plus an estimated overhead per pair, to provide the map's
overall memory usage.

Each column update in a mutation is processed when a list of mutations is to
be performed. While the value in a column update is left alone, the key is
converted to an unmerged `MemKey` (its original bytes are used). The key/value
count for the first new `MemKey` is the key/value count passed to the method
for performing the mutations, and the count is incremented for each subsequent
pair.

### NativeMapWrapper

A `NativeMap` is a C++ implementation of a key/value map. `NativeMapWrapper`
wraps a `NativeMap` instance, and delegates all calls to that instance.
`NativeMap` performs similar key/value count incrementing for mutations, except
that each column update in a mutation is processed with the same key/value
count.

### LocalityGroupMap

A `LocalityGroupMap` aggregates an array of `SimpleMap` instances, one for
each locality group defined for it, plus one extra for the default, unnamed
locality group. The nested maps can be either `DefaultMap` or `NativeMapWrapper`
instances.

The map size and memory usage is the sum of its nested maps' sizes and usages.
Deleting the map simply deletes the nested maps.

This map cannot be used to get a value or an ordinary iterator. A sorted
key/value iterator can be retrieved; the result is a core
`LocalityGroupIterator` through each of the nested groups' sorted key/value
iterators.

A list of mutations is processed by partitioning them (using a
`LocalityGroupUtil.Partitioner`) across the locality groups. The mutations
for each locality group are applied to the corresponding nested map. The
key/value count is incremented by the number of updates in each mutation
applied.

## Finally, InMemoryMap

At its most basic, an `InMemoryMap` wraps one `SimpleMap`. When an `InMemoryMap`
is constructed, it is told whether to use native maps and given an optional
set of locality groups; these parameters drive whether that wrapped map is
a `LocalityGroupMap` or not, and whether `NativeMapWrapper` or `DefaultMap`
instances are used.

Some methods perform a straightforward delegation to the wrapped map.

* `getNumEntries()`: `size()`
* `iterator(Key)`: `iterator(Key)`
* `estimatedSizeInBytes`() : `getMemoryUsed()`

The rest are more involved.

### Mutations

An `InMemoryMap` can process a list of mutations. The list does end up being
sent to the wrapped map, but the key/value count must be determined first.

Initially, the `InMemoryMap` sets its own current count to zero and its own next
count to 1. The next count is what is sent to the wrapped map. After the
mutations are processed, the current count is incremented by the number of
column updates in the mutations. The next count is also incremented by the
same amount, so it always ends up as one more than the current count.

(The next count is actually atomically incremented before the mutations are
performed, as a way of reserving the logical timestamps for the updates. This
feature is a remnant from before writes were serialized, as explained next.)

Because the `InMemoryMap` can be called in parallel for processing mutations,
a lock object is used to serialize processing of the mutations, so that all of
the mutations for one caller are performed before the next caller's are. In
addition, the current count and next count values are adjusted within that
block. The lock object is specifically for writes, so that reads can occur
without blocking.

### Sorted Key/Value Iteration

The sorted key/value iterator returned for an `InMemoryMap` is a custom
implementation called `MemoryIterator`. It is a wrapping iterator that wraps
a `PartialMutationSkippingIterator`, initialized with the current count. This
means that the iterator will not "see" any mutations that are processed after
iterator construction.

The `InMemoryMap` keeps track of all `MemoryIterator` instances it has given
out. When a `MemoryIterator` is closed, it is no longer tracked. Also, when
the map is deleted (more on this below), the deletion process waits for some
time to let active (tracked) iterators complete.

You may recall that a `PartialMutationSkippingIterator` is also a wrapped
iterator. The iterator it wraps is a core `SourceSwitchingIterator` that
handles memory dump files. More on this is described later.

### Compaction Iterator

The sorted key/value iterator is returned for calls to
`InMemoryMap.compactionIterator()`. The call also checks that the map's
current count is one less than its next count, as a sanity check.

### Deletion

An `InMemoryMap` will wait for a given period of time when `delete()` is called
on it, until either the time elapses or all active iterators have been closed.
If all of the iterators do close, then the wrapped map is simply deleted.

If iterators remain active after the timeout, then the contents of the wrapped
map are dumped to a memory dump file on the local file system, so that the
iterators can continue to function. The directory for the file is provided to
the `InMemoryMap` instance when it is constructed; the file name includes a
random UUID.

The dump file itself is an RFile, and so it contains key/value pairs grouped by
locality groups. Since the RFile format cannot preserve the key/value count
saved with each `MemKey`, the `MemValue` merging process is used to save the
count within the value data.

After the dump file is created, each of the remaining active iterators is
switched. This causes those iterators' underlying `SourceSwitchingIterator`
instances to switch over to the memory dump file. The iterator ensures that
the switched-out iterators seek to the next correct location, so that from the
view of the clients, the switch is seamless.

The iterator used to read from the memory dump file is a
`MemKeyConversionIterator`, which properly restores the key/value counts upon
reading.
