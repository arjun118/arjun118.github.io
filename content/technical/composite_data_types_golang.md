+++
date = '2025-10-01T16:30:41+05:30'
draft = false
title = 'Composite Data Types Golang'
tags=['technical']
+++

Let's go over some composite (aggregate data types) that go has to offer

We will be covering

- arrays
- slices
- structs
- maps

# Arrays

Arrays are homoegeneous in golang.Meaning, all elements have the same type

Arrays are fixed in size. once you declare it there is not way for you to increase the size of the said array without creating a new one

indexing arrays in go is same as any other language

## Declaration and Usage

you declare an array in golang as

```go
var a[3]int
```

every element in the above array is initialzed to the empty value of it's element type (0 in this case)

### Looping over an array

```go
for i,x := range a{
    fmt.Println(i,x)
}
```

when yo loop over an array using range, for every element you will be able to use two values, the index of the element in the array and the array itself

if you don't want to the index, just use a `_`

```go
for _,x := range a{
    fmt.Println(i,x)
}
```

### Array Literal

```go
var q [3]int= [3]int{1,2,34}
```

there, you just initalized an array of 3 three elements. you donot need to mention the size of the array on the right side of the equality

it can be replaced using `...`

```go
var q [3]int= [...]int{1,2,34}
```

the compiler determines the array's size during compile time

the size of an array is a part of it's type. which means `[3]int` and `[5]int` are different types, meaning you cannot do this

```go
b:= [3]{1,2,3}
b=[4]{1,2,3,4}
```

array comparision works in the same way

```go
a1 := [2]int{1, 2}
b1 := [...]int{1, 2}
c1 := [2]int{1, 3}
fmt.Println(a1 == b1, a1 == c1, b1 == c1)
// above is entirely possible
d1 := [3]int{1, 2}
fmt.Println(a1 == d1)
// above is not possible, they are different types altogether
```

> in go, function arguments are not passed by reference they are passed by valuesince the
> arrays are inherently immutable their usage is very limited, even you pass them by reference
> there is no way to add or rempve array elements

# Slices

slices are light weight data structures which give access to the subsequence of all the elements of an array (underlying array)

a slice is made up of three components

- pointer
- length
- capacity

**Pointer** : the pointer points to the first element of the underlying array that is reachable through the slic

**Length**: number of elements in the slice which is always < capacity

```go
len(s)
```

**Capacity**: start of the slice and the end of the underlying array

```go
cap(s)
```

## Declaration and Usage

let's declare an array (underlying array for our slice)

```go
days := [...]string{"sunday", "monday", "tuesday", "wednesday", "thursday", "friday", "saturday", "weekend", "weekday"}
```

now we can get a slice

```go
slice1:=days[1:6]
```

```go
fmt.Printf("len of s %d\ncap of s %d\nval %v\n", len(slice1), cap(slice1), slice1)
```

```bash
len of s 5
cap of s 8
val [monday tuesday wednesday thursday friday]
```

here length of slice1 is self explanatory -> you have 5 elements in the slice

and the capacity as per definition is the start of the slice index to the end of the underlying array. since the slice starts from index 1, we will have 8 as capacity (one less than the length of the underlying array)

let's get an another slice, an overlapping one with the previous one

```go
slice2 := days[4:8]
```

here length of slice2 is self explanatory -> you have 4 elements in the slice

and the capacity as per definition is the start of the slice index to the end of the underlying array. since the slice starts from index 4, we will have 5 as capacity

```go
fmt.Printf("len of s %d\ncap of s %d\nval %v\n", len(slice2), cap(slice2), slice2)
```

```bash
len of s 4
cap of s 5
val [thursday friday saturday weekend]
```

you can always reslice a slice to expand it until it's capacity allows

for examples you can do something like

```go
slice1=slice[:8]
```

```go
fmt.Printf("len of s %d\ncap of s %d\nval %v\n", len(slice1), cap(slice1), slice1)
```

output:

```bash
len of s 8
cap of s 8
val [monday tuesday wednesday thursday friday saturday weekend weekday]
```

you cannot re-slice a slice past it's capacity

```go
slice2=slice[:6]
```

this will result in a panic because slice2 has a capacity of 5

### Slice Literal

```go
slice3 := []int{3, 4, 5, 2, 4, 5}
fmt.Println(slice3)
```

you can also create slices using `make`

```go
slice4:=make([]int, 3)
```

removing the size from the array declaration makes it a slice

**we cannot compare slices using == because slices are references and their elements are indirect**

> note that the element at index 0 still exists in the underlying array but just not reachable through slice1 or slice2 because they start at a later index

> changing the values from either of slice1 or slice2 changes the values of underlying arrays and the other slices too

here is what the logical representation of the slices look like

![overlapping slices](/images/overlapping_slices.png)

# Maps

in golang, map is a reference to a hash table. as in anyother language the keys of map should be hashable and comparable

## Declaraiton and usage

```go
prices:=make(map[string]int)
```

you can do a map literal as

```go
prices:= map[string][int]{
    "banana":23,
    "apple":45
}
```

adding elements to a map is straight forward, so is deleting

```go
prices["mango"] = 56
delete(prices, "apple")
```

### Looping over an array

```go
for fruit, price := range prices {
    fmt.Println(fruit, price)
}
```

when you try to access a key from a map, which does not exists, it returns the zero value
of the type of value it holds which is not very helpful if you are trying to truly know whether or not this key exists in the map. you can do that with

```go
price, exists := prices["banana"]
```

`exists` here will be a boolean. `true` if the element exists in the map or else `false`

# Struct

aggregate data type that groups together zero or more names values of arbitrary types as a single entity

each value is called a field

# Declaration and usage

let's define a `Employee` Struct

```go
type Employee struct {
  ID        int
  Name      string
  Address   string
  DoB       time.Time
  Position  string
  Salary    int
  ManagerID int
}
```

we can also group the field names which as the same type as

```go
type Employee struct {
  ID,Salary,ManagerID       int
  Name,Address     string
  DoB       time.Time
  Position  string
}
```

i will use the first method here.

initalize an empty struct as

```go
var employee1 Employee
```

if provided nothing, every field's value of an instance of a struct will be initialized to the zero of their respective types

but be sure to group those which are related because the field order is significant to the type identity

### Accessing fields

you can access inidivudal struct using the dot notation

```go
fmt.Println(employee1.ID)
```

we can also get the pointers to the individal fields

```go
salary := &employee1.Salary
*salary += 5000
fmt.Println(employee1.Salary)
```

the dot notation also works with the pointers to the structs

```go
var employee2 *Employee = &Employee{}
fmt.Println(employee2.ID)
employee2.Position = "Hard Worker"
fmt.Println(employee2.Position)
```

if you don't want a field of a struct to be accessed beyond the package where it is defined,
start the name of the field with a lower case letter

by default go's accesss control mechnaism allows fields that only begin with a captial letter to be exported

### Struct literal

let's define a `Point` type, to represent a cordinate in the cartesian plane

```go
type Point struct{ X, Y int }
```

you an initialize a point in 2 ways

pass the values to all the fields in the same order as they are declared in the struct

```go
p := Point{1, 2}
```

or just pass what you want or have

```go
p1 := Point{X: 2}
```

if you want your function to modify a struct which it receives it as an argument, you need to pass the pointer of the struct to that function

you can create a pointer to the instance of the struct in two ways

```go
pointerp := &Point{1, 2}
```

or

```go
pp := new(Point)
*pp = Point{1, 2}
```

## Struct Embedding and Anonymous fields

when you declare a field with no name but a type in a struct, they are called anonymous fields

to demonstrate their utility, let us define 2 structs

```go
type Circle struct {
  X, Y, Radius int
}
type Wheel struct {
  X, Y, Radius, Spokes int
}
```

we can extract X and Y into a new struct called `Point` as we did above, now the declarations become

```go
type Point struct {
  X, Y int
}
type Circle struct {
  Center Point1
  Radius int
}
type Wheel struct {
  Circle Circle
  Spokes int
}
```

we kind of embedded `Point` into `Circle` and `Circle` into `Wheel`

let's initialize a `Wheel` instance

```go
/ w := Wheel{}
w.Circle.Center.X = 8
w.Circle.Center.Y = 5
w.Circle.Radius = 10
w.Spokes = 30
fmt.Printf("%#v\n", w)
```

to access `X` and `Y` we need to write very verbose statements accessing each of the embedded struct, this can be easily done and bypassed using anon field

here is how the type declarations change

```go
type Point struct {
  X, Y int
}
type Circle struct {
  Point1
  Radius int
}
type Wheel struct {
  Circle
  Spokes int
}
```

now initializing the same wheel changes to

```go
w := Wheel{}
w.X = 8
w.Y = 5
w.Radius = 10
w.Spokes = 30
fmt.Printf("%#v\n", w)
```

we can just refer to the names of the individual fields of the structs which are embedded as
anonymous fields. the anonymous fields still has a name, that is just equal to type name

the above operation is equivalent to,

```go
w := Wheel{}
w.Circle.Point1.X = 8
w.Circle.Point1.Y = 5
w.Circle.Radius = 10
w.Spokes = 30
fmt.Printf("%#v\n", w)
```

That's all for this blog.Thank you for reading.
