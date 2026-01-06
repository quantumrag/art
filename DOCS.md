# art — Full Documentation

A compact, in-memory **Adaptive Radix Tree (ART)** implementation in Go.

This document is the “full” markdown artifact for the repo: overview, design notes, API reference, and usage.

## Contents
- Overview
- Design notes
- Public API reference
- Key semantics and limitations
- Complexity and performance notes
- Usage examples
- Testing
- Benchmarks
- FAQ
- References
- License

## Overview
An Adaptive Radix Tree (ART) is a radix trie that adapts its internal node representation based on fan-out, aiming to be both fast and space-efficient.

This implementation provides:
- Insert / search / delete
- Full-tree traversal (`Each`) using a callback
- An iterator (`Iterator`) that yields *all* nodes (internal + leaf)
- Prefix scan (`Scan`) to traverse the subtree under a prefix

## Design notes
### Node types
Internal nodes change representation as the number of children grows:
- `Node4`
- `Node16`
- `Node48`
- `Node256`

At a high level:
- Smaller nodes keep compact arrays of keys/children.
- Larger nodes trade memory for faster indexing.

### Prefix compression
Internal nodes carry a “compressed prefix” (`prefix`, `prefixLen`) to skip over common key bytes.

`MaxPrefixLen` (currently `10`) bounds how much of the prefix is stored inline. When prefixes exceed this bound, parts of the comparison fall back to inspecting the minimum leaf under a subtree.

### Ordering
Children are traversed in ascending byte order. Practically:
- If you filter traversal to **only leaf nodes**, the leaves are visited in lexicographic order by byte sequence.
- Traversals are *pre-order* with respect to internal nodes (the internal node is visited before its children).

## Public API reference
Package: `github.com/arriqaaq/art`

### Types
#### `type Tree struct`
An ART instance.

#### `type Node struct`
A node in the ART.
- Internal nodes route on key bytes.
- Leaf nodes store an inserted key/value.

#### `type Iterator interface`
```go
type Iterator interface {
	HasNext() bool
	Next() *Node
}
```
The iterator yields nodes in the same overall traversal order used by the tree’s internal iterator implementation.

#### `type Callback func(node *Node)`
Used by traversal methods.

### Constants
Node “kinds” (used by `(*Node).Type()`):
- `Node4`, `Node16`, `Node48`, `Node256`, `Leaf`

Other relevant constants:
- `MaxPrefixLen = 10`

### Functions
#### `func NewTree() *Tree`
Create an empty tree.

### `Tree` methods
#### `func (t *Tree) Size() uint64`
Returns number of keys stored in the tree.

#### `func (t *Tree) Insert(key []byte, value interface{}) bool`
Insert or update a key.
- Returns `true` if an existing key was updated.
- Returns `false` if a new key was inserted.

#### `func (t *Tree) Search(key []byte) interface{}`
Look up a key.
- Returns the stored value, or `nil` if not found.

#### `func (t *Tree) Delete(key []byte) bool`
Delete a key.
- Returns `true` if the key existed and was removed.

#### `func (t *Tree) Each(callback Callback)`
Traverses the entire tree, calling `callback` on **every node** (internal and leaf) in pre-order.

#### `func (t *Tree) Scan(prefix []byte, callback Callback)`
Prefix scan. Traverses the subtree under `prefix` and calls `callback` on nodes visited.

In typical usage, you provide a callback that filters leaf nodes:
```go
leafOnly := func(n *art.Node) {
	if n.IsLeaf() {
		// n.Key(), n.Value()
	}
}
```

#### `func (t *Tree) Iterator() Iterator`
Returns an iterator over **all nodes**.

### `Node` methods
#### `func (n *Node) IsLeaf() bool`
Returns `true` if this node is a leaf.

#### `func (n *Node) Type() int`
Returns one of: `Node4`, `Node16`, `Node48`, `Node256`, `Leaf`.

#### `func (n *Node) Key() []byte`
For leaf nodes only.
- Returns the stored key **without** the internal terminator byte.
- Returns `nil` for internal nodes.

#### `func (n *Node) Value() interface{}`
For leaf nodes only.
- Returns the stored value.
- Returns `nil` for internal nodes.

## Key semantics and limitations
### Terminator byte behavior (`0x00`)
This implementation appends a `0x00` terminator to keys internally (when the key does not already contain one).

Implications:
- Keys are effectively treated as byte strings terminated by `0x00`.
- Keys that contain an embedded `0x00` byte are **not supported** (they can be truncated at the first `0x00`).

If you need arbitrary binary keys (including zero bytes), this implementation will need changes (e.g., storing explicit key lengths and avoiding sentinel terminators).

### Mutability / aliasing
Insert copies the key bytes for leaf nodes, so reusing or mutating the input slice after insertion should not affect the stored key.

### Concurrency
The tree is not documented or implemented as thread-safe. Use external synchronization if accessed concurrently.

## Complexity and performance notes
Let `k` be the key length in bytes.
- Search: typically **O(k)**
- Insert: typically **O(k)** (may include node growth and prefix adjustments)
- Delete: typically **O(k)** (may include node shrink/collapse)

Performance and memory behavior depend heavily on key distribution (shared prefixes, branching factor, etc.).

## Usage examples
### Basic CRUD
```go
package main

import (
	"fmt"
	"github.com/arriqaaq/art"
)

func main() {
	t := art.NewTree()

	t.Insert([]byte("hello"), "world")
	fmt.Println(t.Search([]byte("hello"))) // world

	t.Insert([]byte("hello"), "WORLD")
	fmt.Println(t.Search([]byte("hello"))) // WORLD

	fmt.Println(t.Delete([]byte("hello"))) // true
	fmt.Println(t.Search([]byte("hello"))) // <nil>
}
```

### Traverse leaves (sorted by key)
```go
leafOnly := func(n *art.Node) {
	if n.IsLeaf() {
		fmt.Printf("%s => %v\n", string(n.Key()), n.Value())
	}
}

t.Each(leafOnly)
```

### Iterator (yields internal + leaf nodes)
```go
for it := t.Iterator(); it.HasNext(); {
	n := it.Next()
	if n.IsLeaf() {
		fmt.Printf("%s => %v\n", string(n.Key()), n.Value())
	}
}
```

### Prefix scan
```go
t.Insert([]byte("api"), 1)
	t.Insert([]byte("api.foo"), 2)
	t.Insert([]byte("api.bar"), 3)
	t.Insert([]byte("abc"), 4)

// Visit keys under the "api" prefix.
t.Scan([]byte("api"), func(n *art.Node) {
	if n.IsLeaf() {
		fmt.Println(string(n.Key()))
	}
})
```

## Testing
Run unit tests:
```bash
go test ./...
```

## Benchmarks
This repo includes benchmark fixtures:
- `test/words.txt`
- `test/uuid.txt`

Run benchmarks:
```bash
go test -bench . -benchmem
```

## FAQ
### Does traversal return keys in sorted order?
If you filter callbacks to leaf nodes, children are visited in ascending byte order, so leaf keys are visited in lexicographic (byte-wise) order.

### Does the iterator return only leaves?
No. The iterator yields both internal and leaf nodes. Filter with `IsLeaf()` if you only want stored records.

### Can I store `[]byte` values?
Yes—values are stored as `interface{}`. If you store `[]byte`, you own the semantics (copy it yourself if you don’t want callers to mutate it).

## References
- [The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases (paper)](http://www-db.in.tum.de/~leis/papers/ART.pdf)
- [Kelly Dunn's Go ART implementation](https://github.com/kellydunn/go-art)
- [Beating hash tables with trees? The ART-ful radix trie](https://www.the-paper-trail.org/post/art-paper-notes/)
- [Pavel Larkin's Go ART implementation](https://github.com/plar/go-adaptive-radix-tree)

## License
MIT (see `LICENSE`).
