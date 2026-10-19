# astpp

**Self-Inlining Python Macro Preprocessor & Runtime Polyfill Library**

Project Homepage: [https://github.com/f-fix/astpp](https://github.com/f-fix/astpp)

PyPI Project Page: https://pypi.org/project/astpp/

---

## Overview

`astpp` is a dual-use Python library and AST-level macro preprocessor designed for cross-runtime codebases targeting **MicroPython**, **PyPy**, and **CPython**:

1. **Runtime Polyfill (Zero-Build Development)**:
   Importing `astpp` provides zero-overhead, runtime-compatible implementations of MicroPython compiler pragmas (`const`, `micropython.native`, `micropython.viper`) and conditional decorators (`keep`, `omit`, `keep_if`, `omit_if`). Scripts run directly under CPython, PyPy, or MicroPython without requiring an explicit compilation or preprocessing step.

2. **Compile-Time Preprocessor (Zero-Dependency Production Builds)**:
   When invoked as a script or via `astpp_main()`, `astpp` inspects the consumer's Abstract Syntax Tree (AST), removes `from astpp import ...` statements completely, and inlines minimal self-contained fallback definitions copied directly from its own source code. The emitted code has **zero external runtime dependencies** on `astpp.py`.

## Installation

### Via `pip` (CPython / PyPy / Host PC)

```bash
# Install from PyPI:
pip install astpp

# Or install directly from GitHub:
pip install git+https://github.com/f-fix/astpp.git

# Or install locally in editable development mode:
pip install -e .
```

### Via `mip` (MicroPython)

```python
# In MicroPython REPL (on Wi-Fi / Ethernet enabled board):
import mip
mip.install("github:f-fix/astpp")
```

```bash
# Or from your host computer over USB/UART via mpremote:
mpremote mip install github:f-fix/astpp
```

---

---

## Key Features

- **Macro Definitions (`-DNAME=VALUE`)**:
  Rewrites target assignments `NAME = ...` to `NAME = const(VALUE)`. Inlined constants are propagated directly into expressions throughout the module.
- **Dead-Branch Elimination**:
  Resolves `if`, `elif`, and `else` branches whose conditions evaluate to compile-time constants. Dead branches are pruned from the AST, while active branches are promoted or spliced without leaving ghost `pass` statements.
- **Symmetric Conditional Decorators (`@keep`, `@omit`, `@keep_if`, `@omit_if`)**:
  - `@keep`: Preserves the definition and strips the decorator (zero runtime overhead).
  - `@omit`: Drops the decorated function or class from the parse tree completely.
  - `@keep_if(cond)` / `@omit_if(cond)`: Evaluates compile-time conditions and resolves symmetrically to `@keep` or `@omit`.
- **Compiler Optimization Parity (`-O`, `-OO`, `-O0`–`-O3`)**:
  - **Default / `-O0`**: Preserves `__debug__` and assertions for downstream runtime evaluation.
  - **`-O` / `-O1`**: Sets `__debug__ = False` and purges all `assert` statements.
  - **`-OO` / `-O2`**: Purges assertions and strips docstrings from modules, classes, and functions.
  - **`-O3`**: Full constant propagation, dead branch pruning, and inlining.
- **Self-Contained Testing (`--test`)**:
  Executes built-in regression unit tests that parse and validate the documented examples directly from the module docstring.

---

## Command-Line Usage

```bash
# Transform via stdin / stdout filter:
python3 astpp.py [-O[n]] [-DNAME=VAL ...] < input.py > output.py

# Transform using named input and output files:
python3 astpp.py -O3 -D_IS_DESKTOP_PY=False input.py -o output.py

# Display rich formatted help:
python3 astpp.py --help

# Run self-contained test suite:
python3 astpp.py --test
```

### CLI Options

```text
usage: astpp [-h] [-o OUTPUT] [-D NAME=VALUE] [-O [LEVEL]] [-OO] [--test]
             [input]

Self-Inlining Macro Preprocessor & Runtime Polyfill Library.

positional arguments:
  input                 Input Python source file or '-' for stdin (default: '-')

options:
  -h, --help            Show this help message and exit
  -o OUTPUT, --output OUTPUT
                        Output destination file or '-' for stdout (default: '-')
  -D NAME=VALUE         Define macro replacement or override constant (e.g. -DNAME=VAL or -DNAME)
  -O [LEVEL]            Optimization level: -O (strip asserts, force __debug__=False),
                        -OO or -O2 (strip docstrings), -O3 (full)
  -OO                   Optimization level 2: strip assertions, __debug__=False, and docstrings
  --test                Run self-contained unit test suite
```

---

## Programmatic API

```python
from astpp import astpp_main
import io

source = """
from astpp import const
BUFFER_SIZE = const(1024)
"""

in_stream = io.StringIO(source)
out_stream = io.StringIO()

rc = astpp_main(["-O3", "-DBUFFER_SIZE=512"], in_stream, out_stream)
assert rc == 0
print(out_stream.getvalue())
```

---

## Documented & Tested Transformation Examples

### Example 1: Import Inlining & Self-Parsing
**Input**:
```python
from astpp import const
X = const(10)
```
**Flags**: `-O3`
**Output**:
```python
try:
    from micropython import const
except ImportError:

    def const(x):
        return x
X = const(10)
```

### Example 2: Constant Definition & In-Line Override
**Input**:
```python
_X = const(True)
BUFFER_SIZE = const(1024)
```
**Flags**: `-DBUFFER_SIZE=512 -O`
**Output**:
```python
BUFFER_SIZE = const(512)
```
*(Note: Private constants starting with `_` like `_X` are folded into load expressions and omitted from the output AST to optimize module dictionary space).*

### Example 3: Conditional Decorator Pruning
**Input**:
```python
@keep_if(_IS_DESKTOP_PY)
def desktop_helper():
    return 42
```
**Flags**: `-D_IS_DESKTOP_PY=False`
**Output**: *(Empty — function deleted from AST)*

### Example 4: Assertions and Debug Block Stripping
**Input**:
```python
assert 1 == 1, 'invariant'
if __debug__:
    def debug_only():
        pass
```
**Flags**: `-O`
**Output**: *(Empty — assertions and debug blocks pruned)*

### Example 5: Native Decorator Stripping via Macro Mapping
**Input**:
```python
@micropython.native
def fast_calc(x):
    return x * 2
```
**Flags**: `-Dmicropython.native=keep`
**Output**:
```python
def fast_calc(x):
    return x * 2
```

---

## Function & Runtime Reference

### `const(x)`
Identity fallback for compile-time constants on non-MicroPython runtimes. On MicroPython, `const()` is intercepted natively at parse time to inline values. On CPython and PyPy, this function acts as an identity pass-through returning `x`.

### `keep(obj)`
Identity decorator: keeps the decorated function or class as-is. When processed by `astpp`, the `@keep` decorator is stripped from the AST so that the compiled output incurs zero runtime decorator invocation overhead.

### `omit(obj)`
Omission decorator: drops the decorated function or class completely. When processed by `astpp`, any node decorated with `@omit` is purged from the parse tree.

### `keep_if(cond)`
Conditional decorator factory that preserves the target definition if `cond` is True. Returns the identity decorator `keep` if `cond` is True, or `omit` if False.

### `omit_if(cond)`
Conditional decorator factory that drops the target definition if `cond` is True. Returns the omission decorator `omit` if `cond` is True, or `keep` if False.

### `_MicroPythonShim`
Desktop compatibility shim providing no-op decorators (`native = staticmethod(keep)`, `viper = staticmethod(keep)`) for MicroPython compiler pragmas on non-MicroPython runtimes.

---

## License

MIT License. See `LICENSE` for details.

## Note on the code and the tools used to write it

Parts of this code were written (including some initial ones that began in other, separate projects) with assistance from LLM-integrated coding tools. If you don't like it, feel free to use other software or rewrite parts you dislike. PRs are welcome!

### How did I end up using those? Don't I dislike slop?

Yes, I hate slop. This project began because I wanted tape image conversion tools where the conversion steps were all clearly documented and readable code, but which also performed well enough in terms of accuracy to actually be the tool I use. I started out writing the tools myself, but my manual attempts hadn't yielded comparable accuracy to existing closed-source tools for some steps, so I started using the tools to help find the bugs and suggest improvements, and IMO the result is now good enough to actually be useful in some scenarios. In terms of slop, the tool-generated code doesn't closely resemble any existing solutions I have found. Rather it's a fairly passable translation of my requests into Python.
