Author: Adam Albee (FuriousProgrammer@gmail.com)

# A Primer on Pointers

## I. Intro

For reasons I don't fully understand, many, many people have trouble understanding and conceptualizing **pointers** in programming languages that expose them directly.
This primer aims to break down what pointers are, why they exist, and how they are and can be used.
While the advice herein is applicable to *any* programming language that has pointers, all examples and explanations will revolve around the C language.

## II. Defintions

A very important disctinction needs to be made before we begin: a **variable** refers to the "box" that *holds* a **value**.
A value is, essentially, just a number.
Values are the real "data" the flows inside the CPU.

A **type** is two pieces of information: the *size* of a value (in bits or bytes) and the *encoding* of that value.
(E.g. a `float` is a 32-bit, IEEE 754 encoded Floating Point number, whereas a `char` is an 8-bit value that can either represent very small number (0 to 255 or -128 to 127) *or* an ASCII or UTF8 encoded "character.")
Variables care about the size, whereas the encoding informs the compiler which specific instructions to generate and which specific registers can be used.

A **pointer** is a *type* of variable that "points" to something by holding the *value* of the "address" of that something.

An **address** is a *value* that refers to some specific piece of hardware.
Most addresses &mdash; nearly all, even &mdash; refer to somewhere in RAM.
*Everything* in your computer has an address, however!
Some addresses can be read but not written to, some can be written to but not read, and some simply don't map to anything at all.

A pointer **degree** is the number of indirections a pointer has from its "base" type (E.g. a degree-2 `char**` is a pointer to a pointer to a `char`).

An **array** is a contiguous list of multiple values of a specific type.
In C, arrays are represented as a pointer to the first element in the array.
Contiguous in the context of memory means that the values are all stored at adjacent addresses &mdash; in other words, there are no "gaps" between the values.

A **vector** (or **matrix**) is an array that "knows" how many elements it has space for.
See [IV. Advanced Syntax] for more info about this.

A **buffer** is a container &mdash; typically a vector or array &mdash; for a process that cannot operate "one element at a time," or where it is more efficient to process elements *in bulk*.
Many people use "array" and "buffer" interchangably, especially when referring to strings of `char`s.

A **virtual address** is what your C programs are typically handling.
Virtual addresses refer to hardware addresses *indirectly*, with the OS mediating the translation of virtual<->hardware.
If you're writing an OS yourself and don't understand pointers already, God help you.

A **segfault**, or Segmentation Fault, is a kind of error that occurs when the CPU tries to access an address it isn't allowed to.
If you see SEGFAULT &mdash; you can immediately think "pointer error."

An **instruction**, or opcode (short for "operation code"), is a specific action that a CPU can do.

## III. Basic Syntax

There are three operators to understand when working with pointers: address-of `&`, indirection (aka dereference) `*`, and arrow-access `->`.

- Address-of gets the address of a variable &mdash; effectively increasing the "degree" of a pointer by 1.
- Indirection (Dereference) is the opposite &mdash; it accesses the *value pointed to* by the address a pointer is holding.
- Arrow-access notation is a shortcut for accessing members of a struct or union (or object in C++) via a pointer to that object without having to dereference it.
(E.g. `someStructPtr->member` is equivalent to `(*someStructPtr).member`).
Frankly speaking, this operator is completely redundant and really doesn't need to exist.

Additionally, array access notation is actually using pointers under the hood: `arr[i]` is a shortcut for `*(arr + i)`.
This is a very useful syntax, as nested (i.e. multidimensional) arrays can be quite unwieldy: `arr[z][y][x]` ==> `*(*(*(arr + z) + y) + x)`.

When written with the *type* of a variable, the `*` is not an operator, but rather a notation signaling the degree of the pointer type:

```c
char    value       = 'a';
char*   DegreeOne   = &value;
char**  DegreeTwo   = &DegreeOne; // or &&value
char*** DegreeThree = &DegreeTwo; // or &&DegreeOne or &&&value
```

If you come from C++, the `&` used for pass-by-reference in function parameters *does not exist* in C, but is similary not an operator.

## IV. Advanced Syntax

C uses what are known as "narrow pointers" &mdash; meaning that a pointer to a single variable and a pointer to an array of variables *look identical*.
When vectors are passed to (or returned from) functions, their references "decay" into pointers.
For all intents are purposes, all arrays *are* pointers and all pointers *are* arrays.

It's impossible to figure out from the type alone if a `char*` is a pointer to a single `char` or a string &mdash; an array of `char`s!

Going one step further: it's not possible to determine from the type or variable alone if a `char**` is:
- a pointer to a single pointer to a single `char`
- an array of pointers to single `char`s
- a pointer to a single string
- an array of strings

This distinction mainly comes up in practice because of a trick used to calculate the length of a vector:

```c
char vector[] = {'H', 'e', 'l', 'l', 'o', '!', '\0'};
size_t len = sizeof(vector)/sizeof(vector[0]); // 7/1 = 7
```

For *vectors*, this works because `sizeof(vector)` returns the total number of bytes used to store the vector.
However, the moment this vector is passed to a function, the trick stops working.
The only way to know the size of an array passed to a function is to also pass the length of the array to a parameter.

```c
char vector[] = {'H', 'e', 'l', 'l', 'o', '!', '\0'};

size_t getLen( char* array ) {
	return sizeof(array)/sizeof(array[0]);
}

size_t len = getLen( vector ); // 4/1 or 8/1 - 32-bit mode or 64-bit mode depending
```

You may recognize that *C Strings* are very oftened passed around *without* their length (E.g. `strlen`, `strcpy`, `strcmp`, etc.)!
This is because of a special formatting C Strings are given in the form of two restrictions: a valid C String (1) has no embedded `0`, aka `'\0'` aka `NULL` bytes and (2) has a `NULL` byte 1 past the end of the string itself.
This is an example of a **sentinel** value &mdash; the string *cannot* contain `0`s because a `0` *means the end of the string*.

Importantly, the length of the *string* and the length of the *`char*`* buffer are *different*: the buffer is always at *least* 1 `char` longer than the maximum length of the string it contain.
A "proper" C String specifically refers to `char*` *buffers* that hold a single string and are thus precisely as long as the length of that string plus 1, but this is a fairly unimportant restriction.
Later on, I will show an example of how this sentinel value handling can be used to embed *multiple* strings inside of a single contiguous `char*` buffer.

There is a *significant* difference between **array** access notation and **vector** access notation in terms of the actual instructions generated:

```c
size_t rows = 10;
size_t cols = 20;

// memory management and how to implement a function like make2DArray() will be covered latter.
char** array2D = make2DArray(rows, cols);
char vector2D[rows][cols];

size_t y = 5;
size_t x = 3;

array2D[y][x]++; // ==> *(*(array + 0) + 0) += 1;
vector2D[y][x]++; // ==> vector2D[y*cols + x] += 1;
```

This is, fundamentally, the difference between a **vector** (literally, a *vectorized* array!) and a "real" **array**.
Multidimensional *vectors* can be thought of as a 1-dimensional "flat" array and are necessarily rectangular (that is, all of the "inner" dimensions have the exact same lengths) and the entire contents are contiguous in memory.

Multidimensional *arrays* are a kind of recursive data structure: each dimension is a list of pointers with one fewer degree except for the innermost dimension, which is a 1D vector of the values at those coordinates.
None of these dimensions *have to* have the same lengths as any other, but in exchange the contents of the entire array can be spread far apart at different addresses with potentially massive gaps in between.

In some versions of C, you can do something like the following to inform the compiler that you've passed in a vector (and how big that vector is), but it you still have to pass the size directly for it to work.
Later on, I will show a struct-based "wide point" design that can alleviate a lot of this pain.

```c
void AFunction( size_t len, char vector[len] ) {
	// do some stuff
}
```

<!-- TODO: figure out the sytnax for multi-dimensional vectors + explain type-casting a "flat" array to a multidimension vector -->

## V. Regular Old Variables (The Stack)

It may come as a suprise to some, but (nearly) *all* variables use pointers under the hood! The CPU's registers are the only things that the CPU can *directly* perform computation on.
Because there are a severely limited number of CPU registers (literally *dozens* of bytes in total!), the Stack is used to store additional values.

Every variable in a program &mdash; local variables, function parameters, globals &mdash; ***everything*** &mdash; is in reality an **address** to their value stored on the Stack.
The compiler handles how these values are organized, and generates the instructions that handle copying to and from the CPU registers completely automatically.

A common mistake pointer-beginners often make is "returning a pointer to the Stack":

```c
#include <stdio.h>

typedef struct {
	int foo;
	int bar;
} thing;

thing* CreateAThingList() {
	thing NewThing[10];
	NewThing[0].foo = 12;
	NewThing[1].bar = -23;

	return NewThing;
}

int main() {
	thing* myThingList = CreateAThingList();

	printf("Hello, world, here's my thing.foo!\n");
	printf("%d\n", myThingList[0].foo); 

	return 0;
}
```

<!-- TODO: talk about Stack Framess -->
