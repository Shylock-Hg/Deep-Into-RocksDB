Flush
=======

The data transform
--------------------

::

    autovector<MemTable*>       -------->  vector<unique_ptr<FragmentedRangeTombstoneIterator>>
            |                                       |
            |                                       |
            V                                       V
    vector<InternalIterator*>          --- unique_ptr<CompactionRangeDelAggregator>
            |                          |            |
            |                          |            |
            V                          |            V
    ScopedArenaIterator(MinHeap)       |   unique_ptr<FragmentedRangeTombstoneIterator>
            |                          |            |
            |                          |            |
            V                          |            |
    CompactionIterator       <----------            |
            |             Remove Key Deleted        | Write Tombstone
            |                                       |
            V                                       |
    TableBuilder*, FileMetaData*    <----------------
            |
            |
            V
    L0-level SST files (inner sorted, inter unsorted)

