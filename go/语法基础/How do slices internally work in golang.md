# How do slices internally work in golang

Why do slices exists if you already have arrays in golang? Let’s see why we need slices in golang.

So in this blog, we will talk about

1. What are slices and why do we need them?
2. How do they internally work in golang?
3. What mistakes can we make while using slices and how to avoid those mistake.

For understanding the importance of slices, first, let's see what are arrays in golang? You are familiar with arrays then you can directly jump to slice section.

### Arrays

Similar to `cpp` arrays in golang is the collection of the same type with continuous memory. (as you can access the elements directly through index).

Arrays in golang are declared as `[n]T` where `n` is the size(length) of the array and `T` is the type like `int` , `string` etc. Like:

```
var a [3]int
a[0] = 1
a[1] = 2
a[3] = 3
//or
a := [3]int{1, 2, 3}
```

You can even ignore the length of the array in the declaration of it. You can use `...` instead of size ( `n` ), like

```
a := [...]{1, 2, 3}
```

Here the compiler will find the length for you.

Arrays in golang are a value type, means whenever you assign an array to a new variable then the copy of the original array is assigned to the new variable. This is also called as `deep copy` .

```
a := [...]string{"Alice", "Bob", "Cop"}
b := a // a copy of a is assigned to b
b[0] = "Dytto"
fmt.Println("a is ", a)
fmt.Println("b is ", b)
```

The output of the above snippet will be

```
a is  [Alice Bob Cop]
b is  [Dytto Bob Cop]
```

The main issue an array has is that it can not be resized(which is a very generic requirement) as the length of the array is part of it’s type i.e. `[3]int`and `[5]int` are two different data types.

To overcome this problem we have `Slices` in golang.

### **Slices**

- **What are slices?**

In very simple words slices are the wrapper over arrays. Slices do not own any data of their own, they are just a reference to the existing array.

The syntax is like:

```
a[start:end] creates a slice of array a from index start to end-1 .
a := [5]int{1, 2, 3, 4, 5}
var b []int = a[1:4] //creates a slice from a[1] to a[3]
a := []int{1, 2, 3} //creates a array of 3 intergers and return the slice reference which is stored in a.
```

- **Why do we need slices?**

Very often we do have cases where we need to resize the length of the array. (like append an array into another array). But arrays in golang cannot be resized hence we have slices whose size can be dynamically changed.

Slices have length and capacity. **Length** is the number of elements present in the slice, **capacity** is the number of elements present in the underlying array(the array to which slice is referencing to) starting from the index from which the slice is created.

You can create a slice using `make` (like `make([]T, len, cap)` ) by passing length and capacity (capacity is an optional parameter and by default, it is equal to the length).

```
a := make([]int, 5, 10) // a is a slice of length 5 and capacity 10.
```

- **Modification of slices**

As I mentioned earlier slices do not own any data on their own, they are just a reference to existing arrays. Hence any modification done in the slice will reflect in the underlying array also. For example,

```go
readers := [4]string{"Alice", "Bob", "Charlie", "Dytto"}

a := mentors[0:2]
b := mentors[1:3]
fmt.Println("Before Modification")
fmt.Println("Readers", readers)
fmt.Println("a:" a, "b:" b)

b[0] = "Common"
fmt.Println("After Modification")
fmt.Println("Readers", readers)
fmt.Println("a:" a, "b:" b)
```

the output will be,

```go
Before Modification
Readers [Alice Bob Charlie Dytto]
a: [Alice Bob] b: [Bob Dytto]

After Modification
Readers [Alice Common Charlie Dytto]
a: [Alice Common] b: [Common Dytto]
```

New elements can be added to the slice using `append` function. Like

```
readers := []string{"Alice", "Bob", "Charlie"}
fmt.Println("Readers' array :", readers, "has old length", len(readers), "and capacity", cap(readers))
readers = append(readers, "Dytto")
fmt.Println("Readers' array :", readers, "has old length", len(readers), "and capacity", cap(readers))
```

the output will be

```
Readers' array : [Alice Bob Charlie] has old length 3 and capacity 3
Readers' array : [Alice Bob Charlie Dytto] has new length 4and capacity 6
```

Why the capacity changes to 6? The answer is in the next section.

- **Memory Allocation of slices**

This is the most important part of slices. If you do not know how slices’ size can be dynamically changed then sometimes your code might not behave as you expected. Before the explanation of it lets see an example.

```go
package main

import (
	"fmt"
)

func main() {
	//case 1
	a := []int{}
	a = append(a, 1)
	a = append(a, 2)
	b := append(a, 3)
	c := append(a, 4)
	fmt.Println("a: ", a, "\nb: ", b, "\nc: ", c)
	
	//case 2
	a = append(a, 3)
	x := append(a, 4)
	y := append(a, 5)
	fmt.Println("a: ", a, "\nx: ", x, "\ny: ", y)
	
}
```

most of the people will think that the output will be

```
a:  [1 2] 
b:  [1 2 3] 
c:  [1 2 4]
a:  [1 2 3] 
x:  [1 2 3 4] 
y:  [1 2 3 5]
```

But the output will be

```
a:  [1 2] 
b:  [1 2 3] 
c:  [1 2 4]
a:  [1 2 3] 
x:  [1 2 3 5] 
y:  [1 2 3 5]
```

And the later one is correct. I will explain it to you.

**When the slice length is equal to its capacity and you try to append a new element to it, then the new memory is allocated to the slice which has the double capacity as compared to the earlier capacity.**

- So in case 1 at line 10 `a` has capacity and length both equal to 1,
- In line 11 `a` gets allocated to a new address with capacity 2 (double the earlier one) and its length is also 2. (as two elements are present).
- In line 12 since `a` has capacity = length hence line `appned(a, 3)` create a new array with length 4 (double the capacity of earlier one) and returns the slice reference to `b`. Now b is slice which has length = 3 and capacity =4. Here the important point is that `b` is not referencing to the underlying array of `a` , it is referencing to a new array which has length =4. `a` still has capacity and length =2.
- In line 13 same thing is happening as line 12, since `a` still has the length and capacity = 2, line `append(a, 4)` will create a new array with length 4 (double the capacity of `a` as `a`’s `len(a)` = `cap(a)`) return the slice reference to `c` . Now `c` is also a slice with len = 3 and capacity =4. And same as line 13 `c` is not referencing to the underlying array of `a` , it is referencing to a new array which has length = 4. `a` still has capacity and length = 2.



![img](/img/1_PiSPX0YMBjkd1fSnWlHURA.png)

Case 1

- Before line 17 `a`’s length and capacity both are 2. But in line 17 `a`’s capacity has been changed as line `a = append(a, 3)` creates a new slice with length 4 (double to the `a`’s last capacity) and return a slice reference to `a` only. So after line 17 `a` has length = 3 (elements are 1, 2 and 3) and capacity =4.
- Hence in line 18 `x := append(a, 4)` will append 4 to `a` (as `a` has capacity = 4 and has currently only 3 elements) and return the reference (same as `a` referencing to) to `x`.
- In line 19 same thing is happening as line 18. `y := append(a, 5)` will append 5 to `a` (as `a` has capacity = 4 and has currently 3 element) and return the reference (same as `a` referencing to ) to y.



![img](/img/1_taMU2CaEi3FmGOJhMe9quQ.png)

Case 2 (This is the scene after line 19)

So interesting fact is that in case 2 `a,` `x`and `y`are referencing to the same array hence changes done by `y:= append(a, 5)` is being reflected in `x` also.

That is why the output is printing same elements for `x` and `y` .

So whenever you are working with slice and using append beware of such scenarios.