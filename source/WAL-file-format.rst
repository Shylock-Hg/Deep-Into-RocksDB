WAL file format
=================

| The Write-Ahead-Log file persist the sequence of data/operation will apply to
 *MemTable* later.
| This file are separate to fixed-length block on top level which stepped by
 *kBlockSize*.The *record* which too long will be splitted to multiple
 fragments which specified by *type* (First, Middle and Last).The block can't
 be full fit will be padded

Top-level
-----------

+----+---+--+-------------+
|r0  |r1 |p |r2           |
+----+---+--+-------------+
| kBlockSize| kBlockiSize |
+-----------+-------------+

| Where r* is *record*, p is the padding for fixed length, block* is fixed
 length *block*.

Record format
--------------

+----------+------------+------------------------------------+
| label    | type       | note                               |
+----------+------------+------------------------------------+
| CRC32    | fixed32    |                                    |
+----------+------------+------------------------------------+
| Size     | fixed32    |                                    |
+----------+------------+------------------------------------+
| Type     | fixed8     | kZeroType, kFullType, kFirstType,  |
|          |            | kMiddleType, kLastType             |
+----------+------------+------------------------------------+
| Payload  | char[Size] |                                    |
+----------+------------+------------------------------------+
