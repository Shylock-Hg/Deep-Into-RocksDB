SSTable format
===============

| The full name of SSTable is `Sorted Static Table` which the basic file to
 store the data in RocksDB.

Some Basic Structure
----------------------

..code-block::

    BlockHandle:
        offset: varint64
        size:   varint64

Top Level Structure
--------------------

| <begin of SSTfile>
| 0. [data block] * N  ; sorted order
| 1. [meta block] * K  ; filter, stats, compression dictionary, range deletion
| 2. [metaindex block] * K
| 3. [index block] * N ; N for one-level index, Not N for two-level index
| 4. [footer]  ; fixed size
| <end of SSTfile>

Data Block
-----------

..note::
    This Block is built by `block_builder.cc` in sources of RocksDB.

| <begin of data block>
| [group]
| ...
| [groups_offset] : fixed32[groups_count]
| [groups_count]  : fixed32
| [compress_type] : char
| [CRC32]         : fixed32
| <end of data block>

| <begin of group>
| [shared_bytes]   : varint32  ; compress prefix of key, 0 for restart point
| [unshared_bytes] : varint32  ; unshared key bytes length
| [value_length]   : varint32  ; value length
| [key_delta]      : char[unshared_bytes]  ; unshared key bytes
| [value]          : char[value_length]    ; value bytes
| <end of group>

Meta Block
-----------

..note::
    The meta block is built by `block_builder.cc` in sources of RocksDB too.

Filter Meta Block
````````````````````

| 1. Full filter block for entire SSTable.
| 2. Partitioned filter for too big filter block.In this case, the are two-level
   of filter, the one is the top-level index for 2nd filter block, the other is
   the real of filter meta block.

Properities Block
````````````````````

| <begin of Properities Block>
| [prop0]  ; K/V
| ...
| <end of Properities Block>

Default propertiies as bellow:
 - data size
 - index size
 - filter size
 - raw key size  ; size of key before any process(such as compress etc.)
 - raw value size  ; size of value before any process(such as compress etc.)
 - number of entries
 - number of data block

Compression Dictionary
``````````````````````````

..note::
    This only apply to bottommost level.

Range Deletion
````````````````````

..note::
    Can only be obsoleted during compaction to the bottommost level.

 - User Key : the range's begin key
 - Sequence Number : the sequence number at which range-deletion was inserted
   to the DB
 - Value Type : kTypeRangeDeletion
 - Value : the range's end key

Meta Index
-------------

The entry of each metaindex block.

| K : name of the metablock
| V : BlockHandle point to corresponding metablock.

| <begin of metaindex>
| [metaindex]
| ...
| <end of metaindex>

Index Block
---------------

One-Level
`````````````

The entry of each data block.

| K : string >= last key and before first key in sucessive data block.
| V : BlockHandle

| <begin of index>
| [index block]
| ...
| <end of index>

Two-Level
````````````

..note::
    If enable kTwoLevelIndexSearch

| <begin of index>
| [index block 1st]
| ...
| [index block 1st]
| [index block 2nd]
| ...
| [index block 2nd]
| <end of index>

Footer
-------

| <begin of Footer>
| [metaindex_handle] : char[p]
| [index_handle]     : char[q]
| [padding]          : char[40-p-q]  ; zero for padding to fixed length
| [magic]            : fixed64
| <end of Footer>
