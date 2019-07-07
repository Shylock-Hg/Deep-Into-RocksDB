Top
======

There is the top of RocksDB.

::

           > log files
           |
           | sequencial write
           |
  API ------> MemTable(current in list) ... ImmutableMemTable
      GET/PUT                                      |
                      async-flush (worker threads)  |
                                                   V
                                          0-level SSTable files (unsorted)
                                                   |
                   async-compact (worker threads)  |
                                                   V
                                              SSTable files (sorted)

0. MemTable
------------

The class architecture as below:

::

                          MemTableList
                               |
                               | combine
                               V
                         MemTableVersion (MemTable List)
                               |
                               | combine
                               V
                            MemTable
                               |
                               | combine
                               V
                           MemTableRep
                               |
                               | inherit
         ----------------------------------------------------
         |               |                 |                | factory classes
         V               V                 V                V
   HashLinkListRep  HashSkipListRep   SkipListRep       VectorRep

.. warning::
    There is *MockMemTableRep* (with also factory) for testing.

1. SSTable files
------------------

::

                        L0 files  (flush and unsorted, duplicate keys maybe
                            |      sequence-based linear search,
                            |      compact to L1 when count of files reached)
                            V
                        L* files  (compact and sorted, unique key in one level
                                   key-based binary search,
                                   compact to lower level when size reached)

The class architecture as below:

::

                                      TableBuilder/Reader
                                             |
                                             | inherit
                                             V
            --------------------------------------------------------------
            |                                |                           | factory classes
            V                                V                           V
    BlockBasedTableBuilder/Reader  CuckooTableBuilder/Reader  PlainTableBuilder/Reader

.. warning::
    There are *MockTableBuiler/Reader* (with also factory) for testing.

