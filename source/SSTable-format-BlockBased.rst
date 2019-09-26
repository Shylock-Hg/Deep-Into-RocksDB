SSTable format Block Based
===============================

.. note::
    Block-Based format SSTable is the default format of RocksDB SSTable.

| The full name of SSTable is `Static Sorted Table` which the basic file to
 store the data in RocksDB.

Some Basic Structure
----------------------

The format of BlockHandle as below:

+-----------+-----------+
| label     | type      |
+-----------+-----------+
| offset    | varint64  |
+-----------+-----------+
| size      | varint64  |
+-----------+-----------+

Top Level Structure
--------------------

+----------------------+--------------------+---------------------------------+
| label                | type               | note                            |
+----------------------+--------------------+---------------------------------+
| data blocks          | data_block[N]      | sorted order                    |
+----------------------+--------------------+---------------------------------+
| meta blocks          | meta_block[K]      | filter, stats,                  |
|                      |                    | compression dictionary,         |
|                      |                    | range deletion                  |
+----------------------+--------------------+---------------------------------+
| metaindex blocks     | metaindex_block[K] |                                 |
+----------------------+--------------------+---------------------------------+
| index blocks         | index_block[N]     | N for one-level index,          |
|                      |                    | Not N for two-level index       |
+----------------------+--------------------+---------------------------------+
| footer               | footer             | fixed size                      |
+----------------------+--------------------+---------------------------------+

0. Data Block
--------------

.. note::

    This Block is built by `block_builder.cc` in sources of RocksDB.

The *data block* format as below:

+------------------------+-------------------------+
| label                  | type                    |
+------------------------+-------------------------+
| groups                 | group[groups_count]     |
+------------------------+-------------------------+
| groups_offset          | fixed32[groups_count]   |
+------------------------+-------------------------+
| groups_count           | fixed32                 |
+------------------------+-------------------------+
| compress_type          | char                    |
+------------------------+-------------------------+
| CRC32                  | fixed32                 |
+------------------------+-------------------------+

The *data group* format as below:

+--------------------+-----------------------+--------------------------+
| label              | type                  | note                     |
+--------------------+-----------------------+--------------------------+
| shared_bytes       | varint32              | compress prefix of key,  |
|                    |                       | 0 for restart point      |
+--------------------+-----------------------+--------------------------+
| unshared_bytes     | varint32              | unshared key bytes length|
+--------------------+-----------------------+--------------------------+
| value_length       | varint32              | value length             |
+--------------------+-----------------------+--------------------------+
| key_delta          | char[unshared_bytes]  | unshared key bytes       |
+--------------------+-----------------------+--------------------------+
| value              | char[value_length]    | value bytes              |
+--------------------+-----------------------+--------------------------+

1. Meta Block
---------------

.. note::

    The meta block is built by `block_builder.cc` in sources of RocksDB too.

1.0. Filter Meta Block
```````````````````````

| 1. Full filter block for entire SSTable.
| 2. Partitioned filter for too big filter block.In this case, the are two-level
   of filter, the one is the top-level index for 2nd filter block, the other is
   the real of filter meta block.

1.1. Properties Block
```````````````````````

+------------+----------------+
| label      | type           |
+------------+----------------+
| props      | K/V[P]         |
+------------+----------------+

Default properties as below:

- data size
- index size
- filter size
- raw key size  ; size of key before any process(such as compress etc.)
- raw value size  ; size of value before any process(such as compress etc.)
- number of entries
- number of data block

1.2. Compression Dictionary
````````````````````````````

.. note::

    This only apply to bottommost level.

1.3. Range Deletion
``````````````````````

.. note::

    Can only be obsoleted during compaction to the bottommost level.

+-----------------+----------------------------------------------------------+
| label           | note                                                     |
+-----------------+----------------------------------------------------------+
| User Key        | the range's begin key                                    |
+-----------------+----------------------------------------------------------+
| Sequence Number | the sequence number at which range-deletion was inserted |
|                 | to the DB                                                |
+-----------------+----------------------------------------------------------+
| Value Type      | kTypeRangeDeletion                                       |
+-----------------+----------------------------------------------------------+
| Value           | the range's end key                                      |
+-----------------+----------------------------------------------------------+

2. Meta Index
----------------

The entry of each metaindex block.

| K : name of the metablock
| V : BlockHandle point to corresponding metablock.

+---------------------+--------------------+
| label               | type               |
+---------------------+--------------------+
| metaindex blocks    | metaindex_block[K] |
+---------------------+--------------------+

3. Index Block
---------------

3.0. One-Level
```````````````

The entry of each data block.

| K : string >= last key and before first key in successive data block.
| V : BlockHandle

+-------------------+------------------+
| label             | type             |
+-------------------+------------------+
| index blocks      | index_block[N]   |
+-------------------+------------------+

3.1. Two-Level
```````````````

.. note::

    If enable kTwoLevelIndexSearch

+--------------------+---------------+
| label              | type          |
+--------------------+---------------+
| 1st index blocks   |               |
+--------------------+---------------+
| 2nd index blocks   |               |
+--------------------+---------------+

4. Footer
-----------

+---------------------+-------------+---------------------------------------+
| label               | type         | note                                 |
+---------------------+--------------+--------------------------------------+
| metaindex_handle    | char[p]      | BlockHandle in fact                  |
+---------------------+--------------+--------------------------------------+
| index_handle        | char[q]      | BlockHandle in fact                  |
+---------------------+--------------+--------------------------------------+
| padding             | char[40-p-q] | zero for padding to fixed length     |
+---------------------+--------------+--------------------------------------+
| [magic]             | fixed64      |                                      |
+---------------------+--------------+--------------------------------------+
