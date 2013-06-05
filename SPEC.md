# Binc

**Author:** *Ugorji Nwoke*  
**Version:** *0.2.0 / June 5, 2013*

**Change Log:**

1. 0.2.0 / June 5, 2013
    - Added Symbols, Compact Integers, Compact Floats
1. 0.1.0 / May 16, 2013
    - Initial Public release

Binc is a lightweight, binary language-independent data interchange
format. It supports efficient and extensible encoding (and decoding) of
structured data. 

The overriding design goal is for it to be simple, compact, efficient,
complete and unrestricted. See the feature set below. 

## Feature Set

In general, Binc is designed to be simple, compact, efficient, complete
and unrestricted. 

It supports:

- full spectrum of common and distinct data types.
- arbitrarily large precision signed and unsigned integers;
  Uses up to 2^64-1 bytes to represent integer value (for bignums, etc)
- full range of IEEE 754 2008 floating point types 
  (decimals, half-float, float, double, double extended, ...)
- efficient for common entries (using single bytes)
    - use single byte for special values (bool, null, NaN, +/-Infinity, etc)
    - use single byte for signed integers in range -1 to 16
    - embed length of small containers directly in descriptor byte (if len < 12).
      containers are string, byte array, array, map, extension.
- full unicode strings (utf8, utf16, utf32)
- builtin timestamp type
- builtin binary type (separate from strings)
- special values (NaN, +Inf, -Inf, Null, etc)
- custom user-defined types and extensions (up to 255 user-defined types supported)
- Efficient storage of symbols (const strings, enums, field names, etc)
- Efficient storage of floats; 
  trailing zeros are not encoded if they can save space.

## Binc Stream and Data Types

A Binc stream includes a Binc value, where a value may be one of the
supported data types.

The data types are:

- special values (True, False, null, NaN, +Infinity, -Infinity)
- float
- decimal
- unsigned integer
- signed integer
- small signed int (1 to 16)
- string
- symbol
- other unicode (UTF-16LE, UTF-16BE, UTF-32LE, UTF-32BE)
- byte array
- array
- map
- timestamp
- custom extension

Each data value is represented by a "byte descriptor" (hereafter called
'bd') which either completely encapsulates the value, or denotes the
extra bytes to describe the value. In general, bd (a 8-bit value) is
composed of 2 4-bit values, a value descriptor (hereafter called 'vd')
and a value specification (hereafter called 'vs'). 'vd' denotes the type
of the value, and the use of 'vs' is different for every 'vd'.

To represent length, we use the concept of degree of precision. The
degree of precision 'n' translates directly into the bits of precision
'b' required to represent its size: 'b' = 8 * 2^n. 0 <= n <= 3. For
example:

    if n = 0, b = 8 * 2^0 = 8 * 1 = 8  (represented by 1 byte e.g. uint8). 
    if n = 3, b = 8 * 2^3 = 8 * 8 = 64 (represented by 8 bytes e.g. uint64).

Let's look at each data type one by one.

### Special Values

vd = 0x0

vs represents the special value:

    0x0 Null   (anything)
    0x1 False  (boolean)
    0x2 True   (boolean)
    0x3 NaN    (float)
    0x4 +Inf   (float)
    0x5 -Inf   (float)
    0x6 0.0    (float)
    0x7 0      (signed int)
    0x8 -1     (signed int)

### Integer

There are 2 types of integers: signed and unsigned.

vs represents the number of bytes (l) which contain the integer value.

    If vs <= 7, l = vs+1.
    Else vs-7 bytes subsequent bytes are read, 
        the number is decoded in big endian format as l.

Once l is deciphered, then l subsequent bytes denote the integer value
encoded in big-endian form.

#### Unsigned Integer

vd = 0x1

#### Signed Integer

vd = 0x2

### Small Signed Integer

vd = 0x9

Some common integers (range 1 to 16) are stored directly in vs. This
way, common small integers are encoded as single bytes. 

The value here is vs+1.

### Float

vd = 0x3

vs is of the form 0bXYYY.

YYY denotes the type of float as recommended by IEEE 754, according to
the table below:

    0    binary16:   Half-Precision       (16-bit / 2 bytes)
    1    binary32:   Single-Precision     (32-bit / 4 bytes)
    2    binary32e:  extended             (40-bit / 5 bytes)
    3    binary64:   Double-Precision     (64-bit / 8 bytes)
    4    binary64e:  extended             (80-bit / 10 bytes)
    5    binary128:  Quadruple-Precision  (128-bit / 16 bytes)
    6    binary128e: extended (160-bit)   (160-bit / 20 bytes)
    7

n is number of bytes in table above. l is number of bytes used to encode
the float value.

    If X is not set, then l = n.
    Else subsequent byte contains value of l.

Once l is deciphered, then l subsequent bytes denote the float value. If l < n, 
assume trailing 0 bytes when decoding according to IEEE 754 format.

### Decimal

vd = 0xc

vs is of the form 0bXYYY.

YYY denotes the type of decimal as recommended by IEEE 754, according to
the table below:

    0    decimal32                        (32-bit / 4 bytes)
    1    decimal64                        (64-bit / 8 bytes)
    2    decimal64e extended              (80-bit / 10 bytes)
    3    decimal128                       (128-bit / 16 bytes)
    4    decimal128e extended             (160-bit / 20 bytes)
    5
    6
    7

n is number of bytes in table above. l is number of bytes used to encode
the float value.

    If X is not set, then l = n.
    Else subsequent byte contains value of l.

Once l is deciphered, then l subsequent bytes denote the float value. If l < n, 
assume trailing 0 bytes when decoding according to IEEE 754 format.

### Timestamp

vd = 0x8

vs is the number of bytes representing the timestamp (4 <= vs <= 14).

Subsequent vs bytes denote the timestamp value, which is represented as
a byte array containing a sequence of integer values in big endian
encoding, with optional timezone information.

To illustrate, we will use the key below:

    secs32:  seconds since unix epoch, an int32 value (4 bytes).
             support "current" timestamps: +/- 68 years from Unix Epoch 1/1/1970 i.e. 1902-2038
    secs:    seconds since unix epoch, an int64 value (8 bytes).
             support infinitely all timestamps.
    nsecs:   fractional seconds as nanoseconds, an int32 value (4 bytes).
    tz:      timezone offset in minutes east of UTC, a uint16 value (2 bytes).
             Timezone information is fully represented by a UTC offset (east or west) 
             and a dst (daylight savings time) flag. 
             Timezone offset has a range of 26 hours or 1560 minutes, 
             which is fully represented using the bottom 11 bits.
             The 3 most significant bits (Bits 15, 14 and 13) are used as below:
                 Bit 15 = offset_position: set to 1 if west of UTC (for example in UTC-08:00)
                 Bit 14 = have_dst: set to 1 if we set the dst flag.
                 Bit 13 = dst_on: set to 1 if dst is in effect at the time, or 0 if not.

As mentioned above, a timestamp value is a variable length byte array, where the length
determines the information stored:

    4 bytes: secs32
    6 bytes: secs32 tz
    8 bytes: secs32 nsecs
    10 bytes: secs32 nsecs tz
    .
    9 bytes: secs 0
    11 bytes: secs tz 0
    12 bytes: secs nsecs
    14 bytes: secs nsecs tz

### Containers

Containers are string, byte array, array, map, custom extension.

For all containers, vs represents the length of the container.

If vs is in the form 0b00YY, then YY represents degree of precision n (up
to 3) for the length. Subsequent 2^n bytes is represent the length.

If vs is not of the form 0b00YY, then vs represents the len directly (for
len < 12). len = vs - 4.

#### String

vd = 0x4

A string is a UTF-8 encoded byte array. 

After the container len (l) is encoded, subsequent l bytes represent the
string.

#### Byte Array

vd = 0x5

A byte array is a raw byte-array, typically used for binary data.

After the container len (l) is encoded, subsequent l bytes represent the
byte array.

#### Array

vd = 0x6

An array is an ordered list of values, where a value may be any of the
aforementioned supported data types.

After the container len (l) is encoded, l values are encoded one by
one in sequence.

#### Map

vd = 0x7

A map is an unordered collection of key/value pairs, where each key or
value may be any of the aforementioned supported data types.

After the container len (l) is encoded, l key/value pairs are encoded
one after the other. 

#### User-Defined Custom Extension

vd = 0xf

A custom extension is a user-defined type with a user defined byte
format. Custom extensions are usually mapped to a user defined type by a
tag.

After the container len (l) is encoded, the subsequent byte is the
tag. Thereafter, subsequent l bytes is the custom object encoded.

### Symbol

vd = 0xb

A symbol is a constant string, which is repeated a lot in the encoded stream. It usually
represents enumerated constants, field names or other constant string values.

vs is in the form 0bWXYY, described below:

    W:  If set, the symbol id occupies two bytes (ie 256 <= symbol <= 65536).
        If not set, symbol id occupies one byte (ie 0 <= symbol <= 255).
    X:  If set, the symbol id is followed by the string it represents.
        This is important because the first time a symbol is seen, it's value is recorded.
    YY: This is the degree of precision 'n' of the length of the string.
        It is only used if X above is set.

Subsequent 1 or 2 bytes denote the symbol integer value (depending on W setting above).

If X is set, then subsequent 2^n bytes represent the length l, and subsequent l bytes
represent the symbol string value.

### Unicode Other

vd = 0xa

Unicode other supports UTF-16LE, UTF-16BE, UTF-32LE, UTF-32BE.

vs is broken up into 2 parts: 0bXXYY. 

- XX represents the encoding. 
- YY represents the degree of precision n of the length. This means 0 <=
  n <=3, and consequently length can only go up to 2^64-1 (ie up to 64
  bits of precision).

XX values are:
  
    00     UTF-16BE
    01     UTF-16LE
    10     UTF-32BE
    11     UTF-32LE

subsequent 2^n bytes denote the length l. Subsequent l bytes denote the
string of bytes.

## Codec guidelines

Any given Binc codec library may not support the full format. This may
be due to language limitations or limits on the intended use of the
library. Each library should list its limitations.

For example, the Go library lists the following unsupported features:

- integer values beyond 64 bit integers
- decimals and floats other than IEEE 754 binary32 and binary64 floats
- unicode other 

However, people using the format for scientific data exchange, financial
data exchange, or specific localization concerns, may need big integers,
decimal types or UTF16-LE respectively. Other library implementations
may provide these encoding/decoding features.

## Misc

The recommended mime type is application/x-binc.

When writing out files using the Binc format, the recommended file
extension is "binc" (e.g. mydata.binc).

