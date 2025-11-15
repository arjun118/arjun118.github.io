+++
date = '2025-11-15T13:11:24+05:30'
draft = true
title = 'Interfaces in Golang'
tags=['technical']
+++

I am not familiar with the interfaces that exists in other languages, so i will try my best to explain what interfaces are in golang

# Introduction

by now you should be knowing the terms , `concrete type`, and `abstract type`

here are some quick pointers

- **concrete type**

  - specifies exact representation of its values (struct layout, slice header, pointer, etc.)
  - exposes tht the intrinsic operations of that representation (arithmetic for numbers, or indexing,append and range for slices)
  - may also provide additional behaviour through its methods
  - zero value is well defined

- **abstract type**

  - reveals only some of their methods
  - hide the underlying representation
  - cannot see underlying nil pointers without assertion
  - no built-in operations
  - you have a value of interface type, means you know nothing about it until you assert it

> interfaces are of abstract types

there are different built-in interfaces that go uses extensively

# Declaration

```go
type CustomInterface interface {
    CustomMethod(p []byte) string{
        // your method definition
    }
}
```

> an interface type specifies a set of **methods** that a concrete type must posses to be considered an instance of that interface

to draw parallel from the above declaration, if a type has a method implemented named `CustomMethod` on it with the same exact signature, then that type can be considered an instance of that interface (or that type satisfies the `CustomInterface`)

> you need not explicitly convery in any sort that a type satisfies or implements an interface.just implementing the methods on that type with the same signature and name that the interface declared is sufficient in go. The satisfaction is implicit. A type satisfies an interface if it possesses all the methods the interface requires. neat, right?

you can embed an interface inside an interface

here is an examples from the **io package**

```go
package io

type Reader interface {
  Read(p []byte) (n int, err error)
}
type Closer interface {
  Close() error
}

type ReadWriter interface {
   Reader
   Writer
}
type ReadWriteCloser interface {
   Reader
   Writer
   Closer
}
```

intuitively,for a type to satisfy `ReadWriter` interface should satisfy all the interfaces that are embedded in it.

```
An **empty interface** `interface {}`, places no demands on the types that satisfy it.we can assign any value to an empty interface
```
