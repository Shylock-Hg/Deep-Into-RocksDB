Files
=======

The files used by RocksDB.

Overview
---------

.. note::
    The different *Column Families* share `WAL` and `MANIFEST`.

+----------------------+------------------------------------------------------+
|   MANIFEST-\\d{6}    |   Contains snapshot of RocksDB state and subsequent  |
|                      |   modifications                                      |
+----------------------+------------------------------------------------------+
|       CURRENT        |   File pointer to last MANIFEST                      |
+----------------------+------------------------------------------------------+
|       \\d{6}.log     |   Write-Ahead-Log files persist MemTable             |
+----------------------+------------------------------------------------------+
| LOG|LOG.old.\\d{16}  |   Expired Write-Ahead-Log files                      |
+----------------------+------------------------------------------------------+
|       \\d{6}.sst     |   SSTable files to persist data                      |
+----------------------+------------------------------------------------------+


MANIFEST
---------

| The `CURRENT` file contains the current MANIFEST file.The `MANIFEST-\\d{6}`
 files contains the snapshot of RocksDB.

WAL
-----

| The *Write Ahead Files* persist the data/operation will apply to *MemTable*.
 And these files will be deleted(or archive) after all data persist to SST
 files.

SST
-----

| The *Static Sorted Table* files persist the data which we want to storage.
 These are leveled from 0 to N.The level 0 files are time-based sorted
 (flush directly) among and key-based sorted inner.The level 1 to N files are
 key-based sorted among and inner.So we can apply *Binary Sort Algorithm* to
 key-based sorted structure.
