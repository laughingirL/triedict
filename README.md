# triedict
A compressed and serializable python dictionary implemented as a Trie. Allows for lookup, prefix/predictive search, and Aho-Corasick string matching operations on the dictionary.

## About ##
A serializable Trie-based dictionary (`TrieDict`) for sequence-like objects
(strings, lists, tuples,...) that supports

1. lookup,
2. prefix search - also called predictive search, and 
3. Aho-Corasick string matching.

Given the average length of the sequences is a constant `m` the
runtime complexities are:
* Lookup: O(m)
* Prefix search: O(m)
* Matching: O(t); t = length of the text

## Alpha Version ##
Currently only `unicode` or `str` keys and `int` values
are supported. Support for arbitrary key sequences
and values will follow.

## Objectives ##
In the following, the sequence-like keys of the dictionary
are called a *patterns*, and the items of the sequences
are called *symbols*.

The objectives of the implementation can be summarized as:

1. `Triedict` is memory-efficient by by using native data types (ctypes) to store the Trie data.
2. `Triedict` allows for fast serialization to disc by using an array of structs and `uint32`-based pointers.
3. `Triedict` allows for high-performance by implementing the Trie logic as C routines (not done yet).

## Usage ##
Currently supported data types:
* Keys: Any sequence of `unicode` or `str` chars (e.g., strings or lists)
* Values: An `int` type within the range [0,2**32-2]
Example usage:
```
#from triedict import TrieDict
d = TrieDict()
d["key1"] = 0
d["key2"] = 1
d["key2"] = 11
print "key1" in d  # True
print "key2" in d  # True
print "key3" in d  # False
print d["key1"]    # 0
print d["key2"]    # 11
try:
    print d["key3"]    # exception
except ValueError, e:
    print e
print d.lookup("key1")  # 0
print d.lookup("key2")  # 11
print d.lookup("key3")  # None
print d.prefix_search("ke")  # [("y1",0), ("y2",11)]
# match only works if suffix pointers
# have been generated
d.generate_suffix_pointers()
print d.match("this is key1 and key2key1 in a string")
#    key   val pos
# [("key1", 0, 11), ("key2", 11, 20), ("key1", 0, 24)]
print d.match("this is key1 and key2key1 in a string", bound_chars=" .,;!?'\"()[]$=")
#    key, val, pos
# [("key1", 0, 11)]
```
## Next Version ##
In the next version the user can use arbitrary key and value types:
* The user can provide a encoder function `object -> int` and a
  decoder function `int -> object` to transform the object symbols to
  integers and vice versa. The integers need be be in the range [0,2**32-1].
  Encoding and decoding happens inside the dictionary.
* A inherited class `OTrieDict` allows to use any pickable object as values,
  on the cost of holding them in a list in memory (index by the internal int values).

## Internals ##
Design decisions:
* Small memory footprint - Usage of native data types (ctypes).
* Fast serialization - Trie nodes are kept in array to allow for fast binary
    serialization and de-serialization.
* Performance - Trie routines only use the native data types
    and the node-array. Allows for refactorization of the methods as
    c-routines (not yet done).
To allow for a fix-width node size (struct-size): 
* The parent-children relationship is modelled by a single child pointer and a brother pointer
 (p_child, p_brother). Example Trie containing 'bus' and 'bugs':
```
 * (n0) -> 0  (root node)
 |
 b (n1) -> 0
 |
 u (n2) -> 0
 |
 s (n3) -> g (n4) -> 0
           |
           s (n5) -> 0
```
Corresponding nodes:
```
node[0]: symb  0x00,  p_child -> 0,  p_brother -> 0
node[1]: symb 'b',    p_child -> 2,  p_brother -> 0
node[2]: symb 'u',    p_child -> 3,  p_brother -> 0
node[3]: symb 's',    p_child -> 0,  p_brother -> 4
node[4]: symb 'g',    p_child -> 5,  p_brother -> 0
node[5]: symb 's',    p_child -> 0,  p_brother -> 0
```
Suffix pointers:
* To support Aho-Corasick string matching the nodes have a pointer
  to the node in the Trie, that represents its longest suffix (p_suffix pointer).
Node storage:
* The nodes are saved as structs in an array and the pointers are
  modelled as `uint32` indexes on that array. This allows for fast
  serialization without resolving the pointers. For the pointers
  (p_child, p_brother, p_suffix) a value of 0 corresponds to a null pointer.
