This synthesis brings together the high-performance architectural details of your tool and the workflow friction created by modern AI coding assistants.

Here is the comprehensive, unified Medium post ready to publish.

---

# AI Tools Write Your Code, but Who Keeps Your Docstrings from Lying?

### Subtitle: AI assistants are great at writing a first draft. Here is why you still need a deterministic, C++ engine to stop documentation rot.

---
![Gemini_Generated_Image_iu4ui4iu4ui4iu4u.png](Gemini_Generated_Image_iu4ui4iu4ui4iu4u.png)

If you are writing Python today, you are likely using an AI assistant. Tools like Cursor, Copilot, and Claude Code have completely changed the way we build software. You write a function, hit a keyboard shortcut, and an AI seamlessly populates a perfectly formatted docstring based on your implementation.

Case closed, right? We don’t need specialized documentation tooling anymore.

Except there is a quiet, highly frustrating problem that defeats almost all passive AI-assisted documentation efforts: **Context Drift.**

AI tools are fantastic at *generating* documentation on demand, but they are completely passive. When a developer changes a function signature three weeks down the line—maybe shifting a parameter from `List[int]` to `List[int | str]`—the AI doesn't automatically fix the docstring. It sits silently until someone explicitly prompts it to rewrite the block.

If the developer forgets, that docstring is now actively lying to your team.

To solve this, we don’t need more AI generation; we need a deterministic **gatekeeper**. That is exactly why I built `docstring_generator`—a high-performance tool designed to bridge the gap between AI productivity and strict codebase reality.

---

## The Hidden Costs of AI Compliance

When you rely entirely on an editor extension to manage your team's documentation compliance, you run into three critical walls:

### 1. The Token Tax & Latency

Forcing a massive LLM to scan an entire enterprise repository on every single Git commit just to check if type hints match docstrings is incredibly slow and expensive.

### 2. The Hallucination Factor

For strict API contracts—like ensuring your `Raises` section accurately documents every exception thrown inside a complex validation loop—you want exact static analysis, not a probabilistic guess from a model.

### 3. Destruction of Context

Many basic AI prompts and simple scripts simply blow away your existing text and overwrite the entire block. If you wrote beautiful, nuanced business logic notes inside a parameter description, a blind overwrite destroys that context.

---

## Why I Built a C++ Backbone for Python Docs

When you integrate an auto-formatting tool or linter into a `pre-commit` hook, performance isn't just a vanity metric—it’s a feature. If a tool takes more than a few hundred milliseconds to run, developers will start bypassing it.

To ensure `docstring_generator` scales seamlessly across massive enterprise codebases without hitting performance bottlenecks, I moved away from standard pure-Python parsing.

The core engine is written in **C++** and exposed via **pybind11**. It safely parses and updates files in milliseconds. This delivers the lightning-fast execution speed developers expect from modern tools like `Ruff` or `uv`.

```
                  ┌──────────────────────────────┐
                  │    Target: Python Codebase   │
                  └──────────────┬───────────────┘
                                 │
                   [gendocs_new / pre-commit]
                                 │
                                 ▼
         ┌───────────────────────────────────────────────┐
         │         docstring_generator (CLI)             │
         └───────────────────────┬───────────────────────┘
                                 │
                            (pybind11)
                                 │
                                 ▼
         ┌───────────────────────────────────────────────┐
         │      C++ Core Engine (Non-Destructive AST)     │
         └───────────────────────┬───────────────────────┘
                                 │
     ┌───────────────────────────┼───────────────────────────┐
     ▼                           ▼                           ▼
[NumPy Style]             [Google Style]             [reStructuredText]

```

---

## How It Works Together with Your AI Workflow

The most effective modern workflow isn't AI *instead* of tooling; it’s AI *plus* tooling.

### 1. You Lock the Context, the Engine Handles the Types

You can use your AI assistant to write the initial, deep domain descriptions for your parameters. By dropping simple placeholders (`$1`, `$2`) into your initial text, you "lock" your custom notes in place.

`docstring_generator` takes over the mechanical work. If you alter your type hints later, the C++ engine instantly rewrites the structural boilerplate (supporting NumPy, Google, or reStructuredText) without ever touching or overwriting your custom explanations.

### 2. Exception-Aware Static Analysis

Documenting failure modes is critical, especially when dealing with validation frameworks like **Pydantic** or **FastAPI**.

Instead of guessing what an endpoint might throw, the tool statically extracts the exact `raise` statements directly from the AST (Abstract Syntax Tree) and automatically formats them into a dedicated `Raises` block:

```python
# AFTER RUNNING THE CLI ON A PYDANTIC VALIDATOR
@field_validator("api_config", mode='before')
@classmethod
def validate_api_config(cls, values: dict) -> dict:
    """
    Parameters
    ----------
    cls : [Argument]
    values : dict [Argument]

    Returns
    -------
    dict

    Raises
    -------
    ValueError
        If not required_key_obj
    """
    ...

```

### 3. CI-Ready Coverage Gating

You can’t drop a passive IDE extension into a GitHub Action to block a bad Pull Request. But you *can* drop a fast CLI tool.

The latest version introduces native pipeline flags:

* `--check`: Generates a total project docstring coverage report without touching the code.
* `--strict`: Rejects partial or incomplete docstrings.
* `--threshold 90`: Automatically fails the CI build if docstring coverage falls below your team's threshold.

It integrates seamlessly with `pyproject.toml`, allowing you to define these rules natively alongside your existing linting and testing stacks.

---

## Keeping It Automatic and Out of Sight

By dropping `docstring_generator` into a local `pre-commit` hook, you guarantee that documentation never drifts from the live type signatures—completely out of sight, running locally in milliseconds.

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/FelixTheC/docstring_generator
    rev: v1.0.2
    hooks:
      - id: gendocs
        args: [src/, --style, google]

```

Let AI write your initial thoughts and draft your code. But let a fast, deterministic engine ensure your codebase stays strictly compliant, unified, and accurate across every single commit.

👉 **GitHub Repo:** [https://github.com/FelixTheC/docstring_generator](https://github.com/FelixTheC/docstring_generator)
👉 **PyPI:** [https://pypi.org/project/docstring-generator/](https://pypi.org/project/docstring-generator/)

---

**Tags to use on Medium:** `Python` | `Clean Code` | `Software Development` | `Artificial Intelligence` | `Open Source`