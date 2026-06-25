# Real function overloading in Python — without `isinstance`, without naming sprawl
![Gemini_Generated_Image_8ghj6p8ghj6p8ghj.png](Gemini_Generated_Image_8ghj6p8ghj6p8ghj.png)

## TL;DR One decorator for real, runtime, multi-argument function overloading in Python — Pydantic-aware, works across modules.

[![PyPI](https://img.shields.io/pypi/v/strongtyping-pyoverload.svg)](https://pypi.org/project/strongtyping-pyoverload/)
[![Python versions](https://img.shields.io/pypi/pyversions/strongtyping-pyoverload.svg)](https://pypi.org/project/strongtyping-pyoverload/)
[![GitHub stars](https://img.shields.io/github/stars/FelixTheC/py-overload?style=social)](https://github.com/FelixTheC/py-overload)
![](https://img.shields.io/pypi/dm/pyoverload)
[![License](https://img.shields.io/pypi/l/strongtyping-pyoverload.svg)](https://github.com/FelixTheC/py-overload/blob/main/LICENSE)

Python has `typing.overload` (but it's just hints for your type checker) and `functools.singledispatch` (but only for the first argument). Anything more and you're back to writing 40-line `isinstance` ladders — or polluting your class with `process_str`, `process_list`, `process_dict`…

`strongtyping-pyoverload` gives you the real thing: **one decorator, multiple arguments, Pydantic-aware, works across modules.**

## The 10-second demo

```python
from strongtyping_pyoverload import overload

class Formatter:
    @overload
    def format(self, value: int) -> str:
        return f"ID: {value:04d}"

    @overload
    def format(self, value: str) -> str:
        return value.strip().capitalize()

Formatter().format(42)        # 'ID: 0042'
Formatter().format("  hi ")   # 'Hi'
```

No `isinstance`. No naming sprawl. No `Union` soup. The decorator picks the right implementation at runtime based on the actual argument types.

## Why this matters

Every Python codebase eventually grows the same pattern — a giant dispatcher glued together with `isinstance`:

```python
class DataProcessor:
    def process(self, data):
        if isinstance(data, str):
            return data.upper()
        elif isinstance(data, list):
            return [item * 2 for item in data]
        elif isinstance(data, MyPydanticModel):
            return data.model_dump()
        # It just keeps growing...
```

The "cleaner" alternative — splitting into `process_str`, `process_list`, `process_model` — only moves the mess. You still need a dispatcher, the names lie the moment a type hint changes, and unions make naming impossible:

```python
class DataProcessor:
    def process_str_int(self, data: str | int): ...   # name already lies
    def process_list(self, data: list | tuple | set): ...
    def process_dict(self, data: TypedDictObject): ...

    def process(self, data):
        if isinstance(data, str):
            return self.process_str_int(data)
        # ...and on, and on
```

## How it compares

| Tool                         | Multi-arg dispatch | Pydantic-aware | Cross-module | Runtime check |
|------------------------------|:------------------:|:--------------:|:------------:|:-------------:|
| `typing.overload`            | –                  | –              | –            | ❌ (static only) |
| `functools.singledispatch`   | ❌ (1st arg only)  | partial        | ✅           | ✅            |
| `multipledispatch` / `plum`  | ✅                 | ❌             | ✅           | ✅            |
| **`strongtyping-pyoverload`**| ✅                 | ✅             | ✅           | ✅            |

If you've ever wished `singledispatch` worked on the *second* argument, or wished `typing.overload` actually *did* something at runtime — this is that library.

## Where it really shines: the data boundary

When your business logic hits Pydantic models, deep inheritance trees, or `TypedDict`s, the decorator stays out of your way while enforcing the rules:

```python
class DataProcessor:
    @overload
    def process(self, data: str):
        return data.upper()

    @overload
    def process(self, data: MyPydanticModel):
        return data.model_dump()

    @overload
    def process(self, data: dict):
        """Fallback for plain dicts that aren't valid Pydantic models."""
        return data.values()
```

## Works across modules

Define overloads wherever they make sense — they're collected automatically:

```python
# module_a.py
from strongtyping_pyoverload import overload

@overload
def utils(data: dict, *, k: int): ...

@overload
def utils(data: list, *, k: int): ...

@overload
def utils(data: str, *, k: int): ...
```

```python
# module_b.py
from module_a import utils
from strongtyping_pyoverload.exception import InvalidOverloadException

def foobar(data):
    try:
        utils(data, k=4)
    except InvalidOverloadException:
        # No matching overload for this data type
        ...
```

If no overload matches, you get a dedicated `InvalidOverloadException` pointing at the exact arguments that failed — importable from `strongtyping_pyoverload.exception` so you can catch it precisely:

```
InvalidOverloadException: No function was found which matches your parameters `('This', 'Not', 'Supported')_{}`
```

## Limitations / gotchas

Being upfront so there are no surprises:

- **Runtime overhead.** Dispatch is fast but not free — for hot inner loops, a hand-written `isinstance` may still win.
- **Type hints are required** on every overload — that's the whole point, but worth stating.
- **Generics** (`list[int]` vs `list[str]`) are checked structurally; very deep generic checks can get expensive.
- **`async` functions** are supported; mixing sync and async under the same name is not recommended.

## Getting started

```bash
pip install strongtyping-pyoverload
```

🔗 **GitHub:** [FelixTheC/py-overload](https://github.com/FelixTheC/py-overload)
📖 **Docs:** [strongtyping-pyoverload.readthedocs.io](https://strongtyping-pyoverload.readthedocs.io/en/latest/)

---

*I built this because I got tired of writing the same dispatcher in every project. If it saves you some `isinstance` ladders too, a ⭐ on [GitHub](https://github.com/FelixTheC/py-overload) helps it get discovered — and feedback, issues, and PRs are always welcome.*

— Felix, maintainer of `strongtyping-pyoverload`
