# You're not a Python developer, you're an abstract C developer
A look under the hood of Python

![Gemini_Generated_Image_oe3aufoe3aufoe3a.png](Gemini_Generated_Image_oe3aufoe3aufoe3a.png)

## Disclaimer
This is not for Python beginners you will learn some deep technical details on how Python works under the hood.


## What happens when we run a *.py file?
From a high-level view these are the steps what the Python interpreter does when we run a *.py file:

1. The Python interpreter reads the source code file.
2. Parsing the source code.
3. Execute the operations defined within the source code.

### How does an interpreter reads/parses the source code?
Every interpreter must tokenize the source code, like an LLM to create a meaning out of it. 
This is done by the **lexer**.

#### The lexer 
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

The list of Tokens will be feed into the Parser which extracts with the predefined grammar the "meaning" of the source code.

#### The Python grammar
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

With these definitions we can build the Parser

#### The Parser
generated from the predefined grammar can be found in `Parser/parser.c`. Which contains statements like this
```c
// statement: compound_stmt | simple_stmts
static asdl_stmt_seq*
statement_rule(Parser *p)
{
    // more code
}
```

### What happens after the Parser

And with all of these the Python interpreter can create the AST (Abstract Syntax Tree).
Which itself will be compiled into Python bytecode which then gets executed.

The compilation will be made under `Python/compile.c` there we can find statements like
```C
PyCodeObject *
_PyAST_Compile(mod_ty mod, PyObject *filename, PyCompilerFlags *pflags,
               int optimize, PyArena *arena)
{
    assert(!PyErr_Occurred());
    compiler *c = new_compiler(mod, filename, pflags, optimize, arena);
    if (c == NULL) {
        return NULL;
    }

    PyCodeObject *co = compiler_mod(c, mod);
    compiler_free(c);
    assert(co || PyErr_Occurred());
    return co;
}
```

or like 
```C
int
_PyCompile_AstPreprocess(mod_ty mod, PyObject *filename, PyCompilerFlags *cf,
                         int optimize, PyArena *arena, int no_const_folding)
{
    _PyFutureFeatures future;
    if (!_PyFuture_FromAST(mod, filename, &future)) {
        return -1;
    }
    int flags = future.ff_features | cf->cf_flags;
    if (optimize == -1) {
        optimize = _Py_GetConfig()->optimization_level;
    }
    if (!_PyAST_Preprocess(mod, arena, filename, optimize, flags, no_const_folding, 0)) {
        return -1;
    }
    return 0;
}
```
 

---
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
Pointers are objects that hold the address of objects in memory (the stack). 
These memory objects can be a simple struct or a list then the pointer will point to the first element of the list.

The syntax of a pointer is `PyObject *obj` where `PyObject` is the type of the object and `obj` is the name of the pointer.
It is also possible to have a pointer to a pointer, the syntax of a pointer to a pointer is `PyObject **item`.

---

## How does the code run in the Python virtual machine?
the lifecycle of a Python object

## Add some interesting statements within the Python C API 
maybe why [] is faster than list(); 
or that the first 50 numbers already exist in the Python C API

## Summary
Why you should learn C in a general way and how memory is aligned within a Computer to make you a better Python developer.


## Thanks
This blog post wouldn't be possible without the book [Cpython Internals by Anthony Shaw](https://realpython.com/products/cpython-internals-book/).
