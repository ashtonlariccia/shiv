## The Shiv Programming Language
### Designed by Ashton LaRiccia
**Shiv is a passion project of mine that aims to replace C for low-level systems work**

#### Philosophy and Principles
- As fast and as lightweight as possible
    - Speed & Memory footprint are top priority
    - Safety features are opt-in and designed to be as minimal as possible
    - Optionally safe, easy to write, easy to audit
    - What you see is what you get: Inspired by Zig
        - No hidden control flow, silent returns, or invisible behavior
    - Verbosity and explicitness: absolutely no inference, very strict

#### Overview
| Aspect         | Decision                                                             |
| -------------- | -------------------------------------------------------------------- |
| Paradigm       | Imperative                                                           |
| Typing         | Static, strict, fully annotated, no inference                        |
| Syntax family  | C/Rust style                                                         |
| Error handling | Status codes carried alongside values (`(fl)` system), no exceptions |
| Memory model   | **Open**, to be designed                                             |

