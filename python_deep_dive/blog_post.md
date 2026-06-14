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
### Available data types
### Operators
### Control flow, especially GOTO
### Pointers

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
