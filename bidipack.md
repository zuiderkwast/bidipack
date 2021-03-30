# Bidipack

Note: This is work in progress. The encoding may change.

## General structure

The full bidipack (just as the ziplist and the listpack) consists of a header
and a sequence of elements.

    <header> <element>*

The element structure is more interesting, so it is described next. The header
is described later.

## Element encoding

Notation:

* intN = an integer which can be stored in N bits
* strN = a string whose size can be stored in N bits
* `xxxx` = integer value bits or string length bits
* `{Nbytes}` = N bytes representing an integer value or a string size
* `{string}` = the contents of a string
* L = the length of a string in bytes

The encoding is symmetrical in the sense that the first and the last byte share
the same pattern. String sizes are stored in big endian in the prefix and in
little endian in the suffix, so that the first and the last bytes are identical,
the second and the penultimate bytes are identical and so forth. Integer values
are stored in big endian.

| Type  | Size | Encoding                        | Range                     |
|-------|------|---------------------------------|---------------------------|
| uint7 | 1    | `0xxxxxxx`                      | 0..127                    |
| str6  | 2+L  | `10xxxxxx {string} 10xxxxxx`    | up to 63 bytes            |
| int16 | 3    | `1100xxxx xxxxxxxx 1100xxxx`    | -32768..32767             |
| int24 | 4    | `1101xxxx {2bytes} 1101xxxx`    | -8388608..8388607         |
| int32 | 5    | `1110xxxx {3bytes} 1110xxxx`    | -2147483648..2147483647   |
| str11 | 4+L  | `11110xxx xxxxxxxx {string} xxxxxxxx 11110xxx` | up to 2KiB |
| int48 | 8    | `11111000 {6bytes} 11111000`    | -140T..140T               |
| int64 | 10   | `11111001 {8bytes} 11111001`    | -9E..9E                   |
| str16 | 6    | `11111010 {2bytes} {string} {2bytes} 11111010` | up to 64KiB|
| str32 | 10+L | `11111011 {4bytes} {string} {4bytes} 11111011` | up to 4GiB |
| str0  | 1    | `11111100`                      | the empty string          |
|       |      | `11111101`                      | unused                    |
|       |      | `11111110`                      | unused                    |
| (end) | 1    | `11111111`                      | (reserved for end mark)   |

## Header

The full bidipack consists of a header and a sequence of elements. The header
has the following parts:

    <version> <flags> <capacity> <total-size> <num-elements>

* Version, a single byte with the fixed bits `10000001` (a palindrome).

* Flags, a single byte with the bits `0000aass`, where

    * `ss` is log<sub>2</sub> of number of bytes used for the total-size and
      num-elements fields, i.e. the `1 << ss` (1, 2, 4 or 8) bytes are used;
    * `aa` is the allocation strategy with the values 0 = compact (only
      allocate what's needed and shrink when deleting); 1 = normal (4
      reallocations sizes before the size is doubled); 2 = sparse (2
      reallocations before the size is doubled), 3 = extra sparse (each
      reallocation doubles the size). See capacity below.

* Capacity, a single byte representing size of the allocation.

  The allocation sizes are the size classes used by jemalloc and listed in
  [Table 1](http://jemalloc.net/jemalloc.3.html#size_classes) in jemalloc's
  manual. There are four sizes before the size is doubled. There is no point in
  using sizes in between these, since the underlying allocator would not be able
  to used that memory anyway.

  The single 'capacity' byte encodes these sizes using a formula. The values 1-4
  have the spacial meanings: `cap(1) = 8; cap(2) = 16; cap(3) = 32; cap(4) =
  48`. For values n > 4, the capacity is `cap(n) = cap2(n + 11)`, where `cap2(m)
  = (1 << (p + 2)) + (1 << p) * q`, where `p = (m & 0xfc) >> 2, q = m & 0x3`
  (i.e. q is the two least significant bits of m and p are the other 6 bits of
  m).

  The value 0 can be used to disable this mechanism. Then the allocation size is
  the same as the total size.

  When reallocation is needed, capacity is incremented by 1, and the formula
  gives the size in bytes. When deleting elements, reallocation is only
  performed when the capacity can be reduced by 2 steps, and then it is only
  reduced by 1 step. This is to ensure that reallocation isn't needed again if
  an element is added just after shrinking the allocation. An exception is when
  an explicit shrinking operation is performed.

* Total size, including the header itself, as an 8bit, 16bit, 24bit or 32bit
  unsigned integer, as indicated by the `ss` bits in the flags field, in big
  endian. When reallocating, care needs to be taken that the total size field is
  given enough bytes to represent all possible sizes, up to the size of the
  allocation.

* Num elements, encoded in the same way as the total size.

## Usage and implementation ideas

The possible allocation sizes are chosen to match the sizes of allocators.
Therefore, it is suggested that malloc() is used rather than realloc(), since
the latter would have to copy memory to a new area, which would later need to be
memmove'd again(). Instead, when inserting a new element which would exceed the
capacity, perform the following steps:

1. Allocate a new piece of memory.
2. Copy the parts before and after the new element to the right place in the new
   memory space using memcpy().
3. Encode the new element into the right place in the new memory space.
4. Deallocate the old memory space.

## Variations

### Encoding alternatives

* To favour relatively small integers, str6 can be replaced by str5 and int10 to
  allow integers in the range -512..511 to be encoded using 2 bytes as `100xxxxx
  100xxxxx` (instead of 3 bytes, a saving of 33%) and strings up to 31 bytes as
  `101xxxxx {string} 101xxxxx`. The drawback is that strings of 32 bytes has 4
  bytes overhead instead of 2 (6% increase), with the relative cost being lower
  for longer strings.
* To optimize for relatively small integers, replace str6, int16 and int24 with
  int12 and int18, encoded as `10xxxxxx 10xxxxxx` and `110xxxxx xxxxxxxx
  110xxxxx`. By also removing str11, int48 can be replaced by an int54 encoding.
* To favour strings in the range 2KiB to 4KiB, int24 can be relaced by replaced
  by str12 as `1001xxxx xxxxxxxx {string} 1001xxxx xxxxxxxx`. Then, str11 can
  also be replaced, see next bullet.
* str11 can be replaced by str19 to save 4 bytes for strings of sizes 8KiB to
  524KiB, or by int36 to save two bytes (28%) for integers by of magnitude from
  67M to 281T, or by a str10 and a str18 encoding.
* Many more variants are possible, with increasing complexity for more encoding
  combinations. Any encoding can be removed, trading memory for simplicity.

### Deleted sections

A possible extension is to allow the sequence to contain empty/unused/deleted
sections, represented as a special repeated byte value or by encoding a longer
empty sequence as a tag and a length in both ends.

| Type   | Encoding (idea)          | Comment      |
|--------|--------------------------|--------------|
| del0   | 11111101                 | Unused byte  |
| del32  | 11111110 {4bytes} {undef} {4bytes} 1111110 | A section which can be skipped using its length |

Such sections could easily be skipped when interating over the elements. To keep
track of the amount of unused space, an extra field may need be added to the
header. A flag bit can be used to indicate whether deleted sections may be
present and that an extra header field representing the total size of unused
space is present.

## Author

Viktor Söderqvist
