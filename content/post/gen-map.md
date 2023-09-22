+++
date = "2023-09-22"
title = "Generations-based array"
tags = [
    "[go]",
    "[performance]",
    "[shortread]",
]
description = "A faster sparse-dense array, but without iteration."
draft = false
+++

## Intro

I was fascinated by the sparse map/set described in Russ Cox article [research.swtch.com/sparse](https://research.swtch.com/sparse).

And I'm not the only one: this exact implementation is used in Go source code more than once! The compiler uses it for many ID-like maps and sets; regexp package uses it for a [queue](https://github.com/golang/go/blob/795414d1c628f763defa43199ab51ea3dc3241d8/src/regexp/exec.go#L17).

But there is one thing that is still bugging me: it's hard to make it very efficient. All operations I care about are O(1), just like with normal array, but `get` and `set` operations clearly become slower.

In fact, if your arrays are not that big (less than 0xffff bytes?), you might be better off with an array with O(n) clear operation that will be quite efficient thanks to the memclrNoHeapPointers. If you do many `get`+`set` operations, the increased constant overhead may be too much.

In this article, I'll propose a different data structure that can replace sparse-dense map (and set) if you don't need the iteration over the elements.

## The Problem

Let me start with a problem that I'm trying to solve.

Imagine that you need a mapping structure that you can re-use. Something like a `map[uint16]T`, but with a more predictable allocations pattern.

Your function may look like this:

```go
func doWork(s *state) result {
    s.map.Reset() // You want the memory to be re-used

    // Do the work using the map.
    // Only get+set operations are used here.
}
```

If your "map" can re-use the memory properly, this code may become zero-alloc.

Our requirements can be described as follows:

| Operation | Complexity |
|---|---|
| Set | O(1) |
| Get | O(1) |
| Reset | O(1) |

We want it all, plus the efficient memory re-use.

We'll analyze these choices today:

* `map[uint16]T`
* `[]T`
* `sparseMap`
* `generationsMap`

The slice and map solutions do not fit our requirements, but we'll use them for a comparison.

## Benchmark Results

Let's start by comparing the raw performance.

| Data Structure | Set | Get | Reset |
|-------------|------:|------:|-----:|
| map         | (x16.8) 27988 | (x40.7) 15631 | 1002 |
| slice       | 1665  | 686   | 4144 |
| sparse      | (x6.8) 11347 | (x1.7) 1193  | 11   |
| generations | (x1.2) 1978  | (x1.1) 781   | 16   |

Observations:

* Map is heavily outclassed
* Both sparse and generation maps have a crazy-fast reset
* Even with 5000 elements (8*5000=40000 bytes), a slice reset takes noticeable time
* `sparse.set()` operation is ~7 times slower than slice!
* `sparse.get()` operation is ~2 times slower than slice
* Generations map is almost as fast as a slice, but reset is much faster

We'll talk about the generations map implementation later, don't you worry.

You can find the exact benchmarks code [here]((https://gist.github.com/quasilyte/a64bd66093c20c5e146b60e2cf3f3191)).

Some benchmark notes:

* I used a [real-world](https://github.com/golang/go/blob/795414d1c628f763defa43199ab51ea3dc3241d8/src/cmd/compile/internal/ssa/sparsemap.go) sparse-dense implementation
* Every `get`/`set` goes through a noinline wrapper to avoid the unwanted optimizations
* Every `get`/`set` test runs the operation 5000 times
* Every benchmark is using 5000 elements (it's important for slice reset)
* The measurements above are divided by 10 for an easier interpretation
* The value type used is `int` (8 bytes on my x86-64 machine)

Now, you should be cautious about random benchmarks posted on the internet. But no matter how you write and/or run these, generations map will always be faster than a sparse-dense map (or set). It's almost as fast as a slice solution while still having a very fast `O(1)` reset.

There are reasons for it to be faster. Let's talk about them.

## Sparse Map Issues

Why `sparse.set()` operation is so slow?

When it comes to insertion of a new value, the sparse map has to do two memory writes. One for the `sparse` and one for the `dense`. Updating the existing value only writes to `dense`.

```go
func (s *sparseMap[T]) Set(k int32, v T) {
	i := s.sparse[k]
	if i < int32(len(s.dense)) && s.dense[i].key == k {
		s.dense[i].val = v
		return
	}
	s.dense = append(s.dense, sparseEntry[T]{k, v})
	s.sparse[k] = int32(len(s.dense)) - 1
}
```

Another issue is that two slices mean twice as much bound checks that can occur. And while you can be careful and use uint keys and check for the bounds yourself to stop compiler from generating an implicit boundcheck, you'll still pay for these if statements.

The `sparse.get()` operation also suffers from a double memory read.

## Generations Map

It's possible to use some of the ideas behind the sparse-dense map and create an even more specialized data structure.

```go
type generationsElem[T any] struct {
	seq uint32
	val T
}

type generationsMap[T any] struct {
	elems []generationsElem[T]
	seq   uint32
}

func newGenerationsMap[T any](n int) *generationsMap[T] {
	return &generationsMap[T]{
		elems: make([]generationsElem[T], n),
		seq:   1,
	}
}
```

Every element will have a generation counter (seq). The container itself will have its own counter. The container counter starts with 1, while elements start with 0.

Both `get` and `set` operations look very similar to the slice version, but with a `seq` check.

```go
func (m *generationsMap[T]) Set(k uint, v T) {
	if k < uint(len(m.elems)) {
		m.elems[k] = generationsElem[T]{val: v, seq: m.seq}
	}
}

func (m *generationsMap[T]) Get(k uint) T {
	if k < uint(len(m.elems)) {
		el := m.elems[k]
		if el.seq == m.seq {
			return el.val
		}
	}
	var zero T
	return zero
}
```

If `seq` of the element is identical to the container's counter, then this element is defined. Otherwise, it doesn't matter what are the contents of this element.

Setting the element means updating the element's counter to the container's counter.

You probably can already guess how `Reset` will look like.

```go
func (m *generationsMap[T]) Reset() {
	m.seq++
}
```

Well, this is good enough for the most use cases, but there is a small chance that our `uint32` will overflow, making some undefined elements defined. Increasing the `seq` size to `uint64` could help, but it will increase the per-element size overhead. Instead, we can do a real clear operation one in `MaxUint32` resets.

```go
func (m *generationsMap[T]) Reset() {
	if m.seq == math.MaxUint32 {
		m.seq = 1
		clear(m.elems)
	} else {
		m.seq++
	}
}
```

It's definitely possible to use `uint8` or `uint16` for the `seq` field. That would mean less per-element size overhead at the price of a more frequent data clear.

* The generations map does exactly 1 memory read and write
* It's easier to get rid of all implicit bound checks
* Its memory consumption is comparable to the sparse-dense array
* The `Reset` complexity is constant (amortized)
* Arguably, it's even easier to implement and understand

It's possible to make a generations-based set too. The `get` operation can be turned into `contains` with ease. With sets, only the counters are needed.

```go
func (m *generationsSet) Get(k uint) bool {
	if k < uint(len(m.counters)) {
		return m.counters[k] == m.seq
	}
	return false
}
```

It may sound fascinating, right? Well, you can't use this data structure as a drop-in replacement for a sparse-dense. For instance, a generations-based map can't be iterated efficiently.

You can add a length counter if you really need it, but that will add some extra overhead to the `set` operation. I would advise you not to do so. The main reason this structure can be so fast is its simplicity.

## Conclusion

I used this structure in my [pathfinding](https://github.com/quasilyte/pathing/) library for Go. The results were great: 5-8% speedup just for a simple data structure check. Keep in mind that this library is already heavily optimized, so every coupld of percentages count.

In turn, this pathfinding library was used in a game I released on Steam: [Roboden](https://store.steampowered.com/app/2416030/Roboden/).

Therefore, I would consider this data structure idea to be production-ready.