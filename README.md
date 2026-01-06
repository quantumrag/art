# art
A compact, in-memory **Adaptive Radix Tree (ART)** implementation in Go.

This repo is a small ART/trie intended for fast lookups with efficient inserts/deletes, while keeping keys in sorted order (useful for iteration and prefix scans).

## Features
- Insert / search / delete
- Pre-order traversal via callback
- Iterator API
- Prefix scan (`Scan`) for keys under a prefix
- Adaptive node sizes (Node4/Node16/Node48/Node256)

## Install
```bash
go get github.com/arriqaaq/art
```

## Usage
```go
package main

import (
	"fmt"

	"github.com/arriqaaq/art"
)

func main() {
	tree := art.NewTree()

	// Insert
	tree.Insert([]byte("hello"), "world")
	fmt.Println("value=", tree.Search([]byte("hello")))

	// Delete
	tree.Insert([]byte("foo"), "bar")
	fmt.Println("deleted=", tree.Delete([]byte("foo")))

	// Traverse (callback)
	tree.Each(func(n *art.Node) {
		if n.IsLeaf() {
			fmt.Println("key=", string(n.Key()), "value=", n.Value())
		}
	})

	// Iterator
	for it := tree.Iterator(); it.HasNext(); {
		n := it.Next()
		if n.IsLeaf() {
			fmt.Println("key=", string(n.Key()), "value=", n.Value())
		}
	}

	// Prefix Scan
	tree.Insert([]byte("api"), "bar")
	tree.Insert([]byte("api.com"), "bar")
	tree.Insert([]byte("api.com.xyz"), "bar")

	tree.Scan([]byte("api"), func(n *art.Node) {
		if n.IsLeaf() {
			fmt.Println("match=", string(n.Key()))
		}
	})
}
```

## API notes
- `tree.Insert(key, value) bool` returns `true` if an existing key was updated, `false` if a new key was inserted.
- `tree.Search(key) interface{}` returns the stored value or `nil` if not found.
- `tree.Delete(key) bool` returns `true` if a key was removed.
- Leaf nodes expose:
  - `n.Key() []byte` (the key for leaf nodes)
  - `n.Value() interface{}` (the value for leaf nodes)

### Key limitation
This implementation internally appends a `0x00` terminator to keys that do not already contain one.

Practically: **treat keys as byte strings that must not contain `0x00` bytes**.

## Tests
```bash
go test ./...
```

## Benchmarks
Benchmarks use the fixtures in `test/words.txt` and `test/uuid.txt`.

Run:
```bash
go test -bench . -benchmem
```

Example output (will vary by machine):
```text
// ART tree
BenchmarkWordsArtTreeInsert
BenchmarkWordsArtTreeSearch
BenchmarkUUIDsArtTreeInsert
BenchmarkUUIDsArtTreeSearch
```

## References
- [The Adaptive Radix Tree: ARTful Indexing for Main-Memory Databases (paper)](http://www-db.in.tum.de/~leis/papers/ART.pdf)
- [Kelly Dunn's Go ART implementation](https://github.com/kellydunn/go-art)
- [Beating hash tables with trees? The ART-ful radix trie](https://www.the-paper-trail.org/post/art-paper-notes/)
- [Pavel Larkin's Go ART implementation](https://github.com/plar/go-adaptive-radix-tree)

## License
MIT (see `LICENSE`).
