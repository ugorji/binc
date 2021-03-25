# Binc

**Author:** *Ugorji Nwoke*  
**Version:** *Final / May 16, 2013*

Simple is a lightweight, binary language-independent data interchange
format. It supports efficient and extensible encoding (and decoding) of
structured data. 

The overriding design goal is for it to be simple, compact, efficient,
complete and unrestricted. See the feature set below. 

## Feature Set

It supports:

- full spectrum of common and distinct data types.
- arbitrarily large precision signed and unsigned integers;
  Uses up to 2^64-1 bytes to represent integer value (for bignums, etc)
- efficient for common entries (using single bytes)
    - use single byte for special values (bool, null, NaN, +/-Infinity, etc)
    - use single byte for signed integers in range -1 to 16
    - embed length of small containers directly in descriptor byte (if len < 12).
      containers are string, byte array, array, map, extension.
- separate binary vs text type to denote a sequence of bytes
- builtin timestamp type
- custom user-defined types and extensions (up to 255 user-defined types supported)

## Stream and Data Types

A stream includes a value, which may be one of the supported data types:

- special values (True, False, null)
- float 32-bit
- float 64-bit
- unsigned (positive) integer
- signed (negative) integer
- string
- byte array
- array
- map
- timestamp
- custom extension

Each data value is represented by a "byte descriptor" (hereafter called
'bd') which either completely encapsulates the value, or denotes the
extra bytes to describe the value. 

### Special Values

bd represents the special value:

    0x1 Null   (anything)
    0x2 False  (boolean)
    0x3 True   (boolean)

### Integer

There are 2 types of integers: positive (or unsigned) and negative (always signed).

vs represents the number of bytes (l) which contain the integer value.

    l = vs+1

Once l is deciphered, then l subsequent bytes denote the integer value
encoded in big-endian form.

#### Positive (or unsigned) Integer

bd represents the number of bytes used to represent the number in big endian format

    0x8 1 byte
    0x9 2 bytes
    0xa 3 bytes
    0xb 4 bytes

#### Negative (signed) Integer

bd represents the number of bytes used to represent the number in big endian format

    0xc 1 byte
    0xd 2 bytes
    0xe 3 bytes
    0xf 4 bytes

### Float (32-bit)

bd = 0x4

float value is encoded in subsequent 4 bytes as IEEE 32-bit

### Double (64-bit)

bd = 0x5

float value is encoded in subsequent 8 bytes as IEEE 64-bit

### Timestamp

bd = 24

subsequent byte is the number of bytes `l` needed to encode the time.

subsequent `l` bytes encodes the time.

### Containers

Containers are string, byte array, array, map, custom extension.

For all containers, we first encode the length of the container as follows:

- encoding the number of bytes (0,1,2,4,8) needed for the length `l` along with bd
- subsequent `l` bytes denote the length

bd in this case defines the type of container

    216 string
    224 bytes
    232 array
    240 map
    248 ext

For illustration, in the table above:

- if bd == 216, then use 0 bytes to define length of string ie 0-length string
- if bd == 217, then subsequent 1 byte defines string length
- if bd == 218, then subsequent 2 bytes define string length
- if bd == 219, then subsequent 4 bytes define string length
- if bd == 220, then subsequent 8 bytes define string length

#### String

A string is a UTF-8 encoded byte array. 

#### Byte Array

A byte array is a raw byte-array, typically used for binary data.

#### Array

An array is an ordered list of values, where a value may be any of the
aforementioned supported data types.

#### Map

A map is an unordered collection of key/value pairs, where each key or
value may be any of the aforementioned supported data types.

#### User-Defined Custom Extension

A custom extension is a user-defined type with a user defined byte
format. Custom extensions are usually mapped to a user defined type by a
tag.

