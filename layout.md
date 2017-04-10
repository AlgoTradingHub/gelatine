```
In block-encoded data, when index is xx%8, value must be tagged. Otherwise data is referenced by an opaque ptr with size.

Inline4: {[tag] block} {} = 8 bytes
	Boolean, Byte, Int16, Int32, Int48, Enum, String[6], Byte[6], Composite4
Block: {tag} {block}[1,7] upto 56 byte contiguous data; BlockRef 1 bit (object and object ref have same header, pointer is never inline); therefore blockrefs may form single-linked lists.
Blocks may point only to blockstarts; 2^6*2^32=2^38 bytes = 256 gb heap, ok 2^6*2^24 = 2^30 = 1G
Handle: {[tag] [ShapeID] [PointerID]}
Large ptrs are block types. Indirectors are block types. Type Params are encoded in first block
Block size encoded in type? 7-wide arraylets? so arraylet op is inlined

Assume 2 bits for gc meta and/or fwdptrs

2M 21b packet [http], 256M 28b slab [bz]; guid?
Inline, block7 (64b), do we need medium-size arrays? we might have dicts/hashtables though. next logical unit is page (4096/8=512blocks, tagged bytemap wont fit!) // 2^21=2,097,152

Source: Implied, Inline, TypeParam/Len Card, Value Card, Ref Card; singletons may include "just skip", opcodes, etc
Refbit, or even 2 bits
Types: Double, Singleton(2^32-1), Inline4(2^16-1), InlineInt48, InlineASCIIZ6,
Pointer? TypedHandle? Bucketed Symbol? Ref Mutability? // 512/64=8, sha512 will not fit, so it's 128bit security which is OK
Typed Pointer to Pooled Object Table / 256 tables of 1G objects…
2^40=1E12
What we store inline? Type, Typeparam; GC info for refs; pointer/data info for non-inline pointers
need to enable pointer following [without shapes?]

long = token, 8 longs = block, 64 blocks = page
mark bitmap is 1 bit per block, hence 1 long per page, or per 512 longs.
```

```
| Name             | T      | L   | V     | R    | 1        2         3        4        5        6        7        8        | next       | desc                                        |
|------------------|--------|-----|-------|------|:------------------------------------------------------------------------ |------------|---------------------------------------------|
| Double           | 1      | 0   | 2^62  | 0    | seeeeeee eeee ammm dddddddd dddddddd dddddddd dddddddd dddddddd dddddddd |            | regular IEEE754 with SNaN=OFF               |
| DoubleInfinity   | 1      | 0   | 2     | 0    | s1111111 1111 0000 00000000 00000000 00000000 00000000 00000000 00000000 |            | all 52 bit zero                             |
| Block Meta       | 2^31-1 | 0/k | 1     | 0    | 11111111 1111 0000 00000000 00000000 0ppppppp pppppppp pppppppp pppppppp |            | reserved for impl-specific tags             |
| Singleton/Symbol | 2^31   | 0/k | 1     | 0    | 11111111 1111 0000 00000000 00000000 1ppppppp pppppppp pppppppp pppppppp |            | at least 1 bit nonzero; fit inline16?       |
| Reserved32       | 1      | 0/k | 2^32  | 0    | 11111111 1111 0000 00000000 00000001 pppppppp|pppppppp|pppppppp|pppppppp |            |                                             |
| Int32            | 1      | 0/k | 2^32  | 0    | 11111111 1111 0000 00000000 00000010 pppppppp|pppppppp|pppppppp|pppppppp |            |                                             |
| UInt32           | 1      | 0/k | 2^32  | 0    | 11111111 1111 0000 00000000 00000011 pppppppp|pppppppp|pppppppp|pppppppp |            |                                             |
| Inline32         | 2^16-4 | 0/k | 2^32  | 0    | 11111111 1111 0000 ssssssss ssssssss pppppppp pppppppp pppppppp pppppppp |            | s >= 00000000 00000100                      |
| InlineASCIIZ6    | 1      | i   |       | 0    | 11111111 1111 0001 Mccccccc Uccccccc cccccccc cccccccc cccccccc cccccccc |            | 0-prefixed, M=continue, U=utf8-overlapped5  |
| Int48            | 1      | 0   | 2^48  | 0    | 11111111 1111 0010 dddddddd dddddddd dddddddd|dddddddd|dddddddd|dddddddd |            | Signed int.                                 |
| UInt48           | -      |     |       |      | 11111111 1111 0011 dddddddd dddddddd pppppppp|pppppppp|pppppppp|pppppppp |            | Uint; decrementing 0 makes a signed int     |
| Inline64         | 2^16   |     | 2^64  | 2^32 | 11111111 1111 0100 ssssssss ssssssss pppppppp|pppppppp|pppppppp|pppppppp | 1xdata     | 64+64 bit tagged value, p0 = continue       |
| InlineRef64      | 2^16   |     | 2^64  | 2^32 | 11111111 1111 0101 ssssssss ssssssss pppppppp|pppppppp|pppppppp|pppppppp | 64bit ref  | 64+64 bit tagged ptr, p0 = continue         |
| Inline448        | 2^16   |     | 2^448 | 2^32 | 11111111 1111 0110 ssssssss ssssssss pppppppp|pppppppp|pppppppp|pppppppp | 7xdata/ref | 448 bit tagged value, struct                |
| BlockChain       | 2^16   |     | long  | 2^32 | 11111111 1111 0111 ssssssss ssssssss pppppppp|pppppppp|pppppppp|pppppppp |            | block linked list tagged value, string      |
| DoubleNaN        | 1      |     | 1     | 2^32 | 11111111 1111 1000 00000000 00000000 00000000|00000000|00000000|00000000 |            | ForeignRef with Type=0 is NaN               |
| ForeignRef24     | 2^24-1 |     | fixed | 2^24 | 11111111 1111 10gg hhhhhhhh hhhhhhhh hhhhhhhh|pppppppp|pppppppp|pppppppp |            | 1 table per type, 2 GC bits, nonzero typeid |
| ForeignRef40     | 256    |     | fixed | 2^40 | 11111111 1111 11gg hhhhhhhh pppppppp pppppppp|pppppppp|pppppppp|pppppppp |            | maybe another split makes sense             |
```

```
| Shape bits        | data    |
|-------------------|---------|
| 00000000 00000100 | float32 |
```
