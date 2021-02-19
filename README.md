# Bidipack

Like the ziplist and listpack data structures in [Redis](https://redis.io/), a
bidipack encodes a sequence of strings and integers in a single chunk of
continuous memory. This datastructure is only to be used for limited sizes, to
save memory and utilize CPU cache. The main features are:

* Compact.
* Traversible in both directions.

For a background, see below. See also the specification in
[bidipack.md](bidipack.md) and a comparison with ziplist and listpack in
[comparison.md](comparison.md).

## Status

* [Specification](bidipack.md): WIP
* Implementation: TODO
* Tests: TODO

## Background

Optimizing for size is important when data contains many small elements. A short
string like "user" is 4 bytes, but a pointer would require 8 bytes. Therefore,
it becomes important to avoid pointers and instead pack data together in
continuous chunks.

**Ziplist**, the earlier Redis datastructure, has the following structure:

    <total-byte-size> <num-elements> <element>* <end>

The interesting part is how each `<element>` is encoded. It has the following structure:

    <prevlen> <tag-and-length> <content>

A few small values (the integers 0..12) are represented entirely in the
tag-and-size field as a single byte. Short strings are using some bits of the
tag to store its length, while the characters are stored in `<content>`. Larger
integers have one tag byte and a fixed number of bytes for the value. Longer
strings have one tag byte and one or more size bytes, followed by the string
contents.

Prevlen is the total length of the previous element. This is either 1 or 5
bytes. Inserting and deleting elements in the middle of a ziplist not only
requires moving memory to make space, but also requires updating the prevlen of
surrounding elements. This adds to its complexity.

**Listpack** solves this drawback by encoding the length of each element in the end
of the element itself, in a way which makes it possible to read it backwards
byte-by-byte.

    <tag-and-length> <content> <length-suffix>

The total length (excluding the length-suffix) of the element is encoded in the
suffix in a variable number of bytes, in a way which can be read backwards. A
byte on the form 0xxxxxxx means that there are no more bytes to the left and
1xxxxxxx means that there are more bytes to the left to read. Thus, the suffix
contains 1-5 bytes. The <tag-and-length> is similar as for ziplist. For example,
an integer 0..127 is stored as 0xxxxxxx and the suffix is 00000001 (i.e. two
bytes in total).

A more thorough explanation of the ziplist and the listpack can be found here:
https://github.com/antirez/listpack/blob/master/listpack.md

## Observation

In a listpack, all elements require at least one byte for tag-and-length and one
byte for length-suffix. By allowing tag-and-lengh to overlap with the suffix,
only a single byte 0xxxxxxx is enough to represent an integer in the range
0..127, but then no other suffix can start with a 0 bit. Next, we can represent
a 12bit integer as 10xxxxxx 10xxxxxx, i.e. we let the leading bits 10 represent
that it's a 2-byte element and where the x bits are used to encode a 12bit
integer, and so on. (This is an example. It is not necessarily the exact
encoding used in a bidipack.)

## Author and acknowledgements

Bidipack and these documents were written by Viktor Söderqvist. It was highly
incluenced by the ziplist and listpack encodings by Salvatore Sanfilippo, Pieter
Noordhuis, Oran Agra, Yuval Inbar and others.