---
title: A mostly complete guide to Go's copy function
type: blog
prev: blog/
---

## Introduction

I came across a use of the `copy` function that made my eyes glaze over. Initially I had no idea what I was looking at. Here's the relevant section of code. The goal is to add numbers to a sorted set.

```go
sorted := []int{}

for i := x; i < len(nums); i++ {
	n := nums[i-x]

	idx, found := slices.BinarySearch(sorted, n)

	if !found {
		sorted = append(sorted, 0)

        /*
        =========
        LOOK HERE
        =========
        */
		copy(sorted[idx+1:], sorted[idx:])

        sorted[idx] = n
	}

	// ...
}
```

I wanted to understand what was happening so I started from the beginning and developed this guide for myself. Hopefully you will find it useful as well.

I suggest running the examples in the main function in the [Go playground](https://goplay.tools/) if you want to follow along.

### Stop 🛑

In most cases if you want to make a copy of a slice use `slices.Clone` (Go v1.21 or higher).

```go
package main

import (
	"fmt"
	"slices"
)

func main() {
    numbers := []int{0, 42, -10, 8}

	clone := slices.Clone(numbers)

	// ...
}
```

## Serve me up a slice

For the uninitiated a "slice" in Go is, for all practical purposes, a dynamic array. That is to say, an array that can be resized. Go has a fixed size array as well. But slice is the go-to data structure for values held in a contiguous block of memory.

Now, as the name might suggest, the `copy` function copies values from a source slice to a destination slice.

The docs say:

> The copy built-in function copies elements from a source slice into a destination slice. As a special case, it also will copy bytes from a string to a slice of bytes. The source and destination may overlap. Copy returns the number of elements copied, which will be the minimum of len(src) and len(dst).

Let's look at some examples to better understand how `copy` works.

## Copy to an empty slice

```go
source := []int{1, 2, 3}

dest := []int{}

copy(dest, source)

fmt.Println("source", source, "dest", dest)
// source [1 2 3] dest []
```

Surprisingly, this does nothing 😅

Read more to find out why.

## Copy to a slice with length and capacity > 0

```go
source := []int{1, 2, 3}

dest := make([]int, 1)

fmt.Println(
	"dest len",
	len(dest),
	"dest cap",
	cap(dest),
)
// dest len 1 dest cap 1

copy(dest, source)

fmt.Println("source", source, "dest", dest)
// source [1 2 3] dest [1]
```

Now we have copied 1 number from the source to the destination.

Here is our first clue: length and capacity play some role in copy.

## Copy to an empty slice with capacity higher than the source

```go
source := []int{1, 2, 3}

dest := make([]int, 0, 10)

fmt.Println(
	"dest len",
	len(dest),
	"dest cap",
	cap(dest),
)
// dest len 0 dest cap 10

copy(dest, source)

fmt.Println("source", source, "dest", dest)
// source [1 2 3] dest []
```

Here we get another clue: length must be > 0 regardless of capacity.

## Copy to destination larger than source

```go
source := []int{1, 2, 3}

dest := make([]int, 10, 20)

fmt.Println(
	"dest len",
	len(dest),
	"dest cap",
	cap(dest),
)
// dest len 10 dest cap 20

copy(dest, source)

fmt.Println("source", source, "dest", dest)
// source [1 2 3] dest [1 2 3 0 0 0 0 0 0 0]
```

Here we can see that we have finally copied all the elements in the source slice.

So we have learned that to copy at least one value length must be > 0 and to copy all values then the length of dest must be >= the length of source.

## Copy the source to itself

```go
source := []int{1, 2, 3}

copy(source, source)

fmt.Println("source", source)
// source [1 2 3]
```

This is where things start to get a little nutty: we can copy the source slice to itself.

## Copy part of the source to itself

```go
source := []int{1, 2, 3}

dest := source[1:]

fmt.Println("source", source, "dest", dest)
// source [1 2 3] dest [2 3]

copy(dest, source[0:])

fmt.Println("source", source, "dest", dest)
// source [1 1 2] dest [1 2]
```

To understand this we need to understand what a slice really is.

A slice is a data structure with a length and capacity as we have seen. But, crucially, the slice also contains a pointer to a backing array.

It is the backing array that actually contains the values. In this case numbers.

The `[:]` syntax allows us to copy a slice without copying the backing array.

We can prove this by comparing what happens when we use `slices.Clone` to `[:]`. In this example we print the address in memory of the first element of three slices.

```go
nums := []int{42, 12, 22, 99}

clone := slices.Clone(nums)

fmt.Printf("address of first element in nums %p\n", &nums[0])
fmt.Printf("address of first element in clone %p\n", &clone[0])
// address of first element in nums 0x49743040000
// address of first element in clone 0x49743040020

notARealCopy := nums[:]

fmt.Printf("address of first element in notARealCopy %p\n", &notARealCopy[0])
// address of first element in nums 0x49743040000
```

Notice how `nums` and `notARealCopy` have the same address but `clone` does not.

Note that if you try this yourself you will likely get different addresses but the principle still holds.

That by itself is, arguably, not super useful. However, we can use `[:]` to reference a portion of the backing array.

For example `notARealCopy := nums[1:]` would create a copy of the slice where the first accessible element is actually index 1 of the backing array. In other words `notARealCopy[0] == nums[1]`.

```go
nums := []int{42, 12, 22, 99}

notARealCopy := nums[1:]

fmt.Println(notARealCopy[0] == nums[1])
// true
```

So using this syntax we can copy one portion of the source slice to itself.

Back to the original example.

```go
source := []int{1, 2, 3}

dest := source[1:]

fmt.Println("source", source, "dest", dest)
// source [1 2 3] dest [2 3]

copy(dest, source[0:])

fmt.Println("source", source, "dest", dest)
// source [1 1 2] dest [1 2]
```

In this case `dest = source[1:]` is everything after and including index 1 (e.g. `[2,3]`).

So `copy(dest, source[0:])` means copy everything after and including index 0 to the slice starting at index 1. Because `source[0]` is 1 and `source[1]` is 2 we end up replacing `[2,3]` with `[1,2]` hence the final result `[1,1,2]`.

## Grow the source and copy part if it to itself

```go
source := []int{1, 2, 3}

source = append(source, 0)
// source is now [1 2 3 0]

dest := source[1:]

fmt.Println("source", source, "dest", dest)
// source [1 2 3 0] dest [2 3 0]

copy(dest, source[0:])

fmt.Println("source", source, "dest", dest)
// source [1 1 2 3] dest [1 2 3]
```

Here we can see that by growing the source slice first with `append` we can copy all three original values.

## Put it all together to created a sorted set

```go
// We start with a sorted slice
source := []int{2, 3, 8, 42}

// We want to insert the new value 5
val := 5

// First we need to find where the value should go in the source slice
// idx is either the index of value in source if it exists already
// or it is the index where the value should go if it does not exist
idx, found := slices.BinarySearch(source, val)

// To ensure the "set" only has unique values we only insert the value if it does not already exist
if !found {
    // We grow the source to make room for the new value
    // The use of 0 is arbitrary
    source = append(source, 0)

    dest := source[idx+1:]

    // copy everything from the index where the new value should go to the next index
    copy(dest, source[idx:])

    // Put the new value where it belongs in the sorted set
    source[idx] = val
}

fmt.Println("source", source)
// source [2 3 5 8 42]
```

A sorted set is useful when you need to know the rank order of elements like in a leaderboard. It is also useful when solving [LeetCode 2817 "Minimum Absolute Difference Between Elements With Constraint"](https://leetcode.com/problems/minimum-absolute-difference-between-elements-with-constraint/) if you care about that sort of thing.

## Wrap up

We have learned how to use the built-in copy function to go beyond merely copying a slice.

We have also learned about slice internals along the way.

Hopefully this article illustrates how to break a topic up in a way that promotes deeper understanding.

Thank you for reading!
