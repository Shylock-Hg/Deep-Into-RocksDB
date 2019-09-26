Put
======

The key/value put operation flow:

::

                                                -> WALOnly
    DB::Put -> WriteBatch::Put -> DB::Write ->  -> unorderedWrite
                                                -> pipelineWrite

    DBImpl::WriteWALOnly -> DBImpl::ConcurrentWriteToWAL -> DBImpl::WriteToWAL
    -> Writer::AddRecord

    DBImpl::UnorderedWrite -> WriteBatchInternal::InsertInto
    -> WriteBatch::Iterate -> MemTableInserter::PutCF -> MemTable::Add/Update

    DBImpl::PipelineWrite -> WriteThread::JoinBatchGroup
    -> WriteThread::EnterAsBatchGroupLeader -> DBImpl::WriteToWAL
    -> WriteThread::ExitAsBatchGroupLeader -> WriteThread::EnterAsMemTableWriter
    -> WriteThread::LaunchParallelMemTableWriters/WriteBatchInternal::InsertInto
    -> WriteThread::ExitAsMemTableWriter

    SimpleWrite WriteThread::JoinBatchGroup->WriteThread::EnterAsBatchGroupLeader
    -> DBImpl::WriteToWAL/DBImpl::ConcurrentWriteToWAL
    -> WriteBatchInternal::InsertInto/WriteThread::LaunchParallelMemTableWriters
    -> WriteThread::CompleteParallelMemTableWriter
    -> WriteThread::ExitAsBatchGroupLeader

Multiple Threads Write:

::

                                            Group
                       Thread1              Thread2                 Thread3
                          |                  |                         |
                          V                  V                         V
                                       Join BatchGroup
                          |                  |                         |
                          V                  V                         V
               Enter As BatchGroupLeader          -->  AwaitState
                          |                       |
                          V           |------------
                   Batch Commit WAL   |                    |
                          |           |                    |transform to follower
                          V           |                    |
            Launch Parallel Followers--                    |
                          |                                |
                          V                                V
                                      Insert MemTable
                                            |
                                            V
                                  Complete Parallel Writers
                          |
                          V
            Exit As BatchGroupLeader ------> Notify Successor Leader

                                        |
                                        v
                            Write End and Return


Multiple Threads Pipeline Write:

::

                                            Group
                       Thread1              Thread2                 Thread3
                          |                  |                         |
                          V                  V                         V
                                       Join BatchGroup
                          |                  |                         |
                          V                  V                         V
               Enter As BatchGroupLeader          -->  AwaitState
                          |                       |           |
                          V              |---------           |
                   Batch Commit WAL      |                    |
                          |              |                    |transform to follower
                          V              |                    |
                Exit As BatchGroupLeader --> Notify Successor |
                          |                                   |
                          V                                   V

                                    EnterAsMemTableWriter
                                            |
                                            V
                                      Insert MemTable
                                            |
                                            V
                                    ExitAsMemTableWriter
                                            |
                                            v
                                    Write End and Return

The State of Writer:

+----------------------------------+------------------------------------------+
| state name                       | note                                     |
+----------------------------------+------------------------------------------+
| STATE_INIT                       | Created                                  |
+----------------------------------+------------------------------------------+
| STATE_GROUP_LEADER               | JoinBatchGroup, as Leader if first Join  |
+----------------------------------+------------------------------------------+
| STATE_MEMTABLE_WRITER_LEADER     | Serial Write Leader                      |
+----------------------------------+------------------------------------------+
| STATE_PARALLEL_MEMTABLE_WRITER    | Parallel Write Leader                   |
+----------------------------------+------------------------------------------+
| STATE_COMPLETED                  | Complete Write                           |
+----------------------------------+------------------------------------------+
| STATE_LOCKED_WAITING             | Waiting                                  |
+----------------------------------+------------------------------------------+

The State Transform:

::

                JoinBatchGroup                       ConcurrentWriteToWAL
    STATE_INIT  ---------------> STATE_GROUP_LEADER ----------------------> STATE_COMPLETED
        |                          ^     |                                      ^
        | Waiting Previous   |-----|     |                                      |
        V Writing            |           V                                      |
    STATE_LOCKED_WAITING -----      STATE_LOCKED_WAITING              STATE_LOCKED_WAITING
                                        |                                       ^
                                        |------> STATE_MEMTABLE_WRITER_LEADER --|
                                        |------> STATE_PARALLEL_MEMTABLE_WRITER-|

The data structure transform:

| 1. The (K:string, V:string) tuple user given.
| 2. The WriteBatch rep as below:

+--------+------------+---------------------------------------------+
| Label  | Type       | note                                        |
+--------+------------+---------------------------------------------+
|  Seq   | Byte[8]    | Sequence Number                             |
+--------+------------+---------------------------------------------+
|  count | Fixed32    | KV count                                    |
+--------+------------+---------------------------------------------+
|  kType | Byte       | Key Type                                    |
+--------+------------+---------------------------------------------+
|  kLen  | Var32      | Key Length(Optional timestamp suffix Length)|
+--------+------------+---------------------------------------------+
|  Key   | Byte[kLen] | Key(Optional timestamp suffix)              |
+--------+------------+---------------------------------------------+
|  vLen  | Var32      | Value Length                                |
+--------+------------+---------------------------------------------+
|  Value | Bytes[vLen]| Value                                       |
+--------+------------+---------------------------------------------+
|        | KV...      | More KV                                     |
+--------+------------+---------------------------------------------+

| 3. Merge multiple WriteBatch(Current and others in WriteThread) to one
    WriteBatch;
| 4. Write to WAL record;
| 5. Fetch Key/Value(Slice, Slice),kType from Current WriteBatch
| 6. Write Current Key/Value/kType to WriteBatch for transaction(Optional);
| 7. Write Current Key/Value/kType(Combine to internal Key) to MemTable

The internal key in MemTable:

+--------+----------------+---------------------------------------------+
| Label  | Type           | note                                        |
+--------+----------------+---------------------------------------------+
|  kLen  | Var32          | internal key Length                         |
+--------+----------------+---------------------------------------------+
|  Key   | Byte[kLen-8]   | The key from last flow                      |
+--------+----------------+---------------------------------------------+
|  Packed| Fixed64        | [SeqNumber[56],kType[8]]                    |
+--------+----------------+---------------------------------------------+
|  vLen  | Var32          | The value Length                            |
+--------+----------------+---------------------------------------------+
|  Value | Byte[vLen]     | The value                                   |
+--------+----------------+---------------------------------------------+
