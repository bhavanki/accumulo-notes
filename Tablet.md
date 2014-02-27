# A Tour of Tablet

This is a tour of the nearly 4000 lines of code that constitute Accumulo's
`Tablet` class. The tour is conducted from the beginning of the file to the
end.

You should understand `InMemoryMap` before taking this tour.

## MinorCompactionReason

Minor compaction is the process of flushing tablet data to storage. This class
is an enum for the reasons why minor compaction happens: a user asked for it,
the system decided to do it, or the server is closing.

## CommitSession

A `CommitSession` is something.
It is an inner class of `Tablet` and so associated with a tablet instance.

A `CommitSession` is built from two pieces of data: a write-ahead log sequence
number and a reference to an in-memory map (see below). The session also keeps
track of a number of commits in progress; when the count is decremented to
zero, threads waiting on the session's tablet (which can do so by asking the
session to loop the waiting) are notified. Finally, the session can track the
maximum (most recent) commit time.

A commit session provides context for committing and aborting the commit of
mutations and for something else having to do with logging.

## TabletMemory

`TabletMemory` is a (non-static) inner class that maintains the in-memory
storage of tablet data. It holds a trio of `InMemoryMap` instances for this:

1. "memTable", the primary storage map
2. "otherMemTable", a temporary map variable used when performing minor
   compaction
3. "deletingMemTable", another temporary map variable for minor compactino

When `TabletMemory` is first created, it initializes memTable and creates a
`CommitSession` for it with a sequence number of one. The commit session can
be retrieved at any time.

### Minor Compaction Phases

`TabletMemory` helps to coordinate minor compaction with three phases by
juggling the map initially in memTable to the other map variables. As compaction
moves through these phases, memory usage statistics are kept up-to-date.

#### Preparation

To prepare for minor compaction, the map in memTable is moved to otherMemTable,
and a fresh map is created for memTable. A new `CommitSession` is also built
for the new map with a sequence number two more than the last one. The "old"
commit session for otherMemTable is returned.

This phase fails if a minor compaction is already in progress.

#### Finishing

When minor compaction is finished, oldMemTable is moved to deletingMemTable,
and threads waiting on the tablet are notified.

This phase fails if a minor compaction is not yet in progress, or if a prior
minor compaction has not yet been finalized.

#### Finalizing

To finalize minor compaction, deletingMemTable is deleted (freed), with a
fifteen second wait time for any active iterators to complete before the table
is dumped to a file.

This phase fails if a minor compaction has not just finished.

### Iterators

`TabletMemory` can provide iterators for both memTable and otherMemTable if
it is in use. It can also close each iterator in a given list, the idea being
that the iterators are being "returned".

## DataFileManager

The `DataFileManager` for a tablet manages the set of files that contain the
tablet data. The manager is initially constructed with the set of files passed
to the tablet constructor, in fact, a map of `FileRef` keys and `DataFileValue`
values. This map will be called the "tablet file map". The map contents change
over time. A copy of the map can always be retrieved, as well as a copy of just
the `FileRef` keys as a set.

### FileRef

A `FileRef` object refers to an RFile kept under some file system. The RFile
holds table data. The file is presumed to be kept under a special table
directory structure maintained by Accumulo.

The absolute URI to the file is called its "full reference", while an
optionally abbreviated / relative URI is called its "meta reference". A
`FileRef` object contains one of each. While each can be supplied upon
construction, a `VolumeManager` can convert a meta reference, stored as a
column qualifier in a `Key`, into a full reference. --- metadata table ---

A `FileRef` object also extracts a "suffix" from a full reference. The suffix
is not the file extension, but rather the relative path of the file beneath
the top-level Accumulo table directory.

### DataFileValue

A `DataFileValue` holds three values: a size, a number of entries, and a
timestamp. The object can be encoded as a byte array and decoded back. Two
objects are compared by their size and number of entries only.

### Scanning

The set of tablet files for a scan must remain constant. The `DataFileManager`
allows a thread to reserve those files. A reservation consists of a one-up ID
and the set of `FileRef` objects at the time of reservation. The reserving
thread receives both the reservation ID and a snapshot of the tablet file map.
Meanwhile, the `DataFileManager` increments a reference count for each file
in the reservation.

When the scan is done, the thread can return the files by just providing the
reservation ID. The `DataFileManager` removes the reservation and decrements
the reference count for each file in it. When the reference count for a file
reaches zero, it is deleted from the metadata table if it has been tagged for
deletion (say, because it has been major compacted).

A thread can wait for scans that cover any of a set of files to complete. The
wait method waits until the scan reference count for each file reaches zero,
or until a maximum wait time elapses. Any files which are still in use are
returned from the wait method. (The process does allow for a file whose
reference count hits zero to then be used for a new scan, thus becoming "in
use" by the end.) The waiting thread can optionally block any new scan
reservations while it is waiting.


- reserve for scan: get id and map snapshot, ref counts kept
- return from scan: decrement counts, prep some for deletion, then delete
- bulk import existing files: check if in right dir, remove ones already done,
  optional set timestamps on import, update metadata table
- reserve a "merging minor compaction file": usually null, but if the map has
  the max tablet files or more, finds the smallest one
- unreserve it: nulls it out
- bring minor compaction online: ???
- reserve major compacting files: remembers them
- clear major compacting files: clears them
- bring major compaction online: ???

---- LogEntry ----

`LogEntry` is a core class specific to tablets. Each entry is a set of fields:

* a key extent
* a timestamp
* a server name
* a filename
* an integer tablet ID
* a "log set" as a collection of strings

A `LogEntry` instance can be written to and read from a byte array.

The data in a `LogEntry` can be mapped to an Accumulo metadata row (key/value
pair) as follows:

* metadata entry string for the key extent <=> row
* "log" <=> column family
* server + "/" + filename <=> column qualifier
* timestamp <=> key timestamp
* log set joined with ";" + "|" + tabletID <=> value

---- TabletFiles ----

`TabletFiles` is a simple data structure holding:

* a directory name
* a list of `LogEntry` instances
* a sorted map of `FileRef` keys to `DataFileValue` values

## Construction

There are no less than six constructors for `Tablet`, but all of them end up
at one comprehensive private constructor.

### Constructor Parameters

The tablet server is a reference to the server that is serving the tablet.
There are frequent calls between a tablet and its server.

The location is the directory where the tablet's files reside. This directory
is kept in both the "location" (as a `Path`) and "tabletDirectory" (as a
`String`) fields. The location that is passed in may be updated through either
volume URI replacements in the Accumulo configuration or by responding to the
decommissioning of the location.

The key extent is the span of keys covered by the tablet.

The tablet resource manager is used to manage resources involved in tracking
memory use, running compactions, and other operations.

The configuration is simply the overall Accumulo configuration in effect. The
tablet also creates a table configuration based on the system configuration.

The volume manager provides access to the file systems holding tablet files.

Log entries are supplied if write-ahead log recovery is required. The recovery
work is run by the tablet server, but the tablet listens for each mutation and
performs (replays) it on tablet memory. The tablet keeps track of the number of
mutations replayed and the largest timestamp found among the column updates.
If no mutations were replayed, then the logs are cleared out; otherwise, each
log listed in the entries' log sets are set up as current loggers. The largest
timestamp encountered is possibly set as the tablet time (see below).

The map of file references to data file values is used to set up the tablet's
`DataFileManager` object. The file references (keys) are also passed along to
help with write-ahead log recovery.

The time string is interpreted as the "tablet time". Tablet time can be tracked
either as physical time in milliseconds (e.g., M12345) or as a logical time
(e.g., L100). The time string passed in may be ignored if the tablet is the
root tablet; in that case, the files referred to in the map of file references
to data file values are scanned, and the largest logical time found is used
instead, provided it is larger than the passed-in value. Also, if write-ahead
log recovery is performed, the largest timestamp there is used instead, provided
it is yet larger.

The last location is not a directory, but ---- has something to do with
splits. ----

The scan files are tagged for removal in the `DataFileManager` objects after
all scans against them have completed.

The initial flush ID and initial compact ID ---- have something to do with
splits and are remembered as the "last" ones. ----

### Other Tablet Fields Set during Construction

Provided write-ahead logging is enabled, the tablet gets a log ID from its
tablet server.

The tablet creates a new `TabletStats` structure, which can track statistics
on major compactions, minor compactions, and splits. Curiously, the field
holding the structure is called "timer".

A default security label is determined during construction. It is usually
blank - always for metadata tablets - but if a default scan time visibility
is configured, its expression is used.

A `TabletMemory` field is always created. It is ready to receive mutations
generated during write-ahead log recovery.

The time passed in to the constructor, possibly replaced for the root tablet,
is saved as "persistedTime". The tablet time determined during write-ahead log
recovery is not considered for this field.

As described above, the constructor creates a table-specific configuration. It
also adds a custom observer to the configuration to support reloading when the
configuration changes.

If write-ahead recovery occurs, the "currentLogs" field is initialized with
logs derived from the `LogEntry` objects used in recovery.

Once the `DataFileManager` is created, the number of tablet entries total and
the number just in memory are calculated and saved as fields.

### Other Activities during Construction

If a table classpath is configured, a VFS classloader for it is initialized.

If the tablet appears to have been created after a failure in a previous
incarnation, then an effort is made to find and remove old temporary tablet
files.

### Processing Key/Value Pairs

All of the constructors except the comprehensive one take in a sorted map of
key/value pairs, representing the metadata table. This map is used to figure out
many of the parameters passed to that comprehensive constructor.

* The tablet time saved for the tablet extent is looked up. It is expected that
  only one key/value pair for the tablet time is present.
* Log entries (`LogEntry` instances) are generated from appropriate key/value
  pairs found for the tablet extent.
* Data files (`FileRef` and `DataFileValue` instances) are generated from
  appropriate key/value pairs found for the tablet extent.
* Scan files (`FileRef` objects) are generated from appropriate key/value pairs
  found for the tablet extent.
* ---- Split-related fields too ----

It should be noted that special handing is performed when the tablet being
constructed is either the root tablet or a metadata tablet.

### Filling in the Blanks

Most of the `Tablet` constructors can be called without every single possible
parameter. In those cases, values for the remaining parameters through
processing metadata key/value pairs as described above, or are found as
follows:

* If a configuration is not passed, a `CachedConfiguration` is retrieved.
* If a volume manager is not passed, an instance based on the system
  configuration is retrieved.
* If the new tablet is being constructed due to a split, then its log entries
  and scan files start out empty.

## Lookup

The public `lookup` method is used for looking up tablet data across ranges.

### `KVEntry`

Lookup results are provided as `KVEntry` objects. `KVEntry` extends the core
`KeyValue` class and so includes a `Key` and value bytes as an array.
Additional methods provide the number of bytes taken up by the key and value
and an estimate of their total memory usage.

### `LookupResult`

Key/value pairs retrieved in a lookup are returned as a side effect of the
lookup, in a list. The direct result of a lookup is a `LookupResult` object.
This object tracks information about lookup progress, including:

* a list of "unfinished ranges" through which lookup could not be done
* the total memory usage of all lookup results (as "bytesAdded")
* the number of data bytes taken up by all keys and values (as "numBytes")

### Basic Lookup Procedure

Lookup is performed across one or more `Range` objects. Before scanning starts,
overlapping ranges are merged, and each is checked to ensure it resides within
the tablet. Then, a special iterator is created to perform the scan (details
on it later).

Each range is processed in turn. The iterator is seeked to the beginning of
the range, and each key/value pair iterated is wrapped in a `KVEntry` and added
to the result list. Along the way, the data byte and memory usage totals are
accumulated.

When the lookup is complete, the tablet's query count is increased by the number
of results, and the tablet's query bytes count is increased by the total data
byte count of the results.

### Unfinished Ranges

A lookup can be stopped partway for the following reasons:

* when the total memory usage of the lookup results exceeds the maximum results
  size specified for the lookup
* when the tablet is closed during the lookup, possibly due to shutdown
* when the tablet ends up with too many files

If any of this conditions is detected, then the range that is being scanned,
along with any further ranges yet to be processed, are recorded in the
`LookupResult` as unfinished ranges. If the range being scanned had results
already entered into the result list, then the unfinished range for it is
shrunk to only span the entries not yet scanned in.



