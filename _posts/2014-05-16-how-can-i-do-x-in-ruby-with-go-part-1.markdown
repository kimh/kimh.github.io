---
layout: post
title: "(WIP) Ruby To Go: How can I do X in Ruby with Go? (Part I)"
date: 2014-05-16 23:39
comments: true
permalink: /blog/en/go/how-can-i-do-x-in-ruby-with-go-part-1/
categories:
- en
- go
---
I am a programmer who can write decent Ruby and some Javascript.

These two languages were all I know. I wanted to add Go to my list of programming language, so I started learning Go.

It's boring to read programming books to study programming languages, so I decided to learn Go by porting some programs written in Ruby.

While I was porting Ruby program to Go, I came to think it is useful if there is a cheatsheet that I can loop up in order to convert idiomatic Ruby code to Go.

Ruby and Go are totally different language, so sometimes it is impossible to simply translate Ruby code to Go.
However, it is possible in most cases to write Go code that is sematically equivalant to Ruby code.

So, here is a ***How can I do X in Ruby with Go?*** cheetsheet.

I hope you find it useful.

## Contents of this artcile
### Array and Enumerable Operation
* [Create array](#create_array)
* [Append element to array](#append_an_element_to_array)
* [Concatenate arrays](#concatenate_arrays)
* [Create multi dimension array](#create_multi_dimension_aray)
* [Create empty array](#create_empty_array)
* [Iterate on array](#iterate_on_an_array)
* [Looping N times](#looping_n_times)
* [Clone array](#clone_array)
* [Accessing elements of array by range](#accessing_elements_of_an_array_by_range)
* [Compare array](#compare_array)
* [Check if array includes an element](#check_if_array_includes_an_element)

### Method Definition
* [Define a method with optional parameter](#define_a_method_with_optional_parameter)
* [Define a method with variable length arugment](#define_a_method_with_variable_length_argument)

### MathematicOperation
* [Modular of negative number](#modular_of_negative_number)

### Misc
* [Nil checking](#nil_checking)
* [Checking the class](#checking_the_class)

<a id="array_and_enumerable_operation"></a>
## Array and Enumerable Operation
Array is very powerful data structure and enumerable is probably the most frequently used object in Ruby.

In Go, we have two different enumerable data structures: **array** and **slice**.
I don't write about details about them since it is not the goal of this post, but array is low-level data structure that slice refers to.

<a id="create_array"></a>
### Create array
***Ruby:***
```ruby
numbers = [1,2,3]
fruits = ["apple", "banana", "grape"]
```

***Go:***
```go
numbers := []int{1,2,3}
fruits := []string{"apple", "banana", "grape"}
```
In the case of Go, `numbers` and `words` are **slice**, not **array**. Array is primitive data structure, not frequently used in Go code.
If you want to archive similar things to Ruby array, slice should work for you.

<a id="append_an_element_to_array"></a>
### Append an element to array
***Ruby:***
```ruby
numbers = [1,2,3]
numbers << 4
```

***Go:***
```go
numbers := []int{1,2,3}
numbers.append(numbers, 4)
```
**append** adds elements to slice and ***return new slice***. Therefore, you have to reassign to itself.

<a id="concatenate_arrays"></a>
### Concatenate arrays
***Ruby:***
```ruby
numbers1 = [1,2,3]
numbers2 = [4,5,6]
numbers1 = numbers1 + numbers2
```

***Go:***
```go
numbers1 := []int{1,2,3}
numbers2 := []int{4,5,6}
numbers1 = append(numbers1, numbers2...)
```
`...` suffix on the slice indicates that it should be passed as the variadic argument, expanded as each `int` elements inside `append`.
Thus, this is equivalent to below:

```go
numbers1 := []int{1,2,3}
numbers1 = append(numbers1, 4, 5, 6)
```
<a id="create_multi_dimension_aray"></a>
### Create multi dimension array
***Ruby:***
```ruby
multi_array = [[1,2,3],[4,5,6]]
```

***Go:***
```go
var mul [][]int = [][]int{ {1, 2, 3}, {4,5,6} }
```
<a id="create_empty_array"></a>
### Create empty array
***Ruby:***
```ruby
array = []
```

***Go:***
```go
var array []int
```
When slice is declared, but not initialised, the slice points to an array of size 0.

<a id="iterate_on_an_array"></a>
### Iterate on array
***Ruby:***
```ruby
numbers = [1,2,3]
numbets.each {|num| puts num }
```

***Go:***
```go
numbers := []int{1,2,3}
for _, num := range numbers {
  fmt.Println(num)
}
```
If you want to access the index while iterating over the slice, replace `_` with other variable, for example, `i`.
```go
numbers := []int{1,2,3}
for i, num := range numbers {
  fmt.Println("index: ", i, "number: ", num)
}
```

<a id="looping_n_times"></a>
### Looping N times
***Ruby:***
```ruby
5.times {|num| puts num}
```

***Go:***
```go
for num:=0; num <5; num++ {
  fmt.Println(num)
}
```

<a id="clone_array"></a>
### Clone array
***Ruby:***
```ruby
new_array = old_array.clone
```

***Go:***
```go
new_array := make([]int, len(old_array))
copy(new_array, old_array)
```

<a id="accessing_elements_of_an_array_by_range"></a>
### Accessing elements of array by range
***Ruby:***
```ruby
numbers=[1,2,3,4,5]
numbers[0..3]
```

***Go:***
```go
ary := []int{1,2,3,4,5}
ary[0:4]
```
Note that, with `from:to`, `to` is the index where to end **but not including the index itself**.

<a id="compare_array"></a>
### Compare array
***Ruby:***
```ruby
if ary1 == ary2
  puts "Same array"
end
```

***Go:***
```go
same := true
for i, elm:= range ary1 {
   if ary2[i] != r { same = false }
}

if same == true {
  fmt.Println("Same slice")
}
```
You cannot compare slice in Go. You will get `slice can only be compared to nil` error if you try to do that.

<a id="check_if_array_includes_an_element"></a>
### Check if array includes an element
***Ruby:***
```ruby
fruits = ["apple", "banana", "grape"]

if fruits.include?("apple")
  puts "include!"
end
```

***Go:***
```go
include := false
fruits := []string{"apple", "banana", "grape"}
for _, elm := range fruits {
  if elm == "apple" {
    include = true
    break
  }
}

if include == true {
  fmt.Println("include!")
}
```

## Method Definition
There are two things that are equivalant to Ruby's method in Go: **method** and **function**.
Method is a type of function but requires specific receiver.

<a id="define_a_method_with_optional_parameter"></a>
### Define a method with optional parameter
***Ruby:***
```ruby
def greeting(word="hello!")
  puts word
end
```

***Go:***

This is not possbile in Go. Go does not support optional parameter in function or method definition.
One workaround is using struct.
```go
type greetingArg struct { word string }
func greeting(opt greetingArg) {
  word := opt.word
  if word == "" {
    fmt.Println("hello!")
  } else {
    fmt.Println(word)
  }
}

greeting(greetingArg{})
greeting(greetingArg{"bye!"})
```

<a id="define_a_method_with_variable_length_argument"></a>
### Define a method with variable length argument
***Ruby:***
```ruby
def foo(*args)
  args.each {|arg| puts arg}
end
```

***Go:***
```go
func foo(arg ...int) {
  for _, arg := range arg {
    fmt.Prinln(arg)
  }
}
```


## Mathematic Operation

<a id="modular_of_negative_number"></a>
### Modular of negative number
Both Ruby and Go supports modular of negative number. However, their behavior is different.

***Ruby:***
```ruby
-5 % 3 => 1
```

***Go:***
```go
-5 % 3 => -2
```

Go follows ***truncated toward zero*** for the division of negative number.

If you want to get the same value that Ruby returns, here is how to do this.

***Go:***
```go
divider := 3
mod := -5 % divider
if mod < 0 {
	mod = mod + divider
}
```

Note that values returnd by Ruby and Go are both mathematically correct. It's just there are two ways to define negative modulo.

## Misc
<a id="nil_checking"></a>
### Nil checking
***Ruby:***
```ruby
if val.nil?
  puts "val is nil"
end
```

***Go:***
```go
var str string
if str =="" {
  fmt.Println("str is empty")
}

var i int
if i == 0 {
  fmt.Println("i is zero")
}
```

When you declear a variable without intialization, the variable is set to zero value for its type.
Below is default zero value for primitive types.

| Type | Value |
| ----- |:------:|
| string | **''**
| int    | **0**
| float  | **0.0**
| boolean | **false**
| pointer | **nil**
| interface | **nil**
| slice | **nil**
| map | **nil**

<a id="checking_the_class"></a>
### Checking the class
***Ruby:***
```ruby
puts "abc".class
```

***Go:***
```go
import "reflect"

fmt.Println(reflect.TypeOf("abc"))
```
