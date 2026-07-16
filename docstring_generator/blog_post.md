# Who Keeps the Docstrings Accurate after an AI Assistant creates it initially?
 
### A Python CLI library with a fast C++ engine will keep them all the time aligned with your code.

---

![A developer working with AI-assisted documentation](Gemini_Generated_Image_iu4ui4iu4ui4iu4u.png)

Most Python developers use an AI assistant like Cursor, Copilot, or Claude Code these days. These tools can inspect a function and write a clean docstring in seconds.

But passive AI documentation has a major flaw: **context drift**.

An AI generates a great description the first time, but it won't automatically update that description when someone changes the function three weeks later. 
If a parameter becomes optional, a return type changes, or a new exception is raised, the old docstring stays exactly as it was. The documentation becomes out of date and misleading.

I built [`docstring_generator`](https://github.com/FelixTheC/docstring_generator) to solve this. 
It reads Python signatures and type hints, generates structured docstrings, and updates the mechanical parts when the code changes. 
It does not replace the helpful prose written by a developer or an AI — it just keeps that prose tied to a structure that matches the actual code.

---

## What the Library Does

At its simplest, `docstring_generator` can process a single Python file or an entire directory:

```shell
pip install docstring-generator

gendocs_new service.py
gendocs_new src/
```

It handles standalone functions and class methods (both `def` and `async def`). Pulls parameters and return information from type annotations and default values, automatically omitting `self` and `cls`.

Three common formats are supported:

- NumPy (default)
- Google
- reStructuredText (for Sphinx)

For example:

```python
async def fetch_user(user_id: int, include_profile: bool = False) -> dict:
    ...
```

gets a consistently structured docstring without anyone copying the signature into prose by hand.
 
If you have a mixed codebase, it detects the existing style and won't overwrite it silently. If you want to force a change, you can do it explicitly:

```shell
gendocs_new src/ --style google --overwrite-style true
```

That makes style conversion an explicit decision rather than a surprising side effect.

---

## Why Not Ask an LLM to Do All of This?

LLMs and deterministic tools solve different parts of the documentation problem.

### The token and latency tax

Sending a large repository to a model on every commit just to verify that signatures and docstrings agree is slow and unnecessarily expensive. Parsing code locally is a much better fit for repetitive structural work.

### Probabilistic output

An LLM may infer that a function can fail. A static analyzer can inspect its `raise` statements. For API contracts, that distinction matters.

### Loss of domain context

Regenerating a complete docstring can erase the explanation that only a person familiar with the business domain could write. Structural synchronization should not require throwing away useful prose.

### Governance and code privacy

Sending source code to an external model may be restricted—or prohibited entirely—for repositories that contain proprietary logic, security-sensitive code, or regulated data. Even when an AI provider offers suitable privacy controls, teams still need to decide which code may leave their environment, how prompts and outputs are retained, and who is accountable for reviewing generated documentation.

`docstring_generator` performs its analysis locally and does not need to send source code to a model. Its output follows explicit rules that can be versioned in `pyproject.toml`, reviewed in a diff, and enforced consistently in CI. That makes the documentation process easier to audit and gives security and compliance teams a clear answer to what ran, where it ran, and which policy it applied.

The productive division of labor is straightforward: let AI or a developer explain *why* a parameter exists, and let deterministic tooling maintain names, types, return sections, formatting, and detectable exceptions.

---

## Keeping Your Explanations

You can use `$1`, `$2`, and `>>` markers to pin specific descriptions to parameters and returns:

```python
def calculate_total(items: list[float], tax_rate: float) -> float:
    """Calculate the final invoice amount.

    $1 Prices before tax.
    $2 Tax expressed as a decimal, for example 0.19.
    >> The invoice total including tax.
    """
    return sum(items) * (1 + tax_rate)
```

With Google style selected, the result will become this:

```python
def calculate_total(items: list[float], tax_rate: float) -> float:
    """Calculate the final invoice amount.

    Args:
        items (list[float]): Prices before tax.
        tax_rate (float): Tax expressed as a decimal, for example 0.19.
    Returns:
        float: The invoice total including tax.
    """
    return sum(items) * (1 + tax_rate)
```

This is where AI fits well: use it to draft meaningful descriptions, then use the generator to place them into a predictable format and keep the surrounding structure consistent.

---

## Documenting Exceptions Automatically

The tool parses function bodies for explicit `raise` statements and builds a `Raises` section:

```python
@field_validator("api_config", mode="before")
@classmethod
def validate_api_config(cls, values: dict) -> dict:
    required_keys = values.get("required_keys")
    if not required_keys:
        raise ValueError("required_keys must be provided")
    if not isinstance(required_keys, dict):
        raise TypeError("required_keys must be a dictionary")
    return values
```

It records both `ValueError` and `TypeError` directly from the code instead of waiting for someone to manually transcribe them. 
Note that this tracks explicit exceptions in the function body; it won't catch errors thrown deeper down the call stack.

```python
@field_validator("api_config", mode="before")
@classmethod
def validate_api_config(cls, values: dict) -> dict:
    """
    Parameters
    ----------
    values : dict [Argument]

    Returns
    -------
    dict

    Raises
    -------
    ValueError
        If not required_keys
    TypeError
        If not isinstance(required_keys, dict)
    """
        
    required_keys = values.get("required_keys")
    if not required_keys:
        raise ValueError("required_keys must be provided")
    if not isinstance(required_keys, dict):
        raise TypeError("required_keys must be a dictionary")
    return values
```

More complex examples
```python
def reraise_with_context(data: dict, key: str):
    try:
        return data[key]
    except KeyError as exc:
        raise NotFoundError("DictEntry", key) from exc
```
becomes
```python
def reraise_with_context(data: dict, key: str):
    """
    Catch a KeyError and re-raise it chained to a NotFoundError
    using `raise X from Y` syntax.
    
    Parameters
    ----------
    data : dict [Argument]
    key : str [Argument]

    Raises
    -------
    NotFoundError
        Re-Raised from KeyError
    """
    try:
        return data[key]
    except KeyError as exc:
        raise NotFoundError("DictEntry", key) from exc
```

Limitations, when it comes to re-raising from multiple previous exceptions
```python
def parse_number(text: str) -> float:
    try:
        result = float(text)
    except (ValueError, OverflowError) as exc:
        raise ValidationError("text", f"cannot parse '{text}' as a number") from exc
    return result
```

becomes

```python
def parse_number(text: str) -> float:
    """
    Raise ValueError for non-numeric text or OverflowError for extremely
    large exponent notation; demonstrates multi-exception catching pattern.
    
    Parameters
    ----------
    text : str [Argument]
        text or OverflowError for extremely
    large exponent notation; demonstrates multi-exception catching pattern.

    Returns
    -------
    float

    Raises
    -------
    ValidationError
        If a certain condition is met.
    """
    try:
        result = float(text)
    except (ValueError, OverflowError) as exc:
        raise ValidationError("text", f"cannot parse '{text}' as a number") from exc
    return result
```

*If a certain condition is met.* means that currently I couldn't figure out the exact reason for this exception.

As with any static analysis, this describes what is visible in the function body; it is not a promise to discover every exception that could emerge from arbitrary code called further down the stack.

---

## Safe for Existing Codebases

You can preview all changes before modifying any files using dry-run mode:

```shell
gendocs_new src/ --dry-run
```

To scope the changes to only your current work in Git, use the `--changed-only` flag:

```shell
gendocs_new src/ --changed-only --dry-run
```

Projects can also exclude generated or internal code and skip magic methods:

```shell
gendocs_new src/ \
  --exclude-dir migrations \
  --exclude-dir tests \
  --exclude-file settings.py \
  --ignore-magic
```

Defaults can be defined in `pyproject.toml`:

```toml
[tool.docstring_generator]
strict = true
threshold = 90
exclude_files = ["settings.py"]
exclude_dirs = ["tests", "migrations"]
ignore_magic = true
```

**Command-line options overrides `pyproject.toml` definitions.**

---

## Coverage Checks for CI

You can audit your documentation coverage in your build pipeline.

Using `--check` to only check them without modifying any code.

```shell
gendocs_new src/ --check
```

Combine it with `--strict` to mark incomplete or outdated docstrings as undocumented and a threshold to fail the build if coverage drops too low:

```shell
gendocs_new src/ --check --strict --threshold 90
```

That command can run in a CI pipeline alongside tests and linting. A pull request that adds an undocumented function can then fail for a clear, reproducible reason rather than depending on reviewer memory.

**There is also a pre-commit hook available:**

```yaml
repos:
  - repo: https://github.com/FelixTheC/docstring_generator
    rev: <release-tag>
    hooks:
      - id: gendocs
        args: [src/, --style, google]
```

If the hook catches an outdated docstring, it updates the file and stops the commit so you can review the diff.

---

## Why a C++ Core?

Pre-commit hooks and potential CI checks need to be fast. 
 
The core engine lives in [`docstring_generator_ext`](https://github.com/FelixTheC/docstring_generator_ext), a C++20 extension connected to Python via pybind11. 
Python's `ast` module handles the initial code parsing, while the C++ extension quickly formats and injects the docstrings.

```text
Python files or directories
          |
          v
  gendocs_new CLI
          |
          v
docstring_generator_ext
   (C++20 + pybind11)
          |
          +---- NumPy
          +---- Google
          +---- reStructuredText
```

The extension can also be used directly from Python through `parse_file()` and exposes a read-only `check_docstring()` coverage API. Most users, however, can stay with the CLI and `pyproject.toml` configuration.

Distributed pre-built wheels are available, so you don't need a local C++ compiler to install it. 
If you want to build it from source, you will need Python 3.13+ and a compiler that supports C++20.

---

## The Right Tool for the Job

AI tools are excellent for drafting copy and explaining context, but they are not built to continuously maintain a codebase.

`docstring_generator` automates the mechanical parts: tracking signatures, parsing exceptions, and enforcing formats across your team.

Use AI to write your explanations, and use a fast, local tool to keep them accurate.


- **GitHub:** [github.com/FelixTheC/docstring_generator](https://github.com/FelixTheC/docstring_generator)
- **PyPI:** [pypi.org/project/docstring-generator](https://pypi.org/project/docstring-generator/)
- **C++ extension:** [github.com/FelixTheC/docstring_generator_ext](https://github.com/FelixTheC/docstring_generator_ext)

---

**Suggested Medium tags:** `Python` · `Clean Code` · `Software Development` · `Artificial Intelligence` · `Open Source`
