# Shiv Language Specification

**Status:** Draft 0.2
**Last updated:** September 24, 2026

This document records the design decisions made so far for Shiv. It covers syntax, semantics, and the reasoning behind each choice. Items are marked as **Decided**, **Proposed** (suggested but not yet confirmed), or **Open** (still needs a decision).

**Changes in 0.2:** added pointers, the memory model and allocators, null pointer rules, and `defer`. Updated modules (bare library types), keywords, sigils, built-ins, examples, rejected designs, open questions, and the roadmap.

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
12. [Pointers](#12-pointers)
13. [Memory and Allocators](#13-memory-and-allocators)
14. [Cleanup with defer](#14-cleanup-with-defer)
15. [Compiler Built-ins](#15-compiler-built-ins)
16. [Complete Examples](#16-complete-examples)
17. [Rejected Designs](#17-rejected-designs)
18. [Open Questions](#18-open-questions)
19. [Roadmap](#19-roadmap)

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

| Aspect | Decision |
|---|---|
| Paradigm | Imperative |
| Typing | Static, strict, fully annotated, no inference |
| Syntax family | C/Rust style: braces, semicolons, `//` comments |
| Error handling | Status codes carried alongside values (`(fl)` system), no exceptions |
| Memory model | Manual, with explicit allocators (Zig style) |

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
| `*` | In a type: **pointer to**. In an expression: **the data at** an address | `dv.*i32 px = &x;` / `*px = 7;` |

Because variables and functions use different sigils, a variable and a function may share a name without conflict.

Declarations do **not** use sigils. `dv.i32 x` declares `x`, and `$x` uses its value.

### 3.2 Comments

```
// Single-line comment

/*
   Multi-line comment
*/
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
| `defer` | Run a statement when the enclosing block exits |
| `true`, `false` | Boolean literals |
| `undef` | Explicitly uninitialized value |

### 3.4 Modifiers

Modifiers go in parentheses, separated by commas.

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
:println("{$x}");                 // value of x
:println("{&x}");                 // address of x
:println("{*px}");                // data at the address px holds
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

### 4.5 Types from modules

- **Types defined by a module are always used bare**, whichever `load` form is used. `:` means "function call," so it is never used in type names. For example, `mem` defines `ac`, written `*ac`, never `mem:ac`.
- Because the list of primitive types is small and fixed (see [8.1](#81-primitive-types)), any type not on it is visibly from a library.
- **Name clashes are compile errors.** If two loaded modules define the same type name, or a module's type clashes with one you define, the compiler refuses rather than guessing. Selective loading (`load mem("mem"):{ac, alloc}`) is the way out.

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
- **Zero-initialized by default.** A variable declared without a value starts at zero (or `false`, etc.). **Exception: pointers must be initialized** (see [12.4](#124-initialization-and-null)).
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
df greet(.i32 n, .i8 c) { ... }
```

Mutable parameters use `(mt).type name`, mirroring `dv(mt).type name`:

```
df count((mt).i32 n) {
    $n += 1;
}
```

Pointer parameters follow the pointer type rules (see [12.3](#123-pointer-mutability)):

```
df work(.*(mt)ac a) { ... }   // takes a pointer to a writable allocator
```

**Rules:**

- **Parameters are immutable by default.**
- **Parameters are copies.** Marking a parameter `mt` lets the function change its own local copy. The caller's variable is never affected. To change a caller's data, pass a writable pointer to it.

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

For how these interact with `defer`, see [section 14](#14-cleanup-with-defer).

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

This list is fixed. Anything else is a pointer type ([section 12](#12-pointers)) or a type defined by a module ([4.5](#45-types-from-modules)).

`str` is not yet designed. Strings depend on arrays and memory.

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

`usize` is the type for sizes, counts, and array indexes. Its size matches what the machine can address (64 bits on most modern hardware, 32 bits on 32-bit hardware), so code using it stays correct everywhere. Its exact role will be finalized alongside arrays.

### 8.5 Types are compile-time only

Types exist only while the program is being compiled. After compilation, nothing in the running program says "this is a `u8`." The type has been turned into decisions the compiler made, like how many bytes to read and which instructions to use.

```
dv.u8  a = 5;    →   store 1 byte at address A
dv.i32 b = 5;    →   store 4 bytes at address B
```

Consequences:

- Types are **not values**. They cannot be stored in variables or chosen while the program runs.
- Wherever a type is passed as an argument (as in `varcast` or `mem:alloc`), it must be written **literally and bare** in the source, e.g. `u8`, never `"u8"`.

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
load stdio();
load err();
load stdlib();

df main() {
    dv.i32 x = -5;
    dv(fl).u8 y = :varcast($x, u8);
    if (:readerr(&y) != 0) {
        :println("cast failed: {:readerr(&y)}");
    }
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

`varcast` is not an ordinary function. Because it accepts a type, the compiler generates specific conversion code for each type pair it's used with. It is a compiler built-in that lives under the `stdlib` name. See [Compiler Built-ins](#15-compiler-built-ins).

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

- `seterr(code)` sets the function's error code. **It does not exit the function.** A `return` is still required, in keeping with WYSIWYG: there is no hidden control flow.
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

## 12. Pointers

**Status: Decided** (pointer arithmetic is Open)

### 12.1 Syntax

Pointer syntax is close to C's.

| Syntax | Meaning |
|---|---|
| `*i32` (in a type) | Pointer to an `i32` |
| `&x` | Address of `x` |
| `$px` | Value of `px`, which is an address |
| `*px` (in an expression) | The data at the address `px` holds |

```
load stdio();

df main() {
    dv.i32 x = 5;
    :println("{$x}");      // value of x
    :println("{&x}");      // address of x
    dv.*i32 px = &x;
    :println("{$px}");     // address of x
    :println("{*px}");     // value of x
}
```

`$px` and `&x` print the same thing: a pointer's value *is* an address. This keeps `$` consistent: it always means "the value of this variable."

### 12.2 Writing through a pointer

```
*p = 42;   // store 42 at the address p holds
```

This requires a writable pointer (see 12.3).

### 12.3 Pointer mutability

A pointer has two independent properties:

1. **Can the pointer be repointed** to a different address?
2. **Can the data it points at be written** through it?

The placement rule: **modifiers before the `.` apply to the pointer variable, and modifiers after the `*` apply to the data it points at.**

| Declaration | Can repoint? | Can write through? |
|---|---|---|
| `dv.*i32 p` | No | No |
| `dv(mt).*i32 p` | Yes | No |
| `dv.*(mt)i32 p` | No | Yes |
| `dv(mt).*(mt)i32 p` | Yes | Yes |

**The safety rule: a read-only pointer can point at anything, but a writable pointer (`*(mt)`) can only ever point at mutable data.**

```
dv.i32 a = 5;                  // immutable directly
dv.*i32 pa = &a;               // can't be repointed, a can't be changed through pa

dv(mt).i32 b = 5;              // mutable directly
dv.*(mt)i32 pb = &b;           // writable through pb, can't be repointed
dv(mt).*(mt)i32 p2b = &b;      // can be repointed, writable through p2b

dv.*(mt)i32 pc = &a;           // COMPILE ERROR: a is immutable
$p2b = &a;                     // COMPILE ERROR: p2b is *(mt)i32, a is immutable
```

A pointer's permissions are part of its type, which is fixed at compile time. A writable pointer never silently becomes read-only. Anything that would break its guarantee is refused.

To move one pointer between mutable and immutable data, declare it read-only. Reading mutable data through a read-only pointer is always safe:

```
dv(mt).*i32 r = &a;    // fine: read-only pointer to immutable a
$r = &b;               // fine: read-only pointer to mutable b
```

### 12.4 Initialization and null

**Plain pointers must be initialized.** The zero-initialization default does not apply to pointers, because a zero pointer is the null address, and dereferencing it crashes.

```
dv.*i32 p;            // COMPILE ERROR: pointers must be initialized
dv.*i32 p = &x;       // fine
dv.*i32 p = undef;    // allowed, but dangerous: holds a leftover random address
```

| Declaration | `p` holds | Dereferencing it |
|---|---|---|
| `dv.*i32 p;` | (compile error) | n/a |
| `dv.*i32 p = undef;` | Random leftover address | Crash or silent corruption |
| `dv.*i32 p = &x;` | A real address | Works |

**"No address" only exists through `(fl)`.** Anything that might fail to produce an address, such as `mem:alloc`, returns a `(fl)` pointer. On failure, its value is `0` and its error code is set. Check the code before dereferencing:

```
dv(fl).*(mt)i32 p = mem:alloc($a, i32);
if (:readerr(&p) != 0) { exit 1; }
*p = 42;     // only reached if p is real
```

**The guarantee:** a plain pointer is never null unless the programmer wrote `undef`, and anything that might be null is marked `(fl)`. All of this is enforced at compile time, with no runtime cost.

Automatically assigning a "safe" address to an uninitialized pointer was rejected: it would be hidden behavior, and no address is universally safe.

---

## 13. Memory and Allocators

**Status: Decided**

### 13.1 Stack and heap

- **The stack** holds local variables. Space is reclaimed automatically when a function returns. It's extremely fast, but sizes must be known at compile time, and the data dies with its function.
- **The heap** holds data whose size is only known at runtime, or that must outlive the function that created it. Heap memory must be requested and given back.

Memory management in Shiv is entirely about the heap.

### 13.2 The model: manual, with explicit allocators

Shiv uses manual memory management, as in C: memory is allocated and freed by hand. The difference is that **an allocator is something you create and pass around**, rather than a single hidden global like C's `malloc`.

What this fixes compared to C:

| C | Shiv |
|---|---|
| Any library function may call `malloc` internally. Nothing in its signature tells you. | Code that allocates must be handed an allocator, so it's visible in the signature. Nothing allocates behind your back. |
| One strategy (`malloc`) for everything. | Choose the allocator that fits the job. |
| `malloc` returns `NULL` on failure, and most code never checks. | Allocation is `(fl)`. Failure is an error code, checked like any other. |
| Hunting leaks needs external tools or rebuild flags. | Swap in a checking allocator with one visible line of code. |

### 13.3 The `mem` module

All allocator functionality lives in `mem`.

**Allocator type:** `ac`, an **opaque type**. You never hold an allocator directly, only a pointer to one: `*ac`. You can't look inside it or write to it yourself. Only `mem`'s functions touch its contents. This works like C's `FILE*`.

Allocator pointers are declared `*(mt)ac`, because `mem:alloc` and `mem:free` change the allocator's internal bookkeeping on every call. A read-only `*ac` can be passed to functions that only inspect an allocator, but passing one to `mem:alloc` is a compile error.

**Creating allocators:** each kind has its own constructor, because each kind needs different arguments.

| Function | Creates | Arguments |
|---|---|---|
| `mem:mkgen()` | General-purpose allocator | None |
| `mem:mkarena()` | Arena allocator | None |
| `mem:mkfixed(&buf, size)` | Fixed-buffer allocator | A block of memory and its size |
| `mem:mkcheck($a)` | Checking allocator | An allocator to wrap |

All constructors are `(fl)` and return a `*(mt)ac`.

**Using allocators:**

| Function | Purpose |
|---|---|
| `mem:alloc($a, type)` | Allocate space for one value of `type`. `(fl)`, returns a `*(mt)type`. |
| `mem:free($a, $p)` | Give the memory at `p` back to allocator `a`. |
| `mem:rmalloc($a)` | Destroy the allocator itself. Every allocator needs one. |

Allocators are passed as `$a`, not `&a`. `a` is already a pointer, so its value *is* the allocator's address. `&a` would be the address of the pointer.

`mem:mkfixed`'s buffer argument depends on arrays, which are not yet designed.

### 13.4 Allocator kinds

| Kind | How it works | Best for |
|---|---|---|
| General | Tracks free and used blocks. Any piece can be freed at any time. | Flexible, moderately fast. The default choice. |
| Arena | Keeps a pointer to the next free spot and bumps it forward on each allocation. Individual frees do nothing; everything is freed at once. | Very fast. "Build a lot, use it, throw it all away," e.g. parsing a file. |
| Fixed buffer | Carves allocations out of a block you provide, even one on the stack. Never touches the OS. | Embedded work, no-OS environments. |
| Checking | Wraps another allocator and records every allocation. | Reports leaks, double frees, and use-after-free. |

Underneath, every allocator gets memory from the OS in large chunks (or, for fixed buffers, from memory you provide) and differs only in how it hands out pieces. `mkgen` and `mkarena` can store their own bookkeeping at the start of the memory they get from the OS.

### 13.5 Functions that allocate

Any function that needs heap memory takes an allocator parameter, so its signature says so:

```
df(fl).*(mt)i32 makenum(.*(mt)ac a, .i32 v) { ... }
```

### 13.6 Example

```
load mem("mem");
load err();

df main() {
    dv(fl).*(mt)ac a = mem:mkgen();          // create a general allocator
    if (:readerr(&a) != 0) { exit 1; }

    dv(fl).*(mt)i32 p = mem:alloc($a, i32);  // ask a for memory
    if (:readerr(&p) != 0) { exit 2; }

    *p = 42;
    mem:free($a, $p);                        // give p's space back to a
    mem:rmalloc($a);                         // destroy the allocator
}
```

---

## 14. Cleanup with defer

**Status: Decided** (interaction with `exit` is Open)

### 14.1 The problem

With manual memory, every allocation needs a matching `free` on **every** path out of a function. Error handling creates many paths, and each early `return` needs its own copy of the right cleanup. Forgetting one leaks memory.

### 14.2 `defer`

`defer` lets cleanup sit right next to what it cleans up:

```
load mem("mem");
load err();

df(fl).i32 work(.*(mt)ac a) {
    dv(fl).*(mt)i32 p = mem:alloc($a, i32);
    if (:readerr(&p) != 0) { :seterr(1); return 0; }
    defer mem:free($a, $p);

    dv(fl).*(mt)i32 q = mem:alloc($a, i32);
    if (:readerr(&q) != 0) { :seterr(1); return 0; }   // p is freed automatically
    defer mem:free($a, $q);

    return 1;                                            // q freed, then p
}
```

### 14.3 Rules

- `defer <statement>;` runs the statement when the **enclosing block** exits, by `return`, `break`, or reaching the closing brace.
- Multiple defers run in **reverse order**, so later allocations are freed first.
- A defer only applies if execution actually **reached** it. In the example above, if `p`'s allocation fails, the early `return` does not try to free `p`, because its `defer` line had not been reached.
- **`exit` does not run plain defers.** `exit` ends the program nearly instantly. The OS discards the process's entire memory space in one step, so no memory leaks past the program's end. This is not garbage collection: nothing is tracked or freed individually.

### 14.4 Why defer passes WYSIWYG

The deferred statement runs somewhere other than where it's written. It's accepted because the `defer` keyword is visible and the rule is simple and local: it always runs at the end of *this* block, never anywhere surprising.

### 14.5 Caveat: `exit` skips everything deferred

`exit` skips *every* deferred line, not just frees. If a deferred line matters beyond memory, like flushing a file buffer to disk, an `exit` will skip it. How to handle that is an open question (see [18](#18-open-questions)).

---

## 15. Compiler Built-ins

Some features look like library functions but must be implemented by the compiler, because they touch things only the compiler controls.

| Built-in | Why it must be built in |
|---|---|
| `err:seterr` | Writes into the hidden status slot of a function's return. |
| `err:readerr` | Reads from a fixed offset the compiler computes from the variable's type. Inlined to one load. |
| `stdlib:varcast` | Accepts a type as an argument, which no ordinary function can do. |
| `mem:alloc` | Accepts a type as an argument; the compiler supplies its size. |
| `lib:dumpaddr` | Needs the variable's full layout from the compiler. |

**Open:** whether `err` must be loaded with `load err();` or is always available. Since `(fl)` functions can't signal failure without `seterr`, making `err` always available may be simpler.

---

## 16. Complete Examples

### 16.1 Hello World loop

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

### 16.2 Fallible function and error checking

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

### 16.3 Casting

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

### 16.4 Overflow detection

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

### 16.5 Pointers and mutability

```
load stdio();

df main() {
    dv.i32 x = 5;
    dv.*i32 px = &x;
    :println("{$px}");                 // address of x
    :println("{*px}");                 // 5

    dv(mt).i32 b = 10;
    dv.*(mt)i32 pb = &b;
    *pb = 20;                          // b is now 20
    :println("{$b}");                  // 20
}
```

### 16.6 Allocation with defer

```
load stdio();
load mem("mem");
load err();

df main() {
    dv(fl).*(mt)ac a = mem:mkgen();
    if (:readerr(&a) != 0) { exit 1; }
    defer mem:rmalloc($a);

    dv(fl).*(mt)i32 p = mem:alloc($a, i32);
    if (:readerr(&p) != 0) { exit 2; }
    defer mem:free($a, $p);

    *p = 42;
    :println("{*p}");                  // 42
}                                      // p freed, then a destroyed
```

---

## 17. Rejected Designs

These were considered and rejected. They're recorded so the reasoning isn't lost.

### Error handling and types

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

### Memory and pointers

| Design | Why rejected |
|---|---|
| **Garbage collection** | Unpredictable pauses, extra memory, and work the programmer never wrote. Conflicts with every Shiv principle. |
| **Reference counting** | Hidden counter updates on every copy. Fails WYSIWYG. |
| **Ownership and borrow checking** (Rust) | Strict rules that often reject correct code, which conflicts with "code freedom." Very complex to design. |
| **Plain C-style global `malloc`/`free`** | Allocation is invisible, the strategy can't be swapped, and failure is easy to ignore. Replaced by explicit allocators. |
| **`alloc` as a primitive-like type** | An allocator is compound bookkeeping data, not a basic value like `i32`. Replaced by the opaque `*ac`. |
| **Allocator handles as numeric IDs** | A table lookup on every allocation, and the compiler can't tell an allocator ID from any other `usize`. |
| **A `da` declaration keyword for allocators** | Makes allocators a special case in the language, with no answer for files, threads, or sockets. |
| **Module-prefixed types** (`mem:ac`) | `:` means "function call" and must keep one meaning. Module types are bare. |
| **One `mkalloc(kind)` with named constants** | Each kind needs different arguments. Separate constructors (`mkgen`, `mkarena`, ...) avoid constants entirely. |
| **One `(mt)` for pointers** (covering both repointing and writing through) | Can't express "movable but read-only," which is needed for scanning data safely. |
| **"If the variable is mutable, so is the pointer"** | Breaks immutability: an immutable variable could be changed through a pointer. |
| **Pointers silently becoming read-only** when repointed at immutable data | Permissions are part of the type, fixed at compile time. Changing them silently is hidden behavior. Compile error instead. |
| **Null pointers allowed anywhere** | The most common crash in C. Null only exists through `(fl)`. |
| **Automatically assigning a safe address to uninitialized pointers** | Hidden behavior, and no address is universally safe. Compile error instead. |
| **`exit` freeing all memory** | That would be a garbage collector. `exit` stays nearly instant and leaves memory to the OS. |

---

## 18. Open Questions

1. **Pending cleanup on `exit`.** `exit` skips plain defers, which is fine for memory but not for things like flushing a file. Options under consideration:
    - A marked defer (working name `persist`; `always` also suggested) that runs on normal block exit **and** on `exit`. Implemented with a small list: each marked line adds an entry when reached and removes it on normal block exit; `exit` runs the list, newest first. Costs a few instructions per marked line, nothing for plain `defer`, and no binary growth.
    - Having `exit` scan up through every calling function for marked lines. Zero cost during normal execution, but requires compiler-generated unwind tables and a runtime unwinder, which grows binaries. This is C++'s exception machinery.
2. **Pointer arithmetic.** Can a pointer be moved (e.g. `$p += 1;`), and by how much: one byte, or one element? This will be decided alongside arrays.
3. **Discarding errors on purpose.** Assigning a `(fl)` call to a plain variable is a compile error. Should there be an explicit way to ignore the status? One proposal: `dv.u8 r = :f(5)(ig);`
4. **Size of the error code.** A `u8` allows 255 codes in one byte. A `u16` gives more room but may grow the bundle for small types after alignment.
5. **Clearing sticky codes.** Is `:seterr(&n, 0);` the syntax? Should in-function `seterr(code)` and variable `seterr(&var, code)` share a name?
6. **Is `err` always available**, or must it be loaded? See [section 15](#15-compiler-built-ins).
7. **Custom fallback for `varcast`.** For example, `:varcast($x, u8, 7)` returns 7 instead of 0 on failure.
8. **Name collisions between functions** from unprefixed modules and local functions, since both are called with a bare `:`. (Type clashes are already compile errors; see [4.5](#45-types-from-modules).)
9. **Casting data loss.** Beyond failing with a code, are there cases (e.g. `f64` to `i32`) that should truncate rather than fail?
10. **Escape hatches.** Which unchecked operations exist, and how are they marked in the source?

---

## 19. Roadmap

Topics to design next, roughly in order:

1. **Arrays**: fixed and dynamic, indexing with `usize`, pointer arithmetic, and the buffer argument for `mem:mkfixed`.
2. **Strings**: layout, UTF-8, and the relationship to `ch` and `ch32`.
3. **Compound types**: user-defined bundles of data (like C structs), and named constants (enums).
4. **Pending cleanup on `exit`**: see Open Question 1.
5. **Specific error types**: named errors beyond numeric codes.
6. **Events and interrupts**: handling asynchronous signals.
7. **Hardware faults**: segfaults, division by zero, and other failures the OS reports through signals rather than return values.
8. **Timeouts**: building on sticky `(fl)` codes.
