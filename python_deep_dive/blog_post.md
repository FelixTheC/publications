# You're not a Python developer, you're an abstract C developer
A look under the hood of Python

![Gemini_Generated_Image_oe3aufoe3aufoe3a.png](Gemini_Generated_Image_oe3aufoe3aufoe3a.png)

## Disclaimer
This is not for Python beginners you will learn some deep technical details on how Python works under the hood.


## What happens when we run a *.py file?
From a high-level view these are the steps what the Python interpreter does when we run a *.py file:

1. The Python interpreter reads the source code file and converts it into an abstract syntax tree (AST).
2. The AST is then compiled into bytecode, which is a low-level representation of the code that can be executed by the Python virtual machine.
3. The bytecode is then executed by the Python virtual machine, which interprets the bytecode and executes the corresponding operations.

### How do we get the abstract syntax tree (AST)?

#### The Parser

#### The Lexer

#### Bringing it all together

## A short intro to the C-Programming language
C is a functional programming language and has no object-oriented features. 
It relies heavily on structs and on the concept of pointers and references to pass data around.

### Available data types
The C-Programming language comes with only a handful of data types:
- bool
- char
- short
- int
- long
- float
- double
and 2 modifier types:
- unsigned (example: `unsigned int`)
- signed (example: `signed int`)

### Functions
In C functions can only return one single value; this means we can't return multiple values like in Python or even a list.
If you want to return multiple values you pass a reference to the return value as a parameter, mostly the last parameter.

Simple function declaration:
```c
PyObject* PyObject_GetAttrString(PyObject *o, const char *attr_name) {
    // some code
}
```

Return value within a parameter
```c
void PyObject_SetAttrString(PyObject *o, const char *attr_name, PyObject *v) {
    // some code
}
```

### Control flow, especially GOTO
Besides the control flow statements like if, else, for, while, and switch, C also supports the goto statement, which can be used to jump to a specific label in the code.
Most modern languages doesn't support this statement anymore but in general it looks like this:
```c
if (condition)
    goto label;

// some code

label:
    // code
```

### Pointers
Pointers are objects that hold the address of objects in memory (the stack). 
These memory objects can be a simple struct or a list then the pointer will point to the first element of the list.

The syntax of a pointer is `PyObject *obj` where `PyObject` is the type of the object and `obj` is the name of the pointer.
It is also possible to have a pointer to a pointer, the syntax of a pointer to a pointer is `PyObject **item`.

## How do we get the bytecode?

## How does the code run in the Python virtual machine?
the lifecycle of a Python object

## Add some interesting statements within the Python C API 
maybe why [] is faster than list(); 
or that the first 50 numbers already exist in the Python C API

## Summary
Why you should learn C in a general way and how memory is aligned within a Computer to make you a better Python developer.


## Thanks
This blog post wouldn't be possible without the book [Cpython Internals by Anthony Shaw](https://realpython.com/products/cpython-internals-book/).
