

#### Why the name Binc?

The name Binc was chosen after considering some others, including:

- binpack
- bincodec
- swoosh
- swift

#### Did you consider other binary encodings?

The decision to create Binc was not done lightly. A lot of analysis for
features and performance was done. Other schema-free binary codecs were
evaluated before Binc was created. These include:

- bson: verbose format with features in use only by mongodb
- bjson: simplistic, lacks features
- ubjson: stays too true to json. lacks extensions, binary support
- msgpack: lacks timestamp, binary and extensions
- tnetstrings: simplistic and lacking features
- smile: complex. lacking features
- binary plist: simplistic and lacking features
- protocol buffers, thrift, avro: require schema and pre-compilation step

Msgpack came closest to fulfilling the design requirements. I had
settled on it and engaged the community and author to include timestamp
and distinct binary and string types. However, after a few months
working on it, progress just halted and could not be jumpstarted (see
https://github.com/msgpack/msgpack/issues/128).

However, I believe Binc has significant features beyond those provided
by msgpack, and stands tall on its own.

#### Have you done benchmarks backing up your performance claims?

We ran extensive benchmarks against other Go encoding libraries, to see
encoding size and performance. From our results, Binc encoded into a
much smaller data size while outperforming others significantly while
encoding and decoding.

    Benchmark: 
     Struct recursive Depth:             2
     ApproxDeepSize Of benchmark Struct: 15574 (size of in-memory data object in bytes)
    Benchmark One-Pass Run: (size of encoded data in bytes)
        msgpack: len: 5086
           binc: len: 3390
            gob: len: 4531
           json: len: 8250
           bson: len: 9838
    ..............................................
    Benchmark__Msgpack__Encode     5000     325819 ns/op
    Benchmark__Msgpack__Decode     5000     397460 ns/op
    Benchmark__Binc_____Encode     5000     304940 ns/op
    Benchmark__Binc_____Decode     5000     364586 ns/op
    Benchmark__Gob______Encode     5000     344352 ns/op
    Benchmark__Gob______Decode     5000     667213 ns/op
    Benchmark__Json_____Encode     5000     452073 ns/op
    Benchmark__Json_____Decode     2000     966823 ns/op
    Benchmark__Bson_____Encode     5000     617730 ns/op
    Benchmark__Bson_____Decode     2000     863080 ns/op

In summary, for an in-memory struct with size 16K:

| Codec   | Encoded Size (KB) | Encoding Time (ms) | Decoding Time (ms) |
|---------|-------------------|--------------------|--------------------|
| Binc    | 3.4 | 304 | 364 |
| Msgpack | 5.1 | 325 | 397 |
| JSON    | 8.2 | 452 | 966 |
| BSON    | 9.8 | 617 | 863 |
| Gob     | 4.5 | 344 | 667 |

#### How do you achieve the small encoded size?

The following features were instrumental:

- Symbols: We don't have to duplicate constant strings. Just encode once
  and refer by a 1 or 2-byte tag.
- Pruning insignificant leading bytes from integers.
- Pruning insignificant trailing bytes from floats. 
  For example, this allows us encode double-precision 17.0 using 2 bytes instead of 8.

