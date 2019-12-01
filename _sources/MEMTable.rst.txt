MemTable
==========

| There are some basic knowledge of *skiplist* structure which you need to learn
 before reading remaining.There is a nice blog about *skiplist*:

http://ticki.github.io/blog/skip-lists-done-right/

0. Implementation of SkipList
-------------------------------

0.0. SkipList
`````````````

The `Node` of *SkipList* is defined as below:

.. code-block:: cpp

    struct Node {
        Key const key;  //!< The key of K/V pair (user data)
        // the SkipList Column, next_[0] is the lowest list item
        std::atomic<Node*> next_[1];
    }

0.1. InlineSkipList
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

1. Implementation of MemTable
--------------------------------

1.0. HashSkipListRep
``````````````````````

| Each bucket of the HashMap contains pointer to one
 SkipList (InlineSkipList in fact).

1.1. HashLinkListRep
``````````````````````

| Each bucket of HashMap contains pointer to `LinkList` (for smaller count)
 or `SkipList` (for bigger count).

1.2. SkipListRep
```````````````````

`SkipListRep` store K/V by single `SkipList`.

1.3. VectorRep
``````````````````

`VectorRep` store K/V by single `vector`.
