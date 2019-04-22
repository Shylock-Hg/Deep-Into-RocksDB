Cache in RocksDB
==================

The cache is used to store the lastest used data in memory for fast searching.

0. LRU Cache
--------------

The LRU Cache in RocksDB implement O(1) Put/Get.The basic mechanism is puting
by double-linked list and getting by hash map.

The basic data structure as below:

::

      |  LRUHandle* |                        prev
      |  LRUHandle* | ---> |  LRUHandle  | <-------------------------
      |  LRUHandle* |          |next ^     ----------------------    |
      |  LRUHandle* |          V     |prev    next              |    |
      |  LRUHandle* |   -> |  LRUHandle  |                      V    |
      |  LRUHandle* |   |                              ---->| LRUHandle |
      |  LRUHandle* | ---              prev            |       |
      |  LRUHandle* |             ---------------------|       |next
      |  LRUHandle* |             |  ---------------------------
      |  LRUHandle* | ---|        |  |  ^
                         |        |  V  |next  next_hash(for collision)
                         --> |  LRUHandle  | ----------> |  LRUHandle  | ----->


So the RocksDB put(and evict) item by double-linked list and hash table, get item by hash
table.
