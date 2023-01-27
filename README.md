# Working with Go

This is an opinionated list of the most important things that someone should have explained to me before I started working with Go. The bad news is that the things that I'm going to present here aren't mentioned in most beginner guides for Go (or at least not in the guides that I read).

I wrote this document during my employment at [knowis AG](https://www.knowis.com/). I created it in preparation for a series of knowledge transfer sessions which should introduce new colleagues to Go. I thank my former employer for allowing me to publish this document online.

## 1. The Go documentation is great

The official Go documentation is available at [go.dev/doc](https://go.dev/doc/).

If you're interested in a crash course then I recommend the [Tour of Go](https://go.dev/tour). [Effective Go](https://go.dev/doc/effective_go) provides you with a good foundation for writing good code in Go. The [FAQ](https://go.dev/doc/faq) also gives some advice on how to write good code and how to use some language features.

I also recommend to read the [Uber Go Style Guide](https://github.com/uber-go/guide/blob/master/style.md).

The [official language documentation](https://go.dev/ref/spec) is great. It's short, concise, has a good table of contents and reads very well. It can really help you if something doesn't seem to work as you would expect.  Some good example questions: "Why can't I pass x as an argument to function y?", "Why does my code behave non-deterministic when I'm iterating over a map?", "What happens if I delete a list element while iterating over the list?", "What does keyword z do?", ...

Hint: The answer for the second question is: "The iteration order over maps is not specified and is not guaranteed to be the same from one iteration to the next." This is documented in the section on [for loops](https://go.dev/ref/spec#For_range).

Last but not least: Go has a great [standard library](https://pkg.go.dev/std). I recommend to have a look on the available modules to get an overview about what features are already built into the language. *Hint*: [embed](https://pkg.go.dev/embed) is my favorite package; I haven't yet seen anything similar in other languages.

The biggest pain point of the Go documentation is that the documentation is distributed about multiple pages.
  - Most resources are available on or linked by the official documentation ([go.dev/doc](https://go.dev/doc/)).
  - The documentation of **every** public Go library is available on [pkg.go.dev](https://pkg.go.dev/).
    - The documentation for the standard library is available on [pkg.go.dev/std](https://pkg.go.dev/std)
    - The documentation for every other public library is available on pkg.go.dev/{{import-path-of-the-library}}. For example the documentation of `go.uber.org/zap` can be found at [pkg.go.dev/go.uber.org/zap](https://pkg.go.dev/go.uber.org/zap).
  - Some useful information can be found in the [Go wiki on GitHub](https://github.com/golang/go/wiki). I don't know who is responsible for this content and weather it is accurate and up to date, but it provides a lot of information. I would therefore recommend to first check the official documentation at [go.dev/doc](https://go.dev/doc/) before using the Wiki. However, the official documentation is not always complete. I already encountered a situation where the wiki provided important information that simpliy wasn't mentioned in the official documentation.

## 2. Pointers and values

Go supports two different kinds of variables: Pointers and Values. If you create a value with a literal (e.g. `stringVar := "hello world"`, `structVar := myStruct{}`) you create a value variable. You can obtain a pointer using a `&` sign (e.g. `pointerStruct := &myStruct{}`, `pointerVar := &valueVar`).

### 2.1 Understanding what's a pointer and what not

The variable type usually tells you weather you're working with a pointer or with a value.

```go
pointerVar := &myStruct{} // pointerVar is of type *myStruct
```

However, there are a few exceptions when it comes to builtin types.

#### 2.1.1 Slices

The [language specification](https://go.dev/ref/spec#Slice_types) says: "A slice is a descriptor for a contiguous segment of an underlying array". In other words: A slice is a pointer (plus some metadata) to an underlying array. If you create a copy of a slice you copy the pointer and the metadata. That's a problem if the slice is modified.

Example:

```go

// https://go.dev/play/p/gSMsLtyXz6G

package main

import "fmt"

func main() {
  originalValue := []int32{1, 2, 3}

  copiedValue := originalValue
  copiedValue[1] = 200
  copiedValue = append(copiedValue, 400)

  fmt.Println(originalValue) // prints [1 200 3]
  fmt.Println(copiedValue)   // prints [1 200 3 400]
}
```

copiedValue contains a pointer to the same array as originalValue. Modifying the content of copiedValue modifies originalValue. Appending new items to copiedValue doesn't modify the originalValue. [This blogpost](https://medium.com/swlh/golang-tips-why-pointers-to-slices-are-useful-and-how-ignoring-them-can-lead-to-tricky-bugs-cac90f72e77b) describes the problem in more detail.

Take-away-lesson: Use a pointer to a slice if you want to modify the slice (`copiedValue := &originalValue`). This allows you to append new data to the slice because you're modifying the original variable and not a copy of it. Unfortunately you have to dereference the pointer before you can use the slice (and you have to watch out for NullPointerExceptions!) which makes the code more unreadable.

```go
// https://go.dev/play/p/flYWEINMyhH

package main

import "fmt"

func main() {
  originalValue := []int32{1, 2, 3}

  copiedValue := &originalValue
  (*copiedValue)[1] = 200
  *copiedValue = append(*copiedValue, 400)

  fmt.Println(originalValue) // prints [1 200 3 400]
  fmt.Println(*copiedValue)  // prints [1 200 3 400]
}
```

#### 2.1.2 Maps

Maps are pointers. It's as simple as that.

Every change that's made to a copy of a map is also made to the original map because both maps point to the same underlying data.

```go
// https://go.dev/play/p/umP8z8L275A

package main

import "fmt"

func main() {
	originalMap := map[string]string{
		"a": "originalValue",
		"b": "originalValue",
		"c": "originalValue",
	}

	copiedMap := originalMap
	delete(copiedMap, "a")
	copiedMap["c"] = "updatedValue"
	copiedMap["d"] = "newValue"

	fmt.Println(originalMap) // prints map[b:originalValue c:updatedValue d:newValue]
}
```

#### 2.1.3 Interfaces

Interfaces are complicated. The behavior of interfaces depends on weather you're storing a value or a pointer type inside the interface. Because interfaces just describe a method set ("I need something which implements these methods") it's impossible to tell weather that "something" is a value or a pointer. You can store pointers and values inside the same interface and they can not be distinguished afterwards.

If you store a value inside an interface the interface behaves like a value. The content is always copied.

```go
// https://go.dev/play/p/rkyaNo3dFB9

package main

import "fmt"

type CounterInterface interface {
  GetCount() int
}

type counter struct {
  number int
}

func (c counter) GetCount() int {
  return c.number
}

func main() {
  var interfaceVar CounterInterface = nil // interfaces can store values, e.g. nil

  valueVar := counter{number: 10} // this is a value variable
  interfaceVar = valueVar         // interfaces can store values

  fmt.Println(valueVar.GetCount())     // prints 10
  fmt.Println(interfaceVar.GetCount()) // prints 10

  // the interface just stores a copy of valueVar
  valueVar.number = 12345
  fmt.Println(valueVar.GetCount())     // prints 12345
  fmt.Println(interfaceVar.GetCount()) // prints 10
}
```

If you're storing a pointer inside an interface the interface behaves like a pointer. The pointer (the int64 which stores the memory address) is always copied. The referenced data is _not_ copied.

```go
// https://go.dev/play/p/chSOSa1Z-pK

package main

import "fmt"

type Incrementer interface {
  Increment()
}

type counter struct {
  number int32
}

func (c *counter) Increment() {
  c.number++
}

func main() {
  var interfaceVar Incrementer = nil // interfaces are pointers which can be nil
  var interfaceVar2 Incrementer = nil

  counterVar := &counter{number: 0} // this is a pointer
  interfaceVar = counterVar         // pointers can be stored in interfaces
  interfaceVar2 = interfaceVar

  counterVar.Increment() // all variables point to the same struct which is incremented three times
  interfaceVar.Increment()
  interfaceVar2.Increment()

  fmt.Println(counterVar)    // prints &{3}
  fmt.Println(interfaceVar)  // prints &{3}
  fmt.Println(interfaceVar2) // prints &{3}
}
```

### 2.2 nil and interfaces

Technically speaking this section doesn't belong here because it isn't related to pointers and values. However, most of this section discusses interfaces so this piece of information fits best here.

I only encountered this situation when writing unit tests, so I will explain it with a unit test example. If you write something like `assert.Equal(a, b)` and get an error message like `"nil" is not equal to "interface{}(nil)"` then you should read this entry in the FAQ: [Why is my nil error value not equal to nil?](https://go.dev/doc/faq#nil_error)

### 2.3 Copying variables

If you assign something to a variable (`varA = varB`) or pass a variable to a function (`myFunction(myVariable)`) a _copy_ of the variable is created. To be precise: The section of the memory that's used by the variable is copied.

If it's a value variable the value will be _shallow-copied_. A deep copy is not performed!

If it's a pointer variable (which is just a glorified int64 variable which points to a memory address) the memory address is copied.

Here's an example. Note that all code examples in this document are runnable and contain a link to the Go Playground. The Go Playground provides the possibility to run and experiment with Go programs in the browser.

```go

// https://go.dev/play/p/_6dlVyijSq1

package main

import (
	"encoding/json"
	"fmt"
)

type valueType struct {
	A int32
	B int64
	NestedValue otherType
	NestedPointer *otherType
}

type otherType struct {
	Message string
}

func main() {
	originalValue := valueType{
		A: 1,
		B: 2,
		NestedValue:   otherType{
			Message: "originalMessage",
		},
		NestedPointer: &otherType{
			Message: "originalMessage",
		},
	}

	copiedValue := originalValue // this creates a shallow copy of original value
	copiedValue.A = -1
	copiedValue.B = -2
	copiedValue.NestedValue.Message = "modified message"
	copiedValue.NestedPointer.Message = "modified message"

	bytes, _ := json.Marshal(&originalValue)
	fmt.Print(string(bytes))
	// Output: {"A":1,"B":2,"NestedValue":{"Message":"originalMessage"},"NestedPointer":{"Message":"modified message"}}
}
```

The originalValue variable is stored in a chunk of memory which contains the variables A (int32), B (int64), NestedValue.Message (string) and NestedPointer (int64 / pointer). All value variables (even the variables of nested value variables) are stored in the chunk of memory. Variables in nested pointer variables are stored in a different chunk of memory which is referenced by the pointer.

If you assign the variable to another variable (`copiedValue := originalValue`) or pass the variable to a function (`myFunction(originalValue)`) the chunk of memory that is used by the variable is copied. The chunk of memory contains the pointer / int64 / memory address of the NestedPointer variable. The pointer is copied which means that originalValue.NestedPointer and copied.Value.NestedPointer point to the same memory address. 

### 2.4 Pointer and Value Receiver

I won't explain what pointer and value receivers are because the FAQ provides an excellent explanation: [Should I define methods on values or pointers?](https://go.dev/doc/faq#methods_on_values_or_pointers)

The most important take-away-message from this FAQ entry: If you want to modify the receiver you have to use a pointer receiver. If one method uses a pointer receiver all methods of that type should use pointer receivers for the sake of consistency. That's why almost all functions use a pointer receiver.

## 3. Types and functions / methods

Go differs strongly from other languages like TypeScript or Java which have classes. A class defines a strict binding between the functions / methods and the data / class attributes : Both things are defined by the class.

A type in Go is usually defined like this
```go
type TypeName struct{
	stringVar string
}
```

This means "I want to define a `type` which I want to name `TypeName`. The underlying data structure is a new struct which looks like this: ..." But we can also reuse existing data structures to create a new type:

```go
// https://go.dev/play/p/ujSdNAXPHvO

package main

import "fmt"

type Counter int // counter is just a normal int with an additional method

func (c *Counter) Increment() {
	*c++
}

func main() {
	intVar := 1

	counterVar := Counter(intVar) // This is a type conversion
	counterVar.Increment()
	fmt.Println(counterVar) // prints 2

	// c + 2 // This is a compilation error because Counter can't be used for math
	sum := int(counterVar) + 2 // we can convert back and forth without data loss
	fmt.Println(sum)           // prints 4

	//counterVar = intVar // This doesn't work. Go requires an explicit conversion
	counterVar = Counter(intVar)
	counterVar = 123 // Literals are converted implicitly
	counterVar.Increment()
	fmt.Println(counterVar) // prints 124
}
```

We can use a type conversion to convert between different types that share the same underlying data structure. You would use the same syntax to convert e.g. numbers (`float(1)`). Don't confuse a type conversion with casting in other languages! A type conversion just adds / removes functions to an underlying data structure.

Go doesn't support inheritance and therefore it's not possible / necessary to cast something up or down (e.g. String -> Object or Object -> String in Java). But Go has interfaces which can more or less do the same thing (e.g. string -> interface{} or interface{} -> string). Type conversion can't help you here, because type conversion is used to add / remove functions to a known type. The whole point of interfaces is that you don't need to know which type you're working with; you're only interested in the set of functions it provides.

You can use a type assertion if you have an interface and would like to extract the real type. Type assertions are covered in the [Tour of Go, Section 15](https://go.dev/tour/methods/15). [Type switches](https://go.dev/tour/methods/16) are also a very nice feature if you want to check for several types.

## 4. Constructors

This is a very short section. Go doesn't have constructors. (Why not!?!)

You can instantiate every struct that you have access to with a struct literal. However, this doesn't mean that you should do this because a lot of structs expect to be instantiated by their "constructor".

Here's an example:

```go
package foobarLibrary

type MyDataService struct {
  db OpenDatabaseConnection
}

func NewMyDataService(databaseURL URL, username string, password string) *MyDataService {
  db := myDbDriver.Open(databaseURL, username, password)
  return MyDataService{
    db: db,
  }
}
```

The MyDataService requires an open database connection. The creator of the library doesn't want us to play around with the internals of his library and therefore decided to keep the db variable private. We can not access it. We also can't specify it in the struct literal. We can write `foobarLibrary.MyDataService{}` but that's it. The db field is empty and we would get very interesting error messages if we tried to use some methods on this struct.

The creators of the library expect that their users use the constructor to crate a new instance of MyDataService. The constructor initializes all fields (even the unexported ones) and performs the setup logic. By convention the constructor name is equal to the type name with a prefixed "New" (type: MyDataService -> constructor: NewMyDataService()). It also makes sense to add a private constructor to a private / unexported type if some setup logic is required.

If you're dealing with a type that was created by someone else you should always check for a constructor before you're using a struct literal. Only use a struct literal if you're sure that this is the desired way to create a new instance.

If you're writing a new constructor keep in mind that the return type (pointer vs. value) matters. If you're returning a pointer your type will be used as pointer, if you return a value it will be used as value.

```go
myDataService := NewMyDataService()
otherService := NewOtherService(myDataService)
someFunction(myDataService)
```

If NewMyDataService returns a pointer the myDataService variable contains a pointer. This pointer will then be passed to the constructor of otherService and the pointer will also be passed to someFunction. Everyone operates on the same instance. If you return a value the myDataService variable contains a value and the NewOtherService constructor and someFunction function will be called with a _copy_ of myDataService. This behavior is described in section "2.3 Copying variables". In theory the user can convert the return type from pointer to value and vice versa. However, you can drastically simplify the lives of future developers (which most likely includes you) by returning the correct variable type.

Take away lessons: Always use the constructor if one is present. If you need some setup logic for your own type then you should write a constructor. If you want copies of your type the constructor should return values. If you don't want copies you should return pointers. 

## 5. Error handling

[Effective Go](https://go.dev/doc/effective_go#errors) contains a section on error handling which should be read. [The Uber Style](https://github.com/uber-go/guide/blob/master/style.md#errors) Guide also adds a few guidelines which are probably even more important. I especially want to highlight Uber's recommendation to [wrap almost all errors](https://github.com/uber-go/guide/blob/master/style.md#error-wrapping). This is important because 1) it creates better error messages and 2) Go's native errors don't have stacktraces.

In my opinion error handling should always happen in an `if` clause which is placed directly below the function which produced the error. The error handling usually is to return the _wraped_ error.

This approach has a few advantages:

1. It improves readability. The reader can ignore the if clauses when skimming the code because he knows that they just contain error handling code.
2. It's clear where the error originated.
3. Placing `if ... {return ...}` below every error source makes sure that we don't forget to handle the error / abort the function.

Here's an example:

```go
// This snippet is an example for bad code; don't do this

err := someFunction()
if err != nil { // this code can be simplified; The next code snippet explains how.
	return err // the error handling is in the if clause where it belongs
}

if result, err := otherFunction(); err == nil {
	handleResult(result) // the normal logic is handled in the if clause
}

// At this point the error can be nil or non nil; If a future developer adds any 
// code here the new code will silently ignore the error.
// doStuff(result) // <- This code will be run even if otherFunction encounters an
// internal error and "result" is invalid.

// Where does this error come from?
// Is this error nil or non-nil? If it's non-nil we should wrap it.
return err 
```

```go
// This snippet is an example for good code

if err := someFunction(); err != nil {
	return fmt.Errorf("someFunction: %w", err) // always wrap errors
}

result, err := otherFunction()
if err != nil {
	return fmt.Errorf("otherFunction: %w", err)
}

handleResult(result)

// doStuff(result)

return nil
```

The advice I gave before is definitely true for larger functions with multiple error sources. For very simple functions like the ones below it's a matter of personal preference. I still prefer the long variant for the sake of consistency (functionB) but one can also argue in favor of the short variant (functionA) because it's shorter. 

```go
def functionA() (string, error) {
	// this example could even be simplified to: return someFunc()
	result, err := someFunc()
	
	// this should only be done if you don't have to wrap the error
	return result, err
}

def functionB() (string, error) {
	result, err := someFunc()
	if err != nil {
		// errors should almost always be wrapped
		return "", fmt.Errorf("someFunction: %w", err)
	}
	
	return result, nil
}
```

## 6. File and package structure

There are [some conventions](https://github.com/golang-standards/project-layout) on how to structure the root folder of your repository. I especially want to highlight the `internal` folder which has a hidden feature:

```text
Quoting: https://go.dev/s/go14internal

An import of a path containing the element “internal” is disallowed if the
importing code is outside the tree rooted at the parent of the “internal” directory.
```

This is especially useful for libraries. If most of the codebase is stored in the internal folder the owners of the library can be sure that no one else uses it. This means that everything can be refactored without introducing breaking changes.

[Effective Go](https://go.dev/doc/effective_go#package-names) gives some advice on how packages should be named.

I couldn't find any hard and fast rules on how to structure go code into packages. I therefore had to come up with my own rules which I'm presenting now.

_Every type should have its own file._

This rule should be obvious and is even enforced by languages like Java. If you define a type `foo` you should only do so in a file which is called `foo.go`. All methods of foo should be defined in this file (and not somewhere else in the package). If you need unit tests for the code in `foo.go` you should store them in `foo_test.go`.

_Exported functions and constants should be stored in a file which is named after the package._

This rule is only applicable if the package exports global functions or constants (e.g. `mypackage.SomeFunction()`). This function should be stored in `mypackage.go`. Tests should be stored in `mypackage_test.go`. The intention of this rule is to improve the readability. If the package contains several files and you're looking for the implementation of a function you know where to find it.

There are exceptions to the rule. For example, ff you're defining a lot of constants (e.g. HTTP_STATUS_CODE_OK=200, HTTP_STATUS_CODE_NOT_FOUND=404, ...) it could make sense to create a new file for them.

_Go packages are way smaller than packages in other languages (e.g. Java)._

Go doesn't have private members; everything that's not exported can still be accessed and modified by code in the same package. If you create a type with a lot of internal variables which should not be modified by someone else it could make sense to create a new package just for that type. In general, I would recommend to only keep closely related types and functions in the same package. 

Go also stores the tests in the same directory as the source code. Having 5 files and their associated tests in the same package results in 10 files inside a single folder. In my opinion that's too much and the package should be broken down into smaller packages.

The last reason for smaller packages is the prevention of circular imports. This problem is explained in the next section.

_Organizing packages_

If the project contains multiple packages it becomes more complicated to organize them in a reasonable way. It's important to know that Go [doesn't support circular imports](https://go.dev/ref/spec#Import_declarations) (directly or indirectly) between packages.

Let's assume that we have this folder / package structure:

```text
a
 a1
    a11
 a2
    a22
 a3
b
```

Each folder describes a new package. We're assuming that every package contains some code.

It's important to highlight that Go has no concept for nesting packages. There is no relationship whatsoever between a, a1 and a2. The compiler is only interested in weather something is exported (every other package can use it) or not. But we have to keep in mind that circular imports are not allowed.

To structure the code and to avoid circular imports I came up with these rules:

1. If a package (e.g. `a`) has to make use of another package (e.g. `a1`) the imported package (`a1`) should be in a subfolder. This rule ensures that the purpose of the code is reflected in the file structure.
2. Modules are not allowed to use the code in another packages' subfolder. `a` is allowed to import `a1`, but `a` is not allowed to import `a11`. `b` is not allowed to import `a1`. This rule makes sure that every package is only responsible for its specific job and doesn't depend on the internal implementation of another package.
3. Code in a nested package (`a1`) must not make use of code in its parent package (`a`). This rule avoids circular imports.
4. It is discouraged (but sometimes necessary) to import code from a higher level in the file tree. `a1` must not import `a` (this would create a circular import), but `a1` is allowed to import `b`. Circular imports aren't possible because `b` doesn't use the internal / nested code of another package (`a`). A possible use case for this scenario is logging. If package `b` contains a global logger every other package has to import it to make use of it.
5. If a package (`a3`) is needed by multiple packages (`a1` and `a2`) it should be placed in the deepest folder that both packages have in common. This rule also applies if the two packages don't share the same folder (`a11` and `a22` need `a3`).
6. In case of a circular import you have to create a new package. Rule 5 determines where the new package should be stored. 

## 7. Enums

Go doesn't have enums (why not?!). But Go does have [Iota](https://go.dev/ref/spec#Iota) (I still don't understand why they added it). I recommend to read the specification for iota before going on.

Here's an example for a naive iota based enum implementation that can often be found on Stackoverflow:

```go
// https://go.dev/play/p/1kYehaZH0nJ

package main

import (
  "errors"
  "fmt"
)

type Fruit int

const (
  FruitApple  Fruit = iota + 1 // We need to specify that FruitApple is a Fruit; otherwise the type will be int 
  FruitBanana       // this is the "implicit repetition" that was mentioned in the specification
  FruitStrawberry
)

func MakeSmoothie(someFruit Fruit) {}

func GetFruitOfTheDay() (Fruit, error) {
  // I'm omitting the actual implementation. This is just the return statement for the error handling

  // Returning a literal value together with an error is a good way to indicate that we don't care about the result
  // It's visually very similar to returning nil.
  return 0, errors.New("Internal error")
}

func main() {
  fmt.Println(FruitApple)      // prints 1
  fmt.Println(FruitBanana)     // prints 2
  fmt.Println(FruitStrawberry) // prints 3

  var intVar int
  _ = intVar // without this line the compiler would generate an error because intVar is not used
  MakeSmoothie(FruitBanana)
  MakeSmoothie(0) // do not do this!
  //MakeSmoothie(intVar) // compiler error: Cannot use 'intVar' (type int) as the type Fruit
}
```

A few things to highlight:

1. We created a new type Fruit for the enum which uses an int as the underlying data structure. This has been covered in chapter "2.4 Pointer and Value Receiver". We need this type to make our code more readable and safer. The MakeSmoothie function requires a Fruit (and doesn't accept ints). The compiler makes sure that only Fruits are accepted.
2. Unfortunately enums are no builtin feature. We therefore create a big `const(...)` clause right below the type definition and added all possible values. Every value is prefixed with the name of the type to group them together. If you just type "Fruit" your IDE's autocompletion feature will present you all possible values.
3. The enum starts at `iota + 1` which means that `0` is no valid enum value. This is helpful for uninitialized enums (the default value of int is 0). If the enum has a useful default you can just write `defaultValue = iota`. The default value is then equal to `0`. Keep in mind that changing the default value is a potential breaking change.
4. The compiler implicitly converts a literal number to a Fruit. In general you should never use the numeric value of an enum because the numbers will change if someone adds / removes a new enum value. However, if you're returning an error the returned Fruit will always be ignored. Returning `0` is therefore OK and visually very similar to returning `nil`.

But there is a problem: Technically the Fruit is still an integer number. We can not print it! We could store it in a database and use it for our REST-API but that would mean to store numbers in the database and to put numbers in our JSON. Even worse: If someone adds a new enum value (or changes the order of the existing ones) the numbers will change!

We can avoid this problem by using strings as data structure:

```go
// https://go.dev/play/p/yzMyf0JSH31

package main

import (
	"encoding/json"
	"fmt"
)

type Fruit string

const (
	FruitApple Fruit = "apple"
	FruitBanana Fruit = "banana"
	FruitStrawberry Fruit = "strawberry"
)

type RequestBody struct {
	FavoriteFruit Fruit
}

func main() {
	var requestString = []byte(`{"FavoriteFruit": "someUnexpectedValue"}`)
	var requestBody RequestBody
	json.Unmarshal(requestString, &requestBody)
	fmt.Println(requestBody.FavoriteFruit) // prints someUnexpectedValue
}
```

The problem with strings is that the JSON parser can't verify strings and invalid values won't be rejected.

I finally found [enumer](https://github.com/alvaroloes/enumer) which generates the required code (conversion to and from string, value validation, etc.).

```go
// https://go.dev/play/p/_9PiqCQxeIu

package main

import (
	"encoding/json"
	"fmt"
)

//go:generate enumer -trimprefix Fruit -json -type Fruit enumer.go
type Fruit int

const (
	FruitApple Fruit = iota + 1
	FruitBanana
	FruitStrawberry
)

type RequestBody struct {
	FavoriteFruit Fruit
}

func main() {
	fmt.Println(FruitApple)      // prints Apple
	fmt.Println(FruitBanana)     // prints Banana
	fmt.Println(FruitStrawberry) // prints Strawberry

	request1 := RequestBody{FavoriteFruit: FruitStrawberry}
	marshaled1, _ := json.Marshal(request1)
	fmt.Println(string(marshaled1)) // prints {"FavoriteFruit":"Strawberry"}

	invalidRequestRaw := `{"FavoriteFruit":"someUnexpectedValue"}`
	var invalidRequest RequestBody
	err := json.Unmarshal([]byte(invalidRequestRaw), &invalidRequest)
	fmt.Println(err) // prints "someUnexpectedValue does not belong to Fruit values"
}

///////////////// This code was automatically generated by enumer /////////////////

const _FruitName = "AppleBananaStrawberry"

var _FruitIndex = [...]uint8{0, 5, 11, 21}

func (i Fruit) String() string {
	i -= 1
	if i < 0 || i >= Fruit(len(_FruitIndex)-1) {
		return fmt.Sprintf("Fruit(%d)", i+1)
	}
	return _FruitName[_FruitIndex[i]:_FruitIndex[i+1]]
}

var _FruitNameToValueMap = map[string]Fruit{
	_FruitName[0:5]:   1,
	_FruitName[5:11]:  2,
	_FruitName[11:21]: 3,
}

// FruitString retrieves an enum value from the enum constants string name.
// Throws an error if the param is not part of the enum.
func FruitString(s string) (Fruit, error) {
	if val, ok := _FruitNameToValueMap[s]; ok {
		return val, nil
	}
	return 0, fmt.Errorf("%s does not belong to Fruit values", s)
}

// MarshalJSON implements the json.Marshaler interface for Fruit
func (i Fruit) MarshalJSON() ([]byte, error) {
	return json.Marshal(i.String())
}

// UnmarshalJSON implements the json.Unmarshaler interface for Fruit
func (i *Fruit) UnmarshalJSON(data []byte) error {
	var s string
	if err := json.Unmarshal(data, &s); err != nil {
		return fmt.Errorf("Fruit should be a string, got %s", data)
	}

	var err error
	*i, err = FruitString(s)
	return err
}
```

## 8. Modules and dependencies

Modules are extensively described in the [documentation](https://go.dev/doc/#developing-modules) and the [module reference](https://go.dev/ref/mod). I only highlight the two things which might be surprising.

If you want to use v1.1.0 of a library and you're using another library which has a dependency on v1.2.0 of that library the application will be compiled with version v1.2.0. This mechanism is called [Minimal version selection (MVS)](https://go.dev/ref/mod#minimal-version-selection) and is mandatory. You _can not_ downgrade to an older version of a library if one of your (transitive) dependencies requires a newer version. This is one reason why semantic versioning is mandatory (and very important) when it comes to Go modules.

If the Go compiler (or any other Go tool like `go get`, `go mod tidy`, `go run`, etc.) realizes that the version number in your `go.mod` doesn't match the version that was computed by MVS it automatically adjusts the version in the `go.mod` file. 

MVS would introduce some problems if you have different major version of the same library / module in your (transitive) dependencies. The compiler therefore treats different major versions of a module as two different (and completely unrelated) modules. This means that different major versions of the same library can coexist in the same application ([source](https://go.dev/ref/mod#major-version-suffixes)). Obviously, this can only work if SemVer has been applied correctly and if every breaking change creates a new major version.

## 9. go generate

`go generate` is Go's code generation tool. [This blogpost](https://go.dev/blog/generate) does a great job at introducing it. But the article forgets to mention that you can run all `//go:generate` instructions in the code base by calling `go generate ./...` (or `go generate \...` on Windows) in the module's root folder.

## 10. Downloading modules from a private GitLab repository

**TLDR:**

run

```bash
go env -w GOPRIVATE=gitlab.mycompany.com
```

and create a `.netrc` file (`_netrc` on Windows) in your home folder which just contains this line:

```text
machine gitlab.mycompany.com login USERNAME password APITOKEN
```

Replace USERNAME and APITOKEN with valid credentials.

Go utilizes git to download the modules over https (not ssh). If git doesn’t already know the https credentials it will ask you for your user name and password.

**Technical background:**

By default go downloads modules from the global go-proxy and verifies the downloaded modules against the global checksum database. This is extensively described in the [official documentation](https://go.dev/ref/mod#private-modules) and can be disabled with `GOPRIVATE`.

Go modules are imported with URLs (e.g. `gitlab.mycompany.com/some/group/gomodule`) but go needs the git repository (e.g. `https://gitlab.mycompany.com/some/group/gomodule.git`) to download the source code. Go postfixes the import url with `?go-get=1` (e.g. `https://gitlab.mycompany.com/some/group/gomodule?go-get=1`) and the server returns the location of the git repository: `<meta name="go-import" content="gitlab.mycompany.com/some/group/gomodule git https://gitlab.mycompany.com/some/group/gomodule.git" />`. Go extracts the repository (`https://gitlab.mycompany.com/some/group/gomodule.git`) from the response and uses git to download the data.

This works fine as long as go tries to download a public gitlab repository. But if it's a private gitlab repository and go sends an unauthenticated request (because we didn't provide credentials) gitlab responds with `<meta name="go-import" content="gitlab.mycompany.com/some/group git https://gitlab.mycompany.com/some/group.git" />`. Note that the git repository is group.git and not gomodule.git! Go then tries to download group.git which fails because group is not a repository. This behavior also occurs if the `?go-get=1` endpoint is accessed with another kind of authentication (e.g. cookies in the browser). Gitlab's `?go-get=1` implementation only works correctly when the request uses basic auth. 

If you encounter a similar issue running `go get` with the `-v` flag can help during debugging. `-v` enables verbose logging which includes the http requests to GitLab.

These resources describe how to provide authentication using `.netrc`

- https://go.dev/doc/faq#git_https
- https://docs.gitlab.com/ee/user/packages/go_proxy/#enable-request-authentication
- https://docs.gitlab.com/ee/development/go_guide/dependencies.html#authenticating

Most resources that are available online modify the git configuration to download private repositories over ssh. However, this doesn't solve the authentication problem with the `?go-get=1` endpoint, we have no problem with downloading git over https and gitlab recommends the `.netrc` approach.

- https://stackoverflow.com/questions/29707689/how-to-use-go-with-a-private-gitlab-repo
- https://stackoverflow.com/questions/27500861/whats-the-proper-way-to-go-get-a-private-repository
- https://dev.to/0xbf/golang-go-get-package-from-your-private-repo-in-gitlab-f8m

## 11. Working with go binaries

This last advice is only helpful if you try to invoke a go binary inside a docker container and are getting a "no such file or directory" error. This might be a red herring!

Go is a statically compiled language, but Go can also call code that is written in C. If the C code was not statically compiled it tries to load the shared object files (they are called DLLs on Windows) at runtime. If it encounters a problem when loading the shared objects the problem is logged. In this case the operating system is telling us that the shared objects don't exist.

If you're using a precompiled binary that was provided by someone else you have to add the missing shared objects to the docker image. If it's your own code you can tell the Go compiler to compile the C code statically.

Helpful articles:

[Explanation on Stackoverflow](https://stackoverflow.com/questions/62632340/golang-linux-docker-error-standard-init-linux-go211-no-such-file-or-dir)

[Technical details and tips how to compile C code statically](https://www.arp242.net/static-go.html)