+++
date = '2025-11-15T13:11:24+05:30'
draft = false
title = 'Interfaces in Golang'
tags=['technical']
+++

I am not familiar with the interfaces that exists in other languages, so i will try my best to explain what interfaces are in golang

# Introduction

by now you should be knowing the terms , `concrete type`, and `abstract type`

here are some quick pointers

- **concrete type**

  - specifies exact representation of its values (struct layout, slice header, pointer, etc.)
  - exposes that the intrinsic operations of that representation (arithmetic for numbers, or indexing,append and range for slices)
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
    CustomMethod(p []byte) string
}
```

> an interface type specifies a set of **methods** that a concrete type must posses to be considered an instance of that interface

to draw parallel from the above declaration, if a type has a method implemented named `CustomMethod` on it with the same exact signature, then that type can be considered an instance of that interface (or that type satisfies the `CustomInterface`)

> you need not explicitly declare that a type satisfies or implements an interface.just implementing the methods on that type with the same signature and name that the interface declared is sufficient in go. The satisfaction is implicit. A type satisfies an interface if it possesses all the methods the interface requires. neat, right?

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

An empty interface `interface {}`, places no demands on the types that satisfy it.we can assign any value to an empty interface

# Usage

lets see some examples

```go

package main

import "fmt"


type Vehicle interface {
	HasAWD() bool
	HasSufficientClearance() bool
}

type Bike struct {
	NoOfWheels      int
	CC              float64
	GroundClearance int
	HasSpokeWheels  bool
}

func (b Bike) HasAWD() bool {
	// technically current motorcycles does
	// not have all wheel drive,we are just mimicking here
	// treat spoke wheels as AWD
	if b.HasSpokeWheels {
		return true
	}
	return false
}

func (b Bike) HasSufficientClearance() bool {
	return b.GroundClearance >= 210
}

func (b Bike) OnlyOnBike() bool {
	return true
}

type Car struct {
	NoOfWheels      int
	CC              float64
	GroundClearance int
	AllWheelDrive   bool
}

func (c Car) HasAWD() bool {
	return c.AllWheelDrive
}

func (c Car) HasSufficientClearance() bool {
	return c.GroundClearance >= 210
}

func IsOffRoadSuitable(v Vehicle) bool {
	return v.HasAWD() && v.HasSufficientClearance()
}

func main(){

  Ktm390 := Bike{
		2, 398.63, 183, false,
	}
	Ktm390Adv := Bike{
		2, 398.63, 237, true,
	}

	TharRoxx := Car{
		4, 2148.0, 226, true,
	}

	fmt.Println(IsOffRoadSuitable(Ktm390))
	fmt.Println(IsOffRoadSuitable(Ktm390Adv))
	fmt.Println(IsOffRoadSuitable(TharRoxx))

  veh:= Vehicle(Ktm390)
  fmt.Println(veh.OnlyOnBike()) // you cannot call this, explanation below
}
```

looking at the above code, we have an interface `Vehicle` which states that it requires 2 methods to be implemented by any type for them to satisfy it.

we also have two struct types `Bike` and `Car` which implements these methods hence satisfying the interface.

`IsOffRoadSuitable` is a function which has a parameter of type `Vehicle`. since both `Bike` and `Car` satisfy this interface we can pass the instances of both `Bike` and `Car` as an argument to this function call and that is precisely what we are doing the following lines

consider the below code tho,

```go
veh.OnlyOnBike()
```

here `OnlyonBike` is a method only defined on `Bike` and the interface does not specify anything about this method. so trying to call this method on the instance of an interface would result in a compile-time error. you can also think of this as, veh is of static type `Vehicle` hence the compiler only allows only those that are listed in the interface

# Interface Values

what does an interface value, a value of an interface type actually hold?

well, it has 2 components.

- concrete type
- value of that type

or so called in golang

- dynamic type
- dynamic value

the type component in the interface value is represented by the type descriptor of the concrete type

![interface value annotated](/images/interface_value.png)

lets see some examples and print out some types

```go
var w io.Writer
```

`io.Writer` is a very widely used interface in golang. when declared like above, it assumes the zero-value of the interface which is `nil`.

for an interface value to be nil, both the type and value components has to be nil.

you can test whether or not the interface is nil using `w==nil` and `w!=nil`

![nil interface value annotated](/images/nil_interface_value.png)

let's change this

```go
w=os.Stdout
```

`os.Stdout` is of type `*os.File`, you can check that using

```go
fmt.Printf("%T\n", os.Stdout) // "*os.File"
```

`*os.File` implements a `Write(p []byte) (n int, err error)` which is required by `io.Writer` interface and hence that satisfies the interface.

calling `Write` method on an interface value containing an `*os.File` pointer causes the `(*os.File).Write` methods to be called

in this case

```go
w.Write([]byte("ktm 390 duke"))
```

prints the same string to the stdout (terminal)

looking at some buffers

```go
w= new(bytes.Buffer)
```

dynamic type is now `*bytes.Buffer` and the dynamic value is
a pointer to the newly allocated buffer

since `*bytes.Buffer` also satisfies `io.Writer`, we can the Write method to append the string to the buffer

```go
w.Write([]byte("wow what a string"))
fmt.Println(w.String())
```

the above Write method call appends the string to the buffer

# Emptiness

before closing this lets truly understand when an interface value is truly `nil` and when it's dynamic value is `nil`

> a nil interface value contain no value at all

i'll try to explain the distinction with an example direct from the book

```go
const debug = true

func f(out io.Writer) {
  // when debug is true
  // an empty buffer is created
  // when called f(buf), there is
  // dynamic value of out != nil pointer
  //  but a non-nil pointer referring to an empty buffer
  // and to which we can write to
  //   hence this works

  // but if the debug is false
  // the zero value of buffer is passed (nil) to f
  // and the dynamic value of out be nil and
  // not some pointer pointing to an empty memory space,ready
  // to be written
  // hence the error
  fmt.Printf("Dynamic Type of out: %T\n",out)
  fmt.Printf("Dynamic Value of out: %#v\n",out)
  if out != nil {
    out.Write([]byte("f is writing this\n"))
  }
}

func main() {
   var buf *bytes.Buffer
   // originally buf is totally  nil,nil pointer, the zero value of the
   // buffer
  //    fmt.Printf("zero value of the buffer: %#v", buf)
   if debug {
     buf = new(bytes.Buffer)
	 // non-nil empty buffer
	 // now buffer is empty, but not nil
     // buf points to something which is empty,
     // that something has no data yet
   }
   f(buf)
   if debug{
	fmt.Println("content of buffer: " ,buf.String())
   }
}
```

lets see the results with `debug=true`

```go
Dynamic Type of out: *bytes.Buffer
Dynamic Value of out: &bytes.Buffer{buf:[]uint8(nil), off:0, lastRead:0}
content of buffer:  f is writing this
```

when `debug=false`

```go
Dynamic Type of out: *bytes.Buffer
Dynamic Value of out: (*bytes.Buffer)(nil)
panic: runtime error: invalid memory address or nil pointer dereference
[signal 0xc0000005 code=0x1 addr=0x20 pc=0x7ff6b4c0b977]

goroutine 1 [running]:
bytes.(*Buffer).Write(0x7ff6b4c7f8c8?, {0xc000012198?, 0x7ff6b4c5abe7?, 0x1a?})
        C:/Program Files/Go/src/bytes/buffer.go:182 +0x17
main.f({0x7ff6b4c7f8a8, 0x0})
        C:/Users/Vishnu/Desktop/DEV/golang/the_go_programming_language_book/interfaces/main.go:255 +0x122
main.main()
        C:/Users/Vishnu/Desktop/DEV/golang/the_go_programming_language_book/interfaces/main.go:184 +0x1c
exit status 2
```

take a look at the dynamic value of out

we can make our f more robust,

```go
func f(out io.Writer) {
	fmt.Printf("%T\n", out)
	fmt.Printf("%#v\n", out)
	if out == nil {
		fmt.Println("truly nil interface")
		return
	}

	if b, ok := out.(*bytes.Buffer); ok && b == nil {
		fmt.Println("out is a typed nil *bytes.Buffer")
		return
	}

	if out != nil {
		out.Write([]byte("done!\n"))
	}
}
```

here is some visualization to help you

![emptiness_example_annotated](/images/emptiness.png)

thanks for reading! see you later, once i learn more
