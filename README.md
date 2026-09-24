# Shiv Language Specification

**Status:** Draft 0.1
**Last updated:** September 24, 2026

This document records the design decisions made so far for Shiv. It covers syntax, semantics, and the reasoning behind each choice. Items are marked as **Decided**, **Proposed** (suggested but not yet confirmed), or **Open** (still needs a decision).

---

## Table of Contents

1. [Goals and Philosophy](#1-goals-and-philosophy)
2. [Language Overview](#2-language-overview)
3. [Lexical Conventions](#3-lexical-conventions)
4. [Modules and Loading](#4-modules-and-loading)
5. [Variables](#5-variables)
6. [Functions](#6-functions)
7. [Control Flow](#7-control-flow)
8. [Types](#8-types)
9. [Casting](#9-casting)
10. [Error Handling](#10-error-handling)
11. [Arithmetic and Overflow](#11-arithmetic-and-overflow)
12. [Compiler Built-ins](#12-compiler-built-ins)
13. [Complete Examples](#13-complete-examples)
14. [Rejected Designs](#14-rejected-designs)
15. [Open Questions](#15-open-questions)
16. [Roadmap](#16-roadmap)

---

## 1. Goals and Philosophy

Shiv is a passion project with a serious target: **it should be able to replace C for low-level systems work.**

### Core principles

- **As fast and lightweight as possible.** Speed and memory footprint come first. Safety features are available but opt-in, and they are designed to cost as little as possible.
- **Optionally safe, easy to write, easy to audit.** Safety is something the programmer chooses, visibly, in the source.
- **What you see is what you get (WYSIWYG).** Inspired by Zig. There is no hidden control flow, no silent returns, and no invisible behavior. What is written is exactly what happens.
- **Verbose and explicit.** Every type is written out. **Shiv has no type inference.**
- **Strict.** Type mismatches the compiler can detect are compile errors, not runtime surprises.

---

## 2. Language Overview

| Aspect         | Decision                                                             |
| -------------- | -------------------------------------------------------------------- |
| Paradigm       | Imperative                                                           |
| Typing         | Static, strict, fully annotated, no inference                        |
| Syntax family  | C/Rust style: braces, semicolons, `//` comments                      |
| Error handling | Status codes carried alongside values (`(fl)` system), no exceptions |
| Memory model   | **Open**, to be designed                                             |

**Typing philosophy (Proposed):** static typing with explicit escape hatches. The compiler stops accidents but never stops the programmer from doing something deliberately, such as casts, raw pointers, or reinterpreting memory. This is roughly C's philosophy with safer defaults.

---

## 3. Lexical Conventions

### 3.1 Sigils

Shiv uses sigils so that every use site is unambiguous at a glance. Each sigil has exactly one meaning.

| Sigil | Meaning | Example |
|---|---|---|
| `$` | The **value** of a variable | `$x += 1;` |
| `:` | A **function call** | `:println("hi");` |
| `&` | The **address** of a variable | `:readerr(&r)` |

Because variables and functions use different sigils, a variable and a function may share a name without conflict.

Declarations do **not** use sigils. `dv.i32 x` declares `x`, and `$x` uses its value.

### 3.2 Comments

```
// Single-line comment
```

### 3.3 Keywords (current)

| Keyword | Purpose |
|---|---|
| `load` | Import a module |
| `df` | Declare function |
| `dv` | Declare variable |
| `if`, `else` | Conditionals |
| `until` | Loop |
| `break` | Exit a loop |
| `return` | Exit a function |
| `exit` | Exit the program immediately |
| `true`, `false` | Boolean literals |
| `undef` | Explicitly uninitialized value |

### 3.4 Modifiers

Modifiers go in parentheses directly after `df` or `dv`, separated by commas.

| Modifier | Meaning |
|---|---|
| `mt` | Mutable |
| `fl` | Fallible (carries an error code) |

```
dv(mt, fl).u8 x = 0;
```

### 3.5 The declaration pattern

Every declaration follows the same shape:

```
keyword(modifiers).type name
```

- `dv(mt).i32 x` → declare variable, mutable, type `i32`, named `x`
- `df(fl).u8 f` → declare function, fallible, returns `u8`, named `f`

New modifiers can be added later without new grammar.

### 3.6 String interpolation

Expressions inside `{}` in a string are evaluated and printed.

```
:println("{$r}");                 // value of r
:println("{&r}");                 // address of r
:println("{lib:dumpaddr(&r)}");   // result of a function call
```

---

## 4. Modules and Loading

**Status: Decided**

Modules are brought in with `load`. There are three forms, from least to most verbose.

### 4.1 Load everything, unprefixed

```
load stdio();
:println("Hello");
```

Loads the whole namespace. Its functions are called with a bare `:`.

### 4.2 Load everything, prefixed

```
load stdio("stdio");
stdio:println("Hello");
```

Loads the whole namespace. Its functions must be called with the given prefix.

### 4.3 Load specific items, prefixed

```
load stdio("stdio"):{println};
stdio:println("Hello");
```

Loads only the listed items. They must be called with the prefix.

### 4.4 Calling rules

- `module:func()` calls a function from a prefixed module.
- `:func()` calls a local function or a function from an unprefixed module.

---

## 5. Variables

**Status: Decided**

### 5.1 Declaration

```
dv.i32 x = 5;          // immutable i32
dv(mt).i32 y;          // mutable i32, zero-initialized
dv(fl).u8 r = :f(5);   // fallible u8 (value + error code)
dv(mt, fl).u8 n = 0;   // mutable and fallible
```

### 5.2 Rules

- **Immutable by default.** Mutation requires `mt`.
- **Zero-initialized by default.** A variable declared without a value starts at zero (or `false`, etc.).
- **Opting out of initialization:** `= undef` leaves the variable uninitialized for speed. Its contents are whatever bits were already in memory.

```
dv.i32 x = undef;   // uninitialized, fastest
```

---

## 6. Functions

**Status: Decided**

### 6.1 Declaration

```
df name(params) { ... }                 // returns nothing (void)
df.i32 name(params) { ... }             // returns i32
df(fl).u8 name(params) { ... }          // fallible, returns u8 + error code
```

A function with no return type returns nothing.

### 6.2 Parameters

Parameters use `.type name`:

```
df greet(.i32 n, .str s) { ... }
```

Mutable parameters use `(mt).type name`, mirroring `dv(mt).type name`:

```
df count((mt).i32 n) {
    $n += 1;
}
```

**Rules:**

- **Parameters are immutable by default.**
- **Parameters are copies.** Marking a parameter `mt` lets the function change its own local copy. The caller's variable is never affected. Changing a caller's variable will go through pointers (see [Roadmap](#16-roadmap)).

### 6.3 Calling

```
:func(5);          // local function
stdio:println();   // prefixed module function
```

### 6.4 The `main` function

`main` is the program's entry point. It can either return an exit code or return nothing and use `exit`.

```
df.i32 main() {
    return 0;   // exit code
}
```

```
df main() {
    exit 1;     // exits immediately with code 1
}
```

---

## 7. Control Flow

**Status: Decided**

### 7.1 Conditionals

```
if ($x == 5) {
    ...
} else if ($x == 6) {
    ...
} else {
    ...
}
```

### 7.2 The `until` loop

`until (condition)` repeats the body **until** the condition becomes true.

```
until ($x == $n) {
    $x += 1;
}
```

`until` with no condition loops forever:

```
until {
    if ($x == $n) {
        break;
    }
    $x += 1;
}
```

### 7.3 Termination instructions

Shiv has three:

| Instruction | Effect |
|---|---|
| `break;` | Exits the current loop |
| `return;` | Exits the current function |
| `exit code;` | Exits the whole program immediately with the given code |

---

## 8. Types

**Status: Decided**

### 8.1 Primitive types

| Kind | Types | Notes |
|---|---|---|
| Signed integers | `i8`, `i16`, `i32`, `i64` | |
| Unsigned integers | `u8`, `u16`, `u32`, `u64` | |
| Floats | `f32`, `f64` | |
| Boolean | `bool` | Distinct type, see 8.2 |
| Character (ASCII) | `ch` | 1 byte |
| Character (Unicode) | `ch32` | 4 bytes, one Unicode code point |
| Pointer-sized | `usize` | Matches the machine's address width |

`str` appears in parameter examples but is not yet designed. Strings depend on the memory model.

### 8.2 `bool`

- A **distinct type**, stored as a `u8` in memory.
- Only two legal values: `true` (1) and `false` (0).
- A `u8` cannot be assigned to a `bool` directly, which prevents invalid values like `7`. Conversion goes through `varcast`.

```
dv.bool ready = true;
```

### 8.3 `ch` and `ch32`

- `ch` is one byte and holds an ASCII character.
- `ch32` is four bytes and holds any Unicode code point.
- Naming mirrors the integer types (`u8` / `u32`).
- Unicode *text* will likely be stored as UTF-8 bytes rather than arrays of `ch32`. This is a strings question for later.

### 8.4 `usize`

`usize` is the type for sizes, counts, and array indexes. Its size matches what the machine can address (64 bits on most modern hardware, 32 bits on 32-bit hardware), so code using it stays correct everywhere. Its exact role will be finalized alongside pointers and arrays.

### 8.5 Types are compile-time only

Types exist only while the program is being compiled. After compilation, nothing in the running program says "this is a `u8`." The type has been turned into decisions the compiler made, like how many bytes to read and which instructions to use.

```
dv.u8  a = 5;    →   store 1 byte at address A
dv.i32 b = 5;    →   store 4 bytes at address B
```

Consequences:

- Types are **not values**. They cannot be stored in variables or chosen while the program runs.
- Wherever a type is passed as an argument (as in `varcast`), it must be written **literally and bare** in the source, e.g. `u8`, never `"u8"`.

### 8.6 Strictness

- There are **no implicit conversions** between types.
- Mismatches the compiler can see are **compile errors**:

```
df.u8 f() {
    return -5;   // COMPILE ERROR: -5 is not a valid u8
}
```

---

## 9. Casting

**Status: Decided**

Conversions between types use `varcast` from the standard library.

```
load stdlib();

dv.i32 x = -5;
dv(fl).u8 y = :varcast($x, u8);

if (:readerr(&y) != 0) {
    :println("cast failed: {:readerr(&y)}");
}
```

### 9.1 Rules

- **Signature:** `varcast(value, type)`
- **Takes a value** (`$x`), not an address. A cast only reads the source, and the result goes into a new variable.
- **The type is a bare literal** (`u8`), known at compile time. See [8.5](#85-types-are-compile-time-only).
- **Always fallible.** The result must go into a `(fl)` variable.
- **On success:** returns the converted value with error code `0`.
- **On failure:** returns the **default value of the target type** (`0`, `false`, etc.) and sets error code `1`.

Returning a default of the target type is required, not just convenient. Return types are fixed at compile time, so a cast can never hand back a value of the original type.

### 9.2 Implementation note

`varcast` is not an ordinary function. Because it accepts a type, the compiler generates specific conversion code for each type pair it's used with. It is a compiler built-in that lives under the `stdlib` name. See [Compiler Built-ins](#12-compiler-built-ins).

---

## 10. Error Handling

**Status: Decided** (surface-level error handling; specific error types, events, and interrupts come later)

### 10.1 Overview

Shiv does not use exceptions. Instead, **fallible functions return a status code alongside their value.** This gives try/except-style recovery, where the program checks for errors and handles them instead of crashing, with almost no runtime cost.

The whole system is built on one modifier: **`(fl)`, fallible.**

### 10.2 Fallible functions

A function declared with `(fl)` returns both a value and an error code.

```
df(fl).u8 f(.i32 n) {
    if ($n == 5) {
        err:seterr(1);   // set the failure code
        return 1;        // return is still required to exit
    } else {
        return 5;        // no code set, so it defaults to 0
    }
}
```

**Rules:**

- `err:seterr(code)` sets the function's error code. **It does not exit the function.** A `return` is still required, in keeping with WYSIWYG: there is no hidden control flow.
- **The function always returns a real value of its declared type**, even on failure. That value acts as a fallback.
- If no code is set, the code is `0`, meaning success.
- Failures happen **only** where code visibly calls `seterr`. Nothing becomes an error implicitly.

### 10.3 Fallible variables

The result of a `(fl)` function must be stored in a `(fl)` variable, which holds both the value and the code.

```
dv(fl).u8 r = :f(5);
```

**Assignment rules:**

| Assignment | Result |
|---|---|
| `(fl)` call → `(fl)` variable | Value and code both stored. The normal case. |
| `(fl)` call → plain variable | **Compile error.** This would silently drop the status. |
| Plain value → `(fl)` variable | Allowed. Code is set to `0`. |
| Reassigning a `(fl)` variable | Requires `mt`. See [11.3](#113-sticky-error-codes) for how codes behave. |

`df(fl)` and `dv(fl)` mirror each other. Anyone auditing a file can find every place an error can exist by searching for `(fl)`.

### 10.4 Reading error codes

Error codes are read by passing the **address** of a `(fl)` variable to `readerr`:

```
if (err:readerr(&r) == 1) {
    :println("f() failed");
} else if (err:readerr(&r) == 0) {
    :println("f() returned successfully");
}
```

**Why the address:** `$r` means the *value* of `r`, which is just a `u8` with no status attached. `&r` points at the whole bundle, value plus code, so `readerr` can reach the code.

**Rules:**

- `readerr` only accepts the address of a `(fl)` variable. Passing a plain variable's address is a compile error.
- Using `$r` gives the plain value only. Passing `$r` to a function that takes `.u8` is allowed and strips the status explicitly.

### 10.5 Memory layout and cost

- **At return:** the value and the code come back in two CPU registers. No memory write and no allocation.
- **In a variable:** the value and code are stored side by side. A `dv(fl).u8` is the value byte followed by the code byte. The code sits at a fixed offset the compiler knows from the declared type.
- **Reading:** because the offset is known at compile time, `readerr` can be inlined into a single memory load.
- **Cost:** a small amount of extra memory per `(fl)` variable, and one comparison per check.

### 10.6 Debugging

`lib:dumpaddr(&r)` dumps the full contents at a variable's address. For a `(fl)` variable, that includes both the value and the error code. It works because the compiler knows the variable's full layout.

```
:println("{lib:dumpaddr(&r)}");
```

---

## 11. Arithmetic and Overflow

**Status: Decided**

### 11.1 The rule

| Variable kind | On overflow |
|---|---|
| Plain | Wraps (raw CPU behavior). Nothing is reported. |
| `(fl)` | Wraps, and the error code is set. |

```
dv(mt).u8 a = 255;
$a += 1;             // a wraps to 0, silently

dv(mt, fl).u8 b = 255;
$b += 1;             // b wraps to 0, error code set
```

### 11.2 Why

- **Plain arithmetic** compiles to a raw CPU instruction with zero overhead. It's the same as C, except that signed overflow in Shiv is **defined** as wrapping. In C, it's undefined behavior.
- **`(fl)` arithmetic** costs one extra instruction. The CPU already sets an overflow flag on every operation, and x86 and ARM can copy that flag straight into a byte **without branching**.
- Every overflow is either reported (on a `(fl)` variable) or the result of a visible choice to use a plain variable. **Use `(fl)` wherever overflow matters.**

### 11.3 Sticky error codes

Error codes on `(fl)` variables are **sticky**. Once set, a code stays set until code visibly clears it. Later successful operations do not reset it.

```
dv(mt, fl).u8 n = 0;
$n += 200;
$n += 100;                 // overflows, code set
$n += 1;                   // fine, but the code STAYS set
if (:readerr(&n) == 1) {   // still catches the earlier overflow
    :println("overflow somewhere above");
}
```

**Why sticky:** with overwriting codes, a successful operation could silently erase an earlier error before anything checked it. Sticky codes mean an error can never disappear on its own. This is the WYSIWYG-safe choice.

**Consequences:**

- A batch of operations can be checked once at the end.
- Before reusing a variable, clear its code explicitly (**Proposed** syntax):

```
:seterr(&n, 0);
```

- Sticky codes are also branchless: copy the overflow flag and OR it into the code. They cost the same as overwriting.

### 11.4 Increment

Shiv uses `+= 1`. There is **no `++` operator.**

### 11.5 Example: counting until overflow

```
dv(mt, fl).u8 n = 0;
until (:readerr(&n) == 1) {
    $n += 1;
}
// ends when n wraps from 255 to 0 and the code is set
```

---

## 12. Compiler Built-ins

Some features look like library functions but must be implemented by the compiler, because they touch things only the compiler controls.

| Built-in | Why it must be built in |
|---|---|
| `err:seterr` | Writes into the hidden status slot of a function's return. |
| `err:readerr` | Reads from a fixed offset the compiler computes from the variable's type. Inlined to one load. |
| `stdlib:varcast` | Accepts a type as an argument, which no ordinary function can do. |
| `lib:dumpaddr` | Needs the variable's full layout from the compiler. |

**Open:** whether `err` must be loaded with `load err();` or is always available. Since `(fl)` functions can't signal failure without `seterr`, making `err` always available may be simpler.

---

## 13. Complete Examples

### 13.1 Hello World loop

```
load stdio();

df func(.i32 n) {                   // void, takes an i32
    dv(mt).i32 x;                   // mutable, zero-initialized
    until ($x == $n) {
        :println("Hello, World!");
        $x += 1;
    }
}

df.i32 main() {
    :func(5);
    return 0;
}
```

### 13.2 Fallible function and error checking

```
load stdio();
load err("err");
load stdlib("lib");

df(fl).u8 f(.i32 n) {
    if ($n == 5) {
        err:seterr(1);              // sets the failure code to 1
        return 1;                   // still needed to exit
    } else {
        return 5;                   // code defaults to 0
    }
}

df main() {
    dv(fl).u8 r = :f(5);
    if (err:readerr(&r) == 1) {
        :println("f() failed");
    } else if (err:readerr(&r) == 0) {
        :println("f() returned successfully");
    }
    :println("{$r}");                  // 1
    :println("{&r}");                  // address of r
    :println("{lib:dumpaddr(&r)}");    // value of r and its error code
}
```

### 13.3 Casting

```
load stdio();
load err();
load stdlib();

df main() {
    dv.i32 x = -5;
    dv(fl).u8 y = :varcast($x, u8);    // fails: returns 0, code 1
    if (:readerr(&y) != 0) {
        :println("{:readerr(&y)}");
    }
}
```

### 13.4 Overflow detection

```
load stdio();
load err();

df main() {
    dv(mt, fl).u8 n = 0;
    until (:readerr(&n) == 1) {
        $n += 1;
    }
    :println("wrapped to {$n}");       // wrapped to 0
}
```

---

## 14. Rejected Designs

These were considered and rejected. They're recorded so the reasoning isn't lost.

| Design | Why rejected |
|---|---|
| **C++-style exceptions** | Zero cost when nothing fails, but throwing is slow, binaries grow with unwind tables, and a runtime is required. Too heavy. |
| **`undef` as a failure value** (`if (:f() == undef)`) | Every bit pattern of a type is a valid value. A `u8` has no spare pattern for "undef," so failure can't be detected without an extra flag. `undef` also already means "uninitialized," and giving it two meanings hurts auditability. |
| **Switching the return type at runtime on failure** | The caller compiles against a fixed layout, so types can't change at runtime. This idea became the status-code design. |
| **A `fail` keyword that exits the function** | Replaced by `seterr` + an explicit `return`, which keeps control flow visible. |
| **Implicit failure on type mismatch** (e.g. `return -3;` in a `u8` function failing at runtime) | A line that says `return` but actually fails is hidden control flow. Compile-time mismatches are compile errors instead. |
| **"Last error" global storage** (like C's `errno`) | Any call in between wipes it, even `println`. Hidden state, and unsafe with threads. |
| **Overflow leaves the value unchanged** | Requires a check and undo after every operation, even on plain variables. That's too slow. Replaced by wrapping. |
| **Build modes (safe/fast)** | Rejected in favor of per-variable control through `(fl)`. |
| **Overwriting error codes** | A successful operation could silently erase an unchecked error. Replaced by sticky codes. |
| **Passing types as strings** (`"u8"`) | Strings are runtime data, but casts need the type at compile time. This would invite code like `:varcast($x, $t)`, which can never work. |
| **`++` operator** | `+= 1` only. |
| **Class-based OOP** | Adds hidden costs (vtables, inheritance) that systems programmers usually don't want by default. |

---

## 15. Open Questions

1. **Discarding errors on purpose.** Assigning a `(fl)` call to a plain variable is a compile error. Should there be an explicit way to ignore the status? One proposal: `dv.u8 r = :f(5)(ig);`
2. **Size of the error code.** A `u8` allows 255 codes in one byte. A `u16` gives more room but may grow the bundle for small types after alignment.
3. **Clearing sticky codes.** Is `:seterr(&n, 0);` the syntax? Should in-function `seterr(code)` and variable `seterr(&var, code)` share a name?
4. **Is `err` always available**, or must it be loaded? See [section 12](#12-compiler-built-ins).
5. **Custom fallback for `varcast`.** For example, `:varcast($x, u8, 7)` returns 7 instead of 0 on failure.
6. **Name collisions** between local functions and functions from unprefixed modules, since both are called with a bare `:`.
7. **Casting data loss.** Beyond failing with a code, are there cases (e.g. `f64` to `i32`) that should truncate rather than fail?
8. **Escape hatches.** Which unchecked operations exist, and how are they marked in the source?

---

## 16. Roadmap

Topics to design next, roughly in order:

1. **Pointers and memory**: address syntax beyond `&`, pointer types, the role of `usize`, and the memory model (garbage collection, ownership, manual management, etc.).
2. **Strings**: layout, UTF-8, and the relationship to `ch` and `ch32`.
3. **Arrays**: fixed and dynamic, indexing with `usize`.
4. **Specific error types**: named errors beyond numeric codes.
5. **Events and interrupts**: handling asynchronous signals.
6. **Hardware faults**: segfaults, division by zero, and other failures the OS reports through signals rather than return values.
7. **Timeouts**: building on sticky `(fl)` codes.
