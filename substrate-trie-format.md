
# Substrate trie binary format

Tries in substrate are encoded as follows:

4 node types:
- Empty
- Extension
- Leaf
- Branch

The first byte of the sequence of bytes describing a node is encoded with a modified version of the ethereum Hex Prefix Notation, using the two highest bits to encode whether the node is one of Empty, Branch or Extension/Leaf node. These bits are not used in ethereum HPN. The lower two bits of the high nibble work exactly the same as in ethereum HPN: the lowest encodes oddness, the second lowest whether it's a Leaf or Extension node (sometimes referred to as the "termination" flag). The remaining two high bits of the high nibble are `00` for an Empty node, `01` for a Branch node and `10` for a Leaf or Extension node. To distinguish a Leaf node from an Extension node the second lowest bit is used, exactly like in ethereum HPN.


| First byte    | Node type      | Comment |
|---------------|----------------|----------|
| `0b0000_0000` | Empty node     |
| `0b0100_0000` | Branch node    | 
| `0b1010_0000` | Leaf node      | Even number of nibbles follow |
| `0b1011_xxxx` | Leaf node      | Odd number of nibbles. `xxxx` is the first nibble of the payload, and with the next 7 nibbles, encodes the length (l) in bytes of the key part of the payload (the value follows) |
| `0b1000_0000` | Extension node | Even number of nibbles follow |
| `0b1001_xxxx` | Extension node | Odd number of nibbles follow; `xxxx` is the first nibble of the payload |

The payloads are encoded using [`parity codec`](https://github.com/paritytech/parity-codec). A sequence of bytes is encoded as itself and prepended with the payload length as 4 bytes forming an unsigned 32-bit int. Booleans are encoded as one byte with values `0` or `1`; vectors are prepended with an unsigned 32-bit int indicating the number of items in the vector.

## Examples
Note that the input items are sorted and deduplicated before the trie processing proper starts.

### No input

Input: `[]`
Output: `0x00`

### Single tuple, 1 byte key, 1 byte value

Input: `([0xaa], [0xbb])`
Annotated output:

```
trie: [
    0xa0,	// 0b1010_0000 => Leaf node with even length payload
    0x1,	// Key length: 1 byte
    0x0,
    0x0,
    0x0,
    0xaa,	// Key
    0x1,	// Value length: 1 byte	
    0x0,
    0x0,
    0x0,
    0xbb	// Value
]
```

### Two tuples with disjoint keys

Input: `([0x48, 0x19], [0xfe]), ([0x13, 0x14], [0xff])`

Expected trie structure spelled out for humans:

```
Branch
	0: Empty node
	1: Leaf ([0x03, 0x14], [0xff])
	2: Empty node
	3: Empty node
	4: Leaf ([0x08, 0x19], [0xfe])
	5: Empty node
	…
	f: Empty node
```

Annotated output:
```
[
	0b0100_0000,				// BRANCH
								// <–– TODO: why is there no length here?
	0x00, 						// slot 0
	0x0b, 0x00, 0x00, 0x00, 	// 11 – length in bytes of the following node
	0b1011_0000 + 3, 			// slot 1 LEAF; 176 + 3, i.e. the first of the remaining key nibbles (3'1'4')
		0x01, 0x00, 0x00, 0x00, // key length: 1 bytes
		0x14,					// key
		0x01, 0x00, 0x00, 0x00, // value length: 1 byte
		0xff,					// value
	0x00, 0x00, 				// slots 2,3
	0x0b, 0x00, 0x00, 0x00, 	// 11 – length in bytes of the following node
	0b1011_0000 + 8, 			// slot 4 LEAF; remaining nibbles: 8'1'9'; odd, so 8 goes into lower nibble
		0x01, 0x00, 0x00, 0x00, // key length: 1 bytes
		0x19,					// key
		0x01, 0x00, 0x00, 0x00, // value length: 1 byte
		0xfe,					// value
	0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, 0x00, // slots 5..15
	0x00, 						// slot 16,
]
```

### Complex example

Input is a set of key/value tuples:

```
([0xa7, 0x11, 0x35, 0x5], [0x2d]),
([0xa7, 0x7d, 0x33, 0x7], [0x01]),
([0xa7, 0xf9, 0x36, 0x5], [0x0b]),
([0xa7, 0x7d, 0x39, 0x7], [0x0c]),

```

Expected trie structure spelled out for humans:

```
Extension, partial key: 0xa7
	Branch
		0: Empty node
		1: Leaf ([0x01, 0x35, 0x5], 45)
		2: Empty node
		…
		7: Extension, partial key d3
			Branch
				0: Empty node
				1: Empty node
				2: Empty node
				3: Leaf ([0x03, 0x07], 0x01)
				…
				9: Leaf ([0x09, 0x07], 0x0c)
				…
				16: Empty node
		f: Leaf ([0x09, 0x36, 0x5], 0x0b)
```

Below is the expected binary encoding of the above.

**Note**: when building the trie the branches that are longer than the hashers length (32 bytes) are replaced with the hash of the content. In the bytes sequence showed below this has been disabled to show the full output of the trie construction.
```
let codec_trie = [
    0x80,	// 0b10000000 => extension
    0x1,	// length 1
    0x0,
    0x0,
    0x0,
    0xa7,	// payload: a7
    0x6b,	// length 107 bytes
    0x0,
    0x0,
    0x0,
    0x40,	// Branch node: 0b01_00_0000
    0x0,	// slot 0: empty node
    0xc,	// slot 1: 12 bytes follow
    0x0,
    0x0,
    0x0,
    0xb1,	// 0xb1 == 177 == 0b1011_0001 => 0b10_11_xxxx, leaf, odd length + 0001
    0x2,	// length: 2 bytes
    0x0,
		
    0x0,
    0x0,
    0x35,	// key payload
    0x5,	// key payload
    0x1,	// value length: 1 byte
    0x0,
    0x0,
    0x0,
    0x2d,	// value: 45; 12th byte, ends slot 1
    0x0,	// slot 2
    0x0,	// slot 3
    0x0,	// slot 4
    0x0,	// slot 5
    0x0,	// slot 6
    0x38,	// slot 7; item of length 56
    0x0,
    0x0,
    0x0,
    0x80,	// extension node, 0b10000000
    0x1,	// key length: 1 byte
    0x0,
    0x0,
    0x0,
    0xd3,	// key payload, 0xd3
    0x2e,	// item of length 46
    0x0,
    0x0,
    0x0,
    0x40,	// Branch node: 0b01_00_0000
    0x0,	// slot 0
    0x0,	// slot 1	
    0x0,	// slot 2
    0xb,	// slot 3, item of length 11
    0x0,	
    0x0,	
    0x0,	
    0xa0,	// payload, 0b1010_0000: leaf node, even length
    0x1,	// key length: 1 byte
    0x0,
    0x0,
    0x0,	
    0x7,	// partial key payload: 7
    0x1,	// value length: 1 byte
    0x0,
    0x0,
    0x0,
    0x1,	// value payload: 1
    0x0,	// slot 4
    0x0,	// slot 5
    0x0,	// slot 6
    0x0,	// slot 7
    0x0,	// slot 8
    0xb,	// slot 9,  item of length 11
    0x0,
    0x0,
    0x0,
    0xa0,	// payload, 0b1010_0000: lead node, even length
    0x1,	// key length 1 byte
    0x0,
    0x0,
    0x0,
    0x7,	// key payload: 7
    0x1,	// value length: 1 byte
    0x0,
    0x0,
    0x0,
    0xc,	// value payload: 12
    0x0,	// slot 11
    0x0,	// slot 12
    0x0,	// slot 13
    0x0,	// slot 14
    0x0,	// slot 15; end second branch node
    0x0,	// slot 16; second branch value slot
    0x0,	// slot 8 (first branch)
    0x0,	// slot 9
    0x0,	// slot 10
    0x0,	// slot 11
    0x0,	// slot 12
    0x0,	// slot 13
    0x0,	// slot 14
    0x0,	// slot 15
    0xc,	// slot 16; first branch value slot; item of length 12
    0x0,
    0x0,
    0x0,
    0xb9,	// 0xb9 == 185 == 0b1011_1001 => Leaf node, odd number, partial key payload = 9
    0x2,	// length: 2 bytes
    0x0,
    0x0,
    0x0,
    0x36,	// payload: 0x36, 0x5
    0x5,	
    0x1,	// length: 1 byte
    0x0,
    0x0, 
    0x0,
    0xb,	// value: 11
    0x0 // 107
]

==============
vec![
    0x80, // 0b10000000 => extension
    0x1,  // payload length: 1
    0x0,
    0x0,
    0x0,
    0xa7, // payload: partial key
    0x20, // 32? expected 0b0100_0000
    0x0,
    0x0,
    0x0,
    0x3e,
    0x8e,
    0xfa,
    0x13,
    0xd0,
    0xed,
    0xc5,
    0x8f,
    0xe8,
    0x4d,
    0xf7,
    0x8b,
    0x24,
    0x76,
    0x28,
    0x9b,
    0xa,
    0x6b,
    0x54,
    0x73,
    0xca,
    0xb4,
    0xbb,
    0x7e,
    0x5,
    0x7b,
    0x8e,
    0xc5,
    0x7f,
    0xbb,
    0x35,
    0xa
]
```
