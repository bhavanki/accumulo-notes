# Understanding SourceSwitchingIterator

The `SourceSwitchingIterator` class is heavily used to implement many of
Accumulo's advanced, adaptive iteration behaviors. This document attempts to
demystify how it works.

In a nutshell, a source-switching iterator automatically switches from one
source of key/value pairs to another as sources become no longer current.

## DataSource

A source-switching iterator essentially wraps a class that implements the
`DataSource` interface. A `DataSource` can supply a sorted key/value iterator.
Iteration for a source-switching iterator, in the most basic case, uses the
iterator supplied by the data source.

A data source can determine whether it is "current". The meaning of "current"
depends on the data source. However, a data source that is not current is
"switched" by the source-switching iterator. The non-current data source itself
is called upon to supply a new data source. If that one is also not current,
then the process continues. It is common for a `DataSource` implementation to
return itself as a new data source after adjusting its internal state to
become current again.

## Basic Usage

A source-switching iterator cannot be initialized. The first action on a
source-switching iterator must be to seek. After a seek, `next()` may be
called. A seek results in the same seek being performed on the data source's
iterator.

The top key and value can be retrieved, and the iterator can be checked for
whether it has a top (i.e., whether iteration has completed). The top key
returned by the iterator is a clone of the key supplied by the data source's
iterator.

## Source Switching

At the beginning of most `next()` calls, including the one that automatically
occurs with a seek, the data source is checked for whether it is current. If
not, it is switched. The new iterator from the switched data source is then
seeked to pick up where the last one left off.

A switch can also be explicitly attempted by calling `switchNow()`. The source
will not be switched if it is still current.

### Only Switching After Rows

A source-switching iterator can be created so that source switching only occurs
when iteration proceeds to a new row. When this restriction is in effect, then
a `next()` call will not first check its data source for switching. Instead,
the data source iterator is used to get the next item, and if the row has
changed from the last item, the source is switched at that time, and the new
iterator is seeked to pick up where the last one left off.

Calls to `switchNow()` fail on a source-switching iterator that only switches
after rows.

## Interruption

A source-switching iterator is interruptible. When the interrupt flag is set,
it is also set on the data source iterator. The flag is also set on new
iterators arising from subsequent seeks or source switches.

## Deep Copies

A deep copy of a source-switching iterator can be made. The copy has a
corresponding deep copy of the current data source (another part of the
`DataSource` interface) and the same interrupt flag and restriction on
switching only after rows.

Once a deep copy of a source-switching iterator has been made, the interrupt
flag may not be set on it or any of its copies.

When any source-switching iterator among a set of copies is manually switched
by `switchNow()`, all of them are switched.

## Some DataSource Implementations

### MemoryDataSource

The tablet server's `InMemoryMap` class uses a `MemoryDataSource` object for
switching from iterating over in-memory key/value pairs to those stored in
dump files on disk.

The initial state of the data source is current, reading from memory. The source
reports itself as non-current when it first sees that a dump file has been
created; when asked for a new data source, it returns itself after setting its
internal state to "switched". From that point on, it always considers itself
current again.

Before a `MemoryDataSource` is switched, it provides an iterator over in-memory
key/value pairs. Afterwards, it provides an iterator that reads from the dump
file.

### ScanDataSource
