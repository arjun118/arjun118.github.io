+++
date = '2025-11-09T20:09:57+05:30'
draft = false
title = 'Methods In Golang'
tags=['technical']
+++

I am skipping functions and directly jumping in to methods. I will cover functions in detail but let's get through methods
quickly since it's relatively a simple topic

If you are coming from any programming language, you are already familiar with methods.

# Declaration

A traditional functional declaration in golang is as below

```go
func Greet(name string) string{
    return fmt.Sprintf("hello, %s!", name)
}
```

When functions are tied to a particular type, they are called Methods (just a funny way of thinking about them). For that to happen there has to be some changes in the way declare method from the way we declare a function

Here is a sample method

## Example

```go
type Employee struct {
    ID int
    Salary int
    Designation string
}

func (e *Employee) Promote(newSalary int, newDesignation string){
    e.Salary= newSalary
    e.Designation=newDesignation
}
```

![method image annotated](/images/method_image.png)

Before moving forward, let's about the type of receivers we can have for method

- Pointer Receiver
- Value Receiver

what we have above is a pointer receiver.

By using a value receiver, whenever we call a method which has a value receiver, it makes a copy of each argument value which might not be desirable if the arguments are large.

when a method has a pointer receiver, any changes made on the fields of the receiver argument will reflect in the original receiver (because it is pass by receiver)

Let's create a sample employee

```go
emp:= Employee{
    1, 50000, "IC1"
}
```

Let's call the method `Promote`

```go
emp:= Employee{
    1, 50000, "IC1"
}
&emp.Promote(100000, "IC2")
```

the above code, `&emp.Promote(100000,"IC2")` does not work as intended because,in Go, method selection (emp.Promote) happens before any unary operator like & is applied.so the compiler first interprets emp.Promote(100000, "IC2") as a complete function call expression, evaluates it, and then tries to apply and to the result.

There are few correct ways of using it

# Calling a method with pointer receiver

## Method 1

```go
empptr:=&emp
empptr.Promote(100000, "IC2")
```

## Method 2

```go
(&emp).Promote(100000, "IC2")
```

## Method 3

the language helps us

```go
emp.Promote(100000, "IC2")
```

the compiler will perform an implicit **&emp** on variable emp

## Calling a method with value receiver

calling a method with value receiver is simple and self explanatory

lets declare a method with value receiver

```go
func (e Employee) IsFresher() bool{
    return e.Designation == "IC1"
}
```

using this is simple

```go
fresher:= e.IsFresher()
```

you can, quite possible call this method on a pointer to the `emp` instance and go will perform an implicit conversion

```go
empptr=&emp
fresher:= empptr.IsFresher()
```

# Struct Embedding

we had talked about struct embedding and anonymous fields in a struct and the ease of access they provide in my `composite data types` blog.

the idea here remains the same let's see how we can compose structs and use the methods of other structs by embedding them

## Example

```go
package main

import (
	"fmt"
	"image/color"
)

// Front -> front wheel width in mm
// Rear -> rear wheel width in mm

type Wheels struct {
    Front int
    Rear int
}

type Bike struct {
    Wheels
    Color color.RGBA
}

func (w *Wheels) UpgradeWheels(newWheels Wheels) {
	w.Front = newWheels.Front
	w.Rear = newWheels.Rear
}

func main(){

    // let's define my current bike - rtr 200
    var Rtr200 Bike = Bike{
		Wheels{90, 130},
		color.RGBA{0, 0, 0, 255}, //it's moslty black
	}

	// let's define the bike i want to own in the
	// near future - ktm duke 390
	var Ktm390 Bike = Bike{
		Wheels{110, 150},
		color.RGBA{255, 102, 0, 255}, //THAT ELECTRIC ORANGE LOOKS GORGEOUS
	}

	// let me quickly upgrade the types of my rtr200 to that to duke (don't do this IRL)

	Rtr200.UpgradeWheels(Ktm390.Wheels)
}

```

now if you print what `Rtr200` Wheels are you will see `{110,150}`

lets look at a few things that are happening here

- we are direclty accessing the method `UpgradeWheels` which has a pointer `Wheels` receiver from an instance of the struct `Bike` in which the `Wheels` struct is embedded
- The methods of `Wheels` have been promoted to `Bike`
- we can also access the methods of the `Wheels` in this format
  > Rtr.Wheels.UpgradeWheels(Ktm390.Wheels)
- notice that we are not passing the `Ktm390` variable as an argument direclty to the `UpgradeWheels` call. that is because the argument that `UpgradeWheels` accepts is still the type of `Wheels` and not `Bike`
- to funnily put, `Bike` is not `Wheels`, but it has `Wheels` and it has one additional method `UpgradeWheels` promoted by `Wheels`
- here is the process by which compiler looks for the declared method named `UpgradeWheels` when resolving the call to the same method
  - it first looks for a directly declared method named `UpgradeWheels` on `Bike`
  - then for methods promoted once from `Bike`'s embedded fields like `Wheels`
  - then for methods promoted twice from embedded fields within `Wheels` and so on

# Method Values

When we want to call a method

> we select and call a method in the same expression like

```go
e.IsFresher()
```

the whole idea of method values is that we can seperate these 2 actions and they have nice benifits

the selector

```go
e.IsFresher
```

yields a `method value`, a function that binds a method to a specific receiver value `e`. now the function can be invoked without a receiver value, it only needs the non-receiver arguments.

lets look at an example with a struct which defines a point in a cartesian plane

## Example

```go
type Point struct  {X, Y float64}

func (p Point) Slope(q Point) float64 {
    return (q.Y-p.Y)/(q.X-p.X)
}

func main(){
    p := Point{2, 4}
	q := Point{5, 11}

    slopeOfLineWithP:= p.Slope

    fmt.Println(slopeOfLineWithP(q))
}
```

a silly way to think about this is, assigning functions to a variable in javascript. p.Slope becomes a closure like function, one where p is already captured similar to how js functions can close over variables.

take a look at the signature of `slopeOfLineWithP`

```go
func(main.Point) float64
```

# Methods Expressions

the usual way we do a method call is , we supply the receiver in a special way using the selector syntax

```go
e.IsFresher()
```

now what a method expression is , it yields a function value with a regular first parameter taking the place of the receiver and rest all parameters carried so that we ca call it just like we would a function, without any selector syntax, here is an example

i am adding another method to our `Point` struct named `Translate`

## Example

```go
func (p *Point) Translate(angle float64) {

  oldX := p.X
  p.X = p.X*math.Cos(angle) - p.Y*math.Sin(angle)
  p.Y = oldX*math.Sin(angle) + p.Y*math.Cos(angle)

}
```

```go
p:=Point{1.5, 7}
q:=Point{3, 9.3}
slope := Point.Slope
fmt.Println(slope(p, q))
// lets look at the signature of the function
fmt.Printf("%T\n", slope)
translate := (*Point).Translate
translate(&p, 2*math.Pi)
fmt.Println(p)
fmt.Printf("%T\n", translate)
```

take a look at the ouptut

```go
1.5333333333333339
func(main.Point, main.Point) float64
{1.5000000000000018 7}
func(*main.Point, float64)
```

the function signature of slope and translate are what we expected. since `Translate` is a method which has pointer receiver , it has a pointer to the struct Point as it's first paramter

Thank you for reading!
