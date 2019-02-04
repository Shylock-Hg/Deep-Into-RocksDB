Plain SSTable format
=====================

| *PlainTable* is a RocksDB's SST file format optimized for low query latency on
 pure-memory or really low-latency media.

Top-Level
------------

+--------------+---------------+
| label        | note          |
+--------------+---------------+
| data row0    |               |
+--------------+---------------+
| ...          |               |
+--------------+---------------+
| data rowN    |               |
+--------------+---------------+
| property     |               |
+--------------+---------------+
| footer       | fixed size    |
+--------------+---------------+

0. Row Format
---------------

.. note::
    The format of data row.

+------------------+------------------+
| label            | type             |
+------------------+------------------+
| encoded_key      |                  |
+------------------+------------------+
| value_size       | varint32         |
+------------------+------------------+
| value            | char[value_size] |
+------------------+------------------+

0.0 Key Encoding
``````````````````

1. Plain Encoding
    - internal encoding with fixed given key size
    - [length of key : varint32] + [user key] + [internal encoding] when
      without fixed given key size

2. Prefix Encoding

.. note::
    Share the same prefix of keys to save size.

There are three type packets as below:

| - Full Key: with the full key bytes. [full key flag + size] +
    [full user key] + [internal encoding]
| - Second Key: with the prefix size. [prefix key flag + size] +
    [suffix key flag + size] + [suffix key] + [internal encoding]
| - Others: with the suffix key bytes. [suffix key flag + size] +
    [suffix key] + [internal encoding]

| The [flag + size] is the byte with format as below:
| [type(2b)|size(6b)]. But all bits of size will be set to 1 when size
    beyond the limit, and there will be *varint32* writen after this and the
    size are the sum of 0x3F and value of variable size.The type are *full key*
    , *second key* and *suffix key*.

3. Internal Encoding

| In both of Plain and Prefix encoding data, internal encoding of the internal
 are encoded in the same way. The internal encoding seems as below:

+--------------+----------+-------------------------------------+
| label        | type     |             note                    |
+--------------+----------+-------------------------------------+
| type         | char     | row type(value, delete, merge, etc.)|
+--------------+----------+-------------------------------------+
| sequence ID  |  char[7] |                                     |
+--------------+----------+-------------------------------------+

| This can be compressed as below when no previous value for this key in
 the system.

+------+
| 0x80 |
+------+

1. Property
------------

| 1. data_size : the end of data part of the file.
| 2. fixed_key_len : length of the keys if all keys has the same length,
 0 otherwise.


In-Memory Index
-----------------

.. warning::
    the In-Memory Index was built by scan the Plain SSTable file. So this is
    not a part of Plain SSTable file now.

| On top level, In-memory Index is the hash table with each bucket to be either
 offset in the file or a binary search index. The binary search buffer is
 needed in two cases:

| 1. Hash Collisions: two or more prefixes are hashed to the same bucket.
| 2. Too many keys for one prefix: need to speed-up the look-up inside the
 prefix.

Format
```````

| The index consists of two piece fo memory: an array as hash buckets, and some
 binary search buffers.

+-------------+---------------------------------------------+
| record                                                    |
+-------------+---------------------------------------------+
| Flag(1b)    | Offset to binary search buffer or file(31b) |
+-------------+---------------------------------------------+

| 1. If Flag = 0 and Offset equals to the offset of end of the data of the file,
 it means NULL - no data for this bucket; if the offset is smaller, it means
 there is only one prefix for this hash bucket.
| 2. If Flag = 1, it means the offset is for binary search buffer.

The format of binary search buffer is as below:

+-------------------+-------------------------------------+
| label             | type                                |
+-------------------+-------------------------------------+
| number_of_records | varint32                            |
+-------------------+-------------------------------------+
| records           | fixed32[number_of_records]          |
+-------------------+-------------------------------------+
