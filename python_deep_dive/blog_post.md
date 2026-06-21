# You're not a Python developer, you're an abstract C developer
A look under the hood of Python

![Gemini_Generated_Image_oe3aufoe3aufoe3a.png](Gemini_Generated_Image_oe3aufoe3aufoe3a.png)

## Disclaimer
This is not for Python beginners. You will learn some deep technical details...


## How does Python understand the code?
A short overview on how Programming languages parse the code.

### The lexer 
breaks down the source code into tokens, which are the smallest meaningful units of the language. 
For example, in Python, tokens include keywords, identifiers, operators, and literals.

The list of Tokens can be found within the Python source code `Grammar/Tokens`.

Here a few examples from this list
```Text
LPAR                    '('
RPAR                    ')'
LSQB                    '['
RSQB                    ']'
COLON                   ':'
COMMA                   ','
SEMI                    ';'
```

Some interresting findings within the `Parser/lexer/state.h` for the Lexer are that we can find the **maximal indentation level** which is **100**, 
the **maximal parentheses level** of **200** and the **maximal f-string nesting level** of **150**.

### The grammar
can be found within the Python source code `Grammar/python.gram` or in the online [Documentation](https://docs.python.org/3/reference/grammar.html). 

And looks like this
```Text
compound_stmt[stmt_ty]:
    | &('def' | '@' | 'async') function_def
    | &'if' if_stmt
    | &('class' | '@') class_def
    | &('with' | 'async') with_stmt
    | &('for' | 'async') for_stmt
    | &'try' try_stmt
    | &'while' while_stmt
    | match_stmt
```
everything is written in a grammar format where `&` is used to indicate a lookahead of predefined values which are surrounded by `" "` and `|` is used to indicate alternatives. 
Elements like `function_def` or `if_stmt` which are not surrouned with a `"` are other defined compounds and will be used to reduce redundancy, thing of it like a function you can call.

### The Parser
Mostly the parser will be generated from a different tool together with the predefined grammar, in the case of Python is a so called `PEG parser`.
The Python parser can be found in `Parser/parser.c`. 
Which contains statements like this
```c
// statement: compound_stmt | simple_stmts
static asdl_stmt_seq*
statement_rule(Parser *p)
{
    // more code
}
```

---

## Since the Python interpreter is written in C, let's take a quick look at the language before diving deeper.
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

> Often you will see that the return type is an `int` this is mostly to return a `success/failure` code.

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
Pointers are objects that hold the address of objects in memory (the heap). 
These memory objects can be a simple struct or a list then the pointer will point to the first element of the list.

The syntax of a pointer is `PyObject *obj` where `PyObject` is the type of the object and `obj` is the name of the pointer.
It is also possible to have a pointer to a pointer, the syntax of a pointer to a pointer is `PyObject **item`.

### What is the heap?
In Operating Systems, there are two regions where a program can ask for memory:
- The stack:
  - a notepad on your desk (small, fast, temporary)
  - this is often only a **small amount of memory**, where you can **access** data **fast**
  - allocation and deallocation of memory is done automatically by the Operating System
- The heap:
  - a filing cabinet (large, slower to navigate, but persistent)
  - this is the **largest** region of memory, **access** is to data is **slower** (because the Operating System needs to find a free chunk of memory)
  - the memory is more or less unstructured (programs often get chunks of memory and then allocate them to their needs),
  - allocation and deallocation of memory is done manually by the program

> In CPython, every Python object lives on the heap which is why you see `Py.. *` pointers everywhere in the source code.

For detailed information, check out this [article](https://courses.grainger.illinois.edu/cs225/sp2020/resources/stack-heap/#:~:text=stack%20%3A%20stores%20local%20variables,stores%20the%20code%20being%20executed) from the University of Illinois.

---

## What happens when we run a *.py file?
From a high-level view these are the steps what the Python interpreter does when we run a *.py file:

1. The Python interpreter reads the source code file.
2. Parsing the source code.
3. Execute the operations defined within the source code.

## How does the code run in the Python virtual machine?
the lifecycle of a Python object

## Add some interesting statements within the Python C API 
maybe why [] is faster than list(); 
or that the first 50 numbers already exist in the Python C API

---

## Why does this matter?
Learning basic C, and how memory is aligned within a Computer, can make you a better Python developer.
This can make your overall Python code more efficient cause you know how to handle memory, which can lead to a better performance.


## Thanks
This blog post wouldn't be possible without the book [Cpython Internals by Anthony Shaw](https://realpython.com/products/cpython-internals-book/).
