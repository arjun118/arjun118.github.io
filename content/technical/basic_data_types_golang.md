+++
date = '2025-09-15T09:24:09+05:30'
draft = false
title = 'Basic Data Types Golang'
tags=['technical']
+++

In this blog, we will go over some basic data types of golang.

This is just me writing about what i learn't in golang

Reference: [The Go Programming Language book](https://www.oreilly.com/library/view/the-go-programming/9780134190570/)

Go's data types fall in to 4 types

- **Basic types**: numbers, strings, booleans (these are what we will be going over in this blog.)
- **Aggregate types**: arrays, structs, slices etc. (in the coming blog)
- **Reference types**: pointers, maps, functions, channels etc. (in the coming blogs)
- **Interface type** (in the coming blogs)

# Basic Data Types

In this I'll cover intergers, floats, strings, boolean. Nothing too fancy. just in brief. If you already experienced in Go. Please skip this blog and save your time

## Integers

Integers come in 2 flavours in golang, `signed` and `unsigned`

each of them has again 4 different flavours

**int**- `int8`, `int16`, `int32`, `int64`
**uint** - `uint8`, `uint16`, `uint32`, `uint64`

as the name shows, they differ in their size

> Note: Keep in mind that, if you use just `int` and `unit` to intialize a variable, they assume their most efficient size for signed and unsigned integers on a particular platform. It will either be 32 or 64

### What is a rune?

rune is an alias of `int32` data type, they are the same underlying type which represents a 32 bit integer

why do you need an alias why you already have `int32` ? - just for convenience and clarity

rune becomes incredingly useful and makes sense when we are working with strings. we will have a look at it there

they are used to represent unicode points (more about this in the strings section)

### What is a byte?

a byte (you already know what a byte is if you are reading this), is an alias for `uint8` data type

it emphasizes the value of a piece of raw data but represents the same underlying data type as `uint8`

Signed integers span negative numbers too but unsigned intergers don't

### Range of signed and unsigned numbers

```
int8: -127 to 127 , one bit reserved for sign

uint8: 0 to 255
```

> Note: Explicit conversion is required to convert a value from one type to another type, even for the same flavours, to convert an `int8` to `int32` -> `int32(x)` where x is a variable of type `int8`

## Float

floats come in just 2 flavours -> `float32`, `float64`

> float64 is preferred for most purposes, because float32 computation accumulates error rapidly

## Boolean

boolean, as usual comes in 2 flavours

`true` and `false`

booleans can be combined with && (and), || (or) operations which have short circuit behaviour. && has higher precendence than ||

there is no implicit conversion or interpreation of booleans to int in golang. meaning you cannot do

```go
int(1==1)
```

this doesnot evaluate to 1

if you want, you need to convert them explicitly.

## Strings

in go, strings are simply a sequence of bytes

text strings are interpretedas UTF-8 encoded sequences of Unicode points (runes)

`len(string)` returns the number of bytes in a string and `s[i]` retreives the ith byte of the string

> the number of characters in a string may not be equal to what will be returned by len(string) because some characters in the string might be taking more than 1 byte

```go
s := "hello, world"
fmt.Println(len(s))  // "12"
fmt.Println(s[0], s[7]) // "104 119" ('h' and 'w')
```

> the ith byte of the string is not necessarily the ith character of the string because UTF-8 encoding of a non -ASCII code point requries 2 or more bytes

### ASCII AND UNICODE

previously (very very previously, long time i mean), there was only the american standard code for information interchange (ASCII). ASCII uses 7 bits to represent 12 characters

upper case, lower case english letters, digits, punctuation and [device control characters](https://www.geeksforgeeks.org/computer-organization-architecture/control-characters/)

to handle all the different data in myraid languages comes `unicode`

assign each one of distinct characters a standard number called a unicode code point or `rune` what we call in golang

### UTF-8

an encoding standard which provides variable length encoding of unicode points as bytes. it uses 1-4 bytes to represent each rune but only 1 byte for ASCII characters as they just occupy 1 byte

> higher order bit of the first byte of the encoding for a rune indicates how many bytes to follow

I was confused about this, here's what ChatGPT has to say

![UTF-8 encoding and decoding](/images/utf8_leadingbits_explanation.png)

### Iterating over strings

You can iterate over strings in golang in 2 ways

- conventional for loop
- using range (also for loop)

we need to talk about these differently 'cause they behave differently

#### **Using conventional for loop**

```go
p := "Hello, नमस्ते🙏"

```

when you use the loop like

```go
for i:=0; i<len(p); i++{
    fmt.Println("%d ",p[i])
}
```

the output will be

```bash
72 101 108 108 111 44 32 224 164 168 224 164 174 224 164 184 224 165 141 224 164 164 224 165 135 240 159 153 143
```

we are iterating over the individual bytes of the string, in essence this is expected behaviour because string is a sequence of bytes

#### Using range in for loop

When you use range in for loop

```go
for _, r := range p {
  fmt.Printf("%d ",r)
}
```

output will be

```bash
72 101 108 108 111 44 32 2344 2350 2360 2381 2340 2375 128591
```

if you observe closely the output will be same for first seven 7 characters of the string. this is because the first 7 characters are ASCII which occupies 1 byte each and the rest are not

You can also see that after `32` or the ' ', the numbers are not same. because conventional loop iterates over bytes and each rune can occupy multiple bytes.

to be very specific, we are representing the decimal representation of rune or unicode point

let's do another round of detailed printing

```go
for _, r := range p {
  fmt.Printf("rune: %q | unicode: %U | decimal: %d | bytes used: %d\n", r, r, r, utf8.RuneLen(r))
}
```

> utf8.RuneLen(r) return the number of bytes that a rune occupies, here r is a rune

here is the output

```bash
rune: 'H' | unicode: U+0048 | decimal: 72 | bytes used: 1
rune: 'e' | unicode: U+0065 | decimal: 101 | bytes used: 1
rune: 'l' | unicode: U+006C | decimal: 108 | bytes used: 1
rune: 'l' | unicode: U+006C | decimal: 108 | bytes used: 1
rune: 'o' | unicode: U+006F | decimal: 111 | bytes used: 1
rune: ',' | unicode: U+002C | decimal: 44 | bytes used: 1
rune: ' ' | unicode: U+0020 | decimal: 32 | bytes used: 1
rune: 'न' | unicode: U+0928 | decimal: 2344 | bytes used: 3
rune: 'म' | unicode: U+092E | decimal: 2350 | bytes used: 3
rune: 'स' | unicode: U+0938 | decimal: 2360 | bytes used: 3
rune: '्' | unicode: U+094D | decimal: 2381 | bytes used: 3
rune: 'त' | unicode: U+0924 | decimal: 2340 | bytes used: 3
rune: 'े' | unicode: U+0947 | decimal: 2375 | bytes used: 3
rune: '🙏' | unicode: U+1F64F | decimal: 128591 | bytes used: 4
```

now here is a simple run down of the verbs

- %q print the quoted string which the rune represents
- %U the unicode code point
- %d the decimal representation

here is some clarity over the number of bytes a string occupies and the number of runes it consists of.

We can get the number of runes a string consists of using `utf8.RuneCountInString(p)`

```go
fmt.Println("Number of runes: ", utf8.RuneCountInString(p))
fmt.Println("length of string (bytes): ", len(p))
```

Output

```bash
Number of runes:  14
length of string (bytes):  29
```

That's all for this blog. I'll write about the aggregate data types next.

Thank you for reading.
