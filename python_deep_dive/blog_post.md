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
- the entry point of the interpreter is the `main()` function inside of `Programs/python.c`
  - interesting is that we run in a slightly different way when running on Windows compared to Linux/macOS
```c
#ifdef MS_WINDOWS
    int
    wmain(int argc, wchar_t **argv)
    {
        return Py_Main(argc, argv);
    }
#else
    int
    main(int argc, char **argv)
    {
        return Py_BytesMain(argc, argv);
    }
#endif
```
- from there we're coming to the function `pymain_run_python` inside of `Python/main.c`
  - here the interpreter checks if he runs in a REPL or not
```c
    if (config->run_command) {
        *exitcode = pymain_run_command(config->run_command);
    }
    else if (config->run_module) {
        *exitcode = pymain_run_module(config->run_module, 1);
    }
    else if (main_importer_path != NULL) {
        *exitcode = pymain_run_module(L"__main__", 0);
    }
    else if (config->run_filename != NULL) {
        *exitcode = pymain_run_file(config);
    }
    else {
        *exitcode = pymain_run_stdin(config);
    }
```
- then the interpreter creates a `PyObject *` file object and passing it downwards (with a few checks) to `_PyRun_SimpleFileObject`
  - here the interpreter checks if he can use a fast route via `maybe_pyc_file`, when we already have a `pyc` a Python bytecode file, we can skip the compilation step and directly execute the bytecode.
    - this is the reason why the second run of the same file is faster than the first run.
  - after this we will call `pyrun_file` inside of `Python/pythonrun.c`
  - there the interpreter first creates a new `PyArena` object, this will be used to handle memory for the bytecode in one single place
  - this arena will be first used to create the AST from the file (source code) `_PyParser_ASTFromFile` from `Parser/peg_api.c`
- then a Tokenizer for the source code `_PyTokenizer_FromFile` from `Parser/tokenizer/file_tokenizer.c` and a Parser `_PyPegen_Parser_New` from `Parser/pegen/pegen.c` will be created
  - a look into the `parser` struct
    ```c
    typedef struct {
        struct tok_state *tok;
        Token **tokens;  // this is how C handles arrays a pointer to an pointer
        int mark;
        int fill, size;
        PyArena *arena;  // here lives the memory for the bytecode
        KeywordToken **keywords;
        char **soft_keywords;
        int n_keyword_lists;
        int start_rule;
        int *errcode;
        int parsing_started;
        PyObject* normalize;
        int starting_lineno;
        int starting_col_offset;
        int error_indicator;
        int flags;
        int feature_version;
        growable_comment_array type_ignore_comments;
        Token *known_err_token;
        int level;
        int call_invalid_rules;
        int debug;
        location last_stmt_location;
    } Parser;
    ```
  - the `Parser` is the initialized by setting up the `keywords` and `soft_keywords` arrays,
  ```c
      void *
      _PyPegen_parse(Parser *p)
      {
          // Initialize keywords
          p->keywords = reserved_keywords;
          p->n_keyword_lists = n_keyword_lists;
          p->soft_keywords = soft_keywords;
        
          // Run parser
          void *result = NULL;
          if (p->start_rule == Py_file_input) {
              result = file_rule(p);
          } else if (p->start_rule == Py_single_input) {
              result = interactive_rule(p);
          } else if (p->start_rule == Py_eval_input) {
              result = eval_rule(p);
          } else if (p->start_rule == Py_func_type_input) {
              result = func_type_rule(p);
          }
        
          return result;
      }
  ```
  - in the next step we can also figure out the max stack size/depth of the Python interpreter `define MAXSTACK 6000`
  - after everything is parsed we must clean everything up and free the memory `_PyPegen_Parser_Free(p);`
  - when everything is parsed we create the `_PyAST_Module` within `Parser/action_helpers.c` and execute the statements within the module via `run_mod` from `Python/pythonrun.c`
  - then the interpreter starts to compile the AST and return a `PyCodeObject *`, `PyCodeObject *co = _PyAST_Compile(mod, interactive_filename, flags, -1, arena);`
    - compiling the AST takes several steps, get compiler flags, get futures from the AST, optimize the AST, preprocess it and create a symbol table
    - when the AST gets folded with optimization, it removes also the docstrings
    ```c
    if (docstring && (state->optimize >= 2)) {
        /* remove the docstring */
        if (!remove_docstring(stmts, 0, ctx_)) {
            return 0;
        }
        docstring = 0;
    }
    ```
  - then a `PyCodeObject *` is created within `optimize_and_assemble_code_unit` from `Python/compile.c`

## Add some interesting statements within the Python C API 
maybe why [] is faster than list(); 
or that the first 50 numbers already exist in the Python C API

- Even within Cpython you can find such code comments like these
```c 
-- Objects/unicodeobject.c
   
// I don't know this check is necessary or not. But there is a test
// case that requires size=PY_SSIZE_T_MAX cause MemoryError.
if (PY_SSIZE_T_MAX - sizeof(PyCompactUnicodeObject) < (size_t)size) {
    PyErr_NoMemory();
    return NULL;
}
```

- Why do we don't need to write `return None` at the end of a function?
```c
int
_PyCodegen_AddReturnAtEnd(compiler *c, int addNone)
{
    /* Make sure every instruction stream that falls off the end returns None.
     * This also ensures that no jump target offsets are out of bounds.
     */
    if (addNone) {
        ADDOP_LOAD_CONST(c, NO_LOCATION, Py_None);
    }
    ADDOP(c, NO_LOCATION, RETURN_VALUE);
    return SUCCESS;
}
```
- `ADDOP_LOAD_CONST` is a macro that adds a LOAD_CONST opcode to the bytecode stream. This opcode loads a constant value onto the stack. In this case, it loads the value `Py_None` onto the stack.

---

## Why does this matter?
Learning basic C, and how memory is aligned within a Computer, can make you a better Python developer.
Your Python code can become more efficient because you know how the underlying memory is handled and moves between functions, which can lead to a better performance and less bugs.


## Thanks
This blog post wouldn't be possible without the book [Cpython Internals by Anthony Shaw](https://realpython.com/products/cpython-internals-book/).
