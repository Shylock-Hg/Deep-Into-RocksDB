MEMTable skiplist
===================

| There are some basic knowledge of *skiplist* structure which you need to learn
 before reading remaining.There is a nice blog about *skiplist*:

http://ticki.github.io/blog/skip-lists-done-right/

Implementation of RocksDB
---------------------------

0. SkipList
`````````````

The `Node` of *SkipList* is defined as below:

.. code-block:: cpp

    struct Node {
        Key const key;  //!< The key of K/V pair (user data)
        std::atomic<Node*> next_[1];  //!< the SkipList Column, next_[0] is the lowest list item
    }

1. InlineSkipList
`````````````````````

The `Node` of *InlineSkipList* is defined as below:

.. code-block:: cpp

    struct Node {
        // the next_[0] is the lowest list item, next_[-1] is higher one
        // the next_+1 point to the value of key
        std::atomic<Node*> next_[0];
    }

| The `Node` of *InlineSkipList* are defined not very property, so you must do
 `reinterpret_cast` to access value of *key*.But there are two optimization
 while the *Key* are pointer in *SkipList*:

1. Less memory consumption for one Key pointer.
2. Less cache missing for better locality memory.
