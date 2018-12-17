# Golang Variables

## What is Variable in Golang

Variables are "containers" for storing information, like string of text, numbers, etc. Variable values can change over the execution of a program. Here're some important things to know about Golang variables:

- Golang is statically typed language, which means that when variables are declared, they either explicitly or implicitly assigned a type even before your program runs.
- Golang requires that every variable you declare inside main() get used somewhere in your program.
- You can assign new values to existing variables, but they need to be values of the same type.
- A variable declared within brace brackets {}, the opening curly brace { introduces a new scope that ends with a closing brace }.

------

### Declaring an integer and string variable

The assignment of a value inline with the initialization of the variable.

```
package main
 
import (
    "fmt"
)
 
func main() {
    var i int = 10
    var s string = "Japan"
    fmt.Println(i)
    fmt.Println(s)
}
```

A variable in Go is initialized by var keyword.
In above example variables are assigned by name i, s and type integer, string respectively.
The assignment operator = signifies that the variable should be assigned a value of whatever is to the right of = .
The fmt standard library package uses the variable name i and s as a reference to the value of i and s.

------

### Assignment after initialization

First a variable is initialized and then assigned a value later in program.

```
package main
 
import (
    "fmt"
)
 
func main() {
    var intVar int
    var strVar string
 
    intVar = 10
    strVar = "Australia"
 
    fmt.Println(intVar)
    fmt.Println(strVar)
}
```

------

### Short Variable Declaration

The := short variable assignment operator indicates that short variable declaration is being used.

```
package main
 
import (
	"fmt"
)
 
func main() {
	s := "Japan"
	fmt.Println(s)
}
```

------

### Scope of Variables Defined by Brace Brackets

Inner block can access its outer block defined variables, but outer block cannot access inner block defined variables.

```
package main
 
import (
	"fmt"
)
 
var s = "Japan"
 
func main() {
	fmt.Println(s)
	x := true
 
	if x {
		y := 1
		if x != false {
			fmt.Println(s)
			fmt.Println(x)
			fmt.Println(y)
		}
	}
	fmt.Println(x)
}
```

------

### Naming Conventions for Golang Variables

These are the following rules for naming a Golang variable:

- A name must begin with a letter, and can have any number of additional letters and numbers.
- A variable name cannot start with a number.
- A variable name cannot contain spaces.
- If the name of a variable begins with a lower-case letter, it can only be accessed within the current package this is considered as unexported variables.
- If the name of a variable begins with a capital letter, it can be accessed from packages outside the current package one this is considered as exported variables.
- If a name consists of multiple words, each word after the first should be capitalized like this: empName, EmpAddress, etc.
- Variable names are case-sensitive (car, Car and CAR are three different variables).