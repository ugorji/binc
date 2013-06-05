# Binc

**Author:** *Ugorji Nwoke*  
**Version:** *0.1.0 / May 16, 2013*

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
- arbitrarily large precision signed and unsigned integers, 
  up to 2^15 bits of precision (for bignums, etc)
- full range of IEEE 754 2008 floating point types 
  (decimals, half-float, float, double, double extended, ...)
- efficient for common entries (using single bytes)
    - use single byte for special values (bool, null, NaN, +/-Infinity, etc)
    - use single byte for signed integers in range -1 to 16
    - embed length of small containers directly in descriptor byte (if len < 12).
      containers are string, byte array, array, map, extension.
- full unicode strings (utf8, utf16, utf32)
- builtin timestamp 
- builtin binary (separate from strings)
- special values (NaN, +Inf, -Inf, Null, etc)
- custom user-defined types and extensions (up to 255 user-defined types supported)

## Binc Stream and Data Types

A Binc stream includes a Binc value, where a value may be one of the
supported data types.

The data types are:

- special values (True, False, null, NaN, +Infinity, -Infinity)
- float
- unsigned integer
- signed integer
- small signed int (1 to 16)
- string
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

For some integer values, the concept of degree of precision 'n'
translates directly into the bits of precision 'b' required to store it:
'b' = 8 * 2^n. n can go up to 15. For example:

    if n = 0, b = 8 * 2^0 = 8 * 1 = 8  (represented by 1 byte e.g. uint8). 
    if n = 3, b = 8 * 2^3 = 8 * 8 = 64 (represented by 8 bytes e.g. uint64).
    if n = 15, b = 8 * 2^15 = 8 * 32768 = 262144 (representable as a bignum).

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
    0x6 0      (signed int)
    0x7 -1     (signed int)

### Unsigned Integer

vd = 0x1

This type represents unsigned integers.

vs represents the degree of precision (up to 15). Subsequent 2^n bytes
denote the actual value, encoded in big-endian form.

### Signed Integer

vd = 0x2

This type represents signed integers.

vs represents the degree of precision (up to 15). Subsequent 2^n bytes
denote the actual value, encoded in big-endian form.

### Small Signed Integer

vd = 0x9

Some common integers (range 1 to 16) are stored directly in vs. This
way, common small integers are encoded as single bytes. 

The value here is vs+1.

### Float

vd = 0x3

vs represents the type of float as recommended by IEEE 754, 
according to the table below (values in octal form):

    0000    binary16:   Half-Precision       (16-bit / 2 bytes)
    0001    binary32:   Single-Precision     (32-bit / 4 bytes)
    0002    binary32e:  extended             (40-bit / 5 bytes)
    0003    binary64:   Double-Precision     (64-bit / 8 bytes)
    0004    binary64e:  extended             (80-bit / 10 bytes)
    0005    binary128:  Quadruple-Precision  (128-bit / 16 bytes)
    0006    binary128e: extended (160-bit)   (160-bit / 20 bytes)
    0007
    
    0010    decimal32                        (32-bit / 4 bytes)
    0011    decimal64                        (64-bit / 8 bytes)
    0012    decimal64e extended              (80-bit / 10 bytes)
    0013    decimal128                       (128-bit / 16 bytes)
    0014    decimal128e extended             (160-bit / 20 bytes)
    0015
    0016
    0017

Subsequent n bytes (where n is number of bytes in table above) denote
the actual value, encoded in big-endian form.

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

### Unicode Other

vd = 0xa

Unicode other supports UTF-16LE, UTF-16BE, UTF-32LE, UTF-32BE.

vs is broken up into 2 parts: 0bXXYY. 

- XX represents the encoding. 
- YY represents the degree of precision n of the length, as in unsigned
  integer above. This means 0 <= n <=3, and consequently length can only
  go up to 64 bits of precision.

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

- integer values with degree of precision > 3 (ie beyond 64 bit integers)
- floats other than IEEE 754 binary32 and binary64 floats
- unicode other 

However, people using the format for scientific data exchange, financial
data exchange, or specific localization concerns, may need big integers,
decimal types or UTF16-LE respectively. Other library implementations
may provide these encoding/decoding features.

## Misc

The recommended mime type is application/x-binc.

When writing out files using the Binc format, the recommended file
extension is "binc" (e.g. mydata.binc).
