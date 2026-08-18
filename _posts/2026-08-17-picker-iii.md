---
layout: post
title: "picoCTF: Picker III"
date: 2026-08-17
categories: [picoCTF, reverse-engineering]
tags: [python, eval, exec, sandbox-escape]
---

## Overview

**Challenge:** Picker III
**Category:** Reverse Engineering
**Difficulty:** Medium
**Author:** LT 'syreal' Jones

> Can you figure out how this program works to get the flag?

Picker III is the third entry in a series, and its whole premise is that it's supposed to be the *fixed* version of Picker II — a program that got compromised because it let users call arbitrary functions by name through `eval()`. This time, the challenge author restricts function calls to a predefined table of "safe" functions. The interesting part of this challenge isn't finding a hole in the filters directly — it's realizing the filters were never the actual attack surface.

## Design Intent

Reading through `picker-III.py`, the program exposes a small REPL-style menu:

- `1`–`4` — call one of four functions listed in a `func_table`
- `help` — print help text and the current table
- `reset` — reset the table to its default state
- `quit` — exit

Two hidden functions are reachable *through* the table's entries — `read_variable` and `write_variable` — which let the user define arbitrary global variables and read them back:

```python
def read_variable():
  var_name = input('Please enter variable name to read: ')
  if( filter_var_name(var_name) ):
    eval('print('+var_name+')')
  else:
    print('Illegal variable name')

def write_variable():
  var_name = input('Please enter variable name to write: ')
  if( filter_var_name(var_name) ):
    value = input('Please enter new value of variable: ')
    if( filter_value(value) ):
      exec('global '+var_name+'; '+var_name+' = '+value)
    else:
      print('Illegal value')
  else:
    print('Illegal variable name')
```

Two filters guard this:

```python
def filter_var_name(var_name):
  r = re.search('^[a-zA-Z_][a-zA-Z_0-9]*$', var_name)
  return bool(r)

def filter_value(value):
  if ';' in value or '(' in value or ')' in value:
    return False
  else:
    return True
```

`filter_var_name` restricts `var_name` to a legal Python identifier: it must start with a letter or underscore, and every character after that must be a letter, digit, or underscore. `filter_value` blocks `;`, `(`, and `)` from the value — clearly aimed at stopping statement chaining and function calls.

The design intent is: let the user create and inspect their own variables, but make it structurally impossible to *call* anything through that mechanism, since a function call syntactically requires parentheses. Function calls, meanwhile, are restricted entirely to whatever's sitting in `func_table`, which is populated only by `reset_table()`:

```python
def reset_table():
  global func_table
  func_table = \
'''\
print_table                     \
read_variable                   \
write_variable                  \
getRandomNumber                 \
'''
```

`FUNC_TABLE_SIZE = 4` and `FUNC_TABLE_ENTRY_SIZE = 32` — so this is really one 128-character string split into four fixed-width, space-padded 32-character slots. `get_func(n)` slices out slot `n` and trims at the first space; `call_func(n)` calls whatever name comes out:

```python
def call_func(n):
  ...
  func_name = get_func(n)
  eval(func_name+'()')
```

So the intended security boundary is: *variables* are freely user-controlled but can't be turned into calls, while *calls* are tightly restricted to a hardcoded table that (in the author's mental model) only `reset_table()` ever writes to.

## Initial Approach

I started by mapping out what was actually reachable from the menu, then spent most of my time just tinkering with `read_variable` and `write_variable` to get a feel for what the filters would and wouldn't let through.

My first real idea was to try to sneak parentheses past `filter_value` by encoding them — I tried setting a variable's value to something like `win\x28\x29`, hoping `exec` would decode the escape sequence before the check mattered, or that the check itself would fail to catch it. That didn't pan out; it just caused the program to hang.

From there I went back to the source and noticed `read_variable` uses `eval('print('+var_name+')')` — not `write_variable`. That distinction mattered. Since `eval` in `read_variable` interpolates into `var_name`'s *slot*, not `value`'s, and `filter_var_name` only allows identifier characters, I couldn't inject parentheses there either. But I could still make `var_name` be the literal name of an existing function, like `win`.

When I read a variable named `win`, the output was:

```
<function win at 0x7fbf69676dc0>
```

This was the same behavior I'd seen in Picker II — `eval('print(win)')` just prints the function object's repr, since `win` all by itself (no trailing `()`) is only ever *evaluated as an expression*, not *called*. It confirmed that `win` exists as a real function in the program's namespace and that I could refer to it by name — but referring to it isn't the same as invoking it, and the filters made sure I couldn't append `()` to force a call this way.

At this point I was stuck on the idea that the vulnerability had to be some way of getting parentheses — or an equivalent of them — past `filter_value` or `filter_var_name`. I checked the hint:

> Is there any way to modify the function table?

That reframed the whole problem for me. I'd been treating `write_variable` as a way to define *new*, harmless variables. But `write_variable` doesn't check *which* global it's writing to — it just checks that the name is a syntactically legal identifier. And `func_table` is a syntactically legal identifier.

## Vulnerability / Concept

The actual vulnerability isn't in either filter individually — both do exactly what they claim to do. `filter_var_name` really does restrict names to legal identifiers. `filter_value` really does block `;`, `(`, and `)`. The bug is a **scope/trust boundary failure**: the program assumes `func_table` is a privileged, program-internal variable that only `reset_table()` can touch, but at runtime it's just an ordinary global string, and `write_variable` treats *all* globals as equally writable as long as the name passes the identifier check.

`write_variable` builds and runs:

```python
exec('global '+var_name+'; '+var_name+' = '+value)
```

Nothing here distinguishes "variables the program considers safe to expose" from "variables the program uses internally for control flow." If `var_name` is `func_table`, this becomes:

```python
global func_table; func_table = <value>
```

And since `value` is dropped into the `exec` string *unquoted* — it's evaluated as a Python expression, not treated as a plain string — I can supply a full string literal (single quotes and all) as long as it contains none of the three blocked characters. A quoted string like `'...win...                     '` doesn't need `;`, `(`, or `)` anywhere in it, so it passes `filter_value` cleanly.

That means I can reassign `func_table` to any 128-character string I want, including one where the `getRandomNumber` slot (or any slot) is replaced with `win`, padded with spaces out to 32 characters to keep `check_table()` happy:

```python
def check_table():
  global func_table
  if( len(func_table) != FUNC_TABLE_ENTRY_SIZE * FUNC_TABLE_SIZE):
    return False
  return True
```

`check_table()` only verifies the *total length* — it has no idea what names are supposed to be in the table, so a forged table with `win` in it looks completely legitimate to the rest of the program.

## Solution

The exploit path, in order:

1. Choose option `4` (`write_variable` is the third table entry, but the accessible route in — reachable from the menu itself via the table — walks through the write path).
2. When prompted for a variable name, enter `func_table`. This passes `filter_var_name` since `func_table` is a legal identifier.
3. When prompted for the new value, enter a quoted string literal reconstructing the table, with the `getRandomNumber` slot replaced by `win`, space-padded to 32 characters (matching the fixed-width slot layout `reset_table()` uses). This passes `filter_value` because a string literal doesn't need `;`, `(`, or `)`.
4. `exec('global func_table; func_table = '+value)` runs, and `func_table`'s fourth slot now reads `win` instead of `getRandomNumber`.
5. Call `call_func(3)` — menu option `4` — which does `get_func(3)` (returns `"win"`) and then `eval('win()')`. The sandboxed dispatcher, trusting its own table, calls `win()` directly.
6. `win()` reads `flag.txt`, strips it, and prints each character as a hex-encoded byte (`hex(ord(c))` per character) rather than as plain text.
7. The hex string decodes (e.g. via CyberChef's "From Hex") straight to the flag:

```
picoCTF{7h15_15_wh47_w3_g37_w17h_u53r5_1n_ch4rg3_a186f9ac}
```

## Retrospective

The biggest lesson here was recognizing where I was pattern-matching the *wrong* prior challenge. Because Picker II was about smuggling a call through `eval`/`exec` directly, I spent a lot of time assuming Picker III's vulnerability had to be some clever bypass of the same two filters — trying to sneak `()` past character-level string checks. Those filters, though, were doing their job correctly the entire time. The real gap was architectural: the program trusted that `func_table` was insulated from user input just because no menu option *directly* exposed it, when in fact `write_variable` — a completely separate, seemingly unrelated feature — had unrestricted write access to *every* global in the module, including the one the author never intended to be user-writable.

That's a good general takeaway for reading source code defensively (or offensively, for that matter): a filter can be airtight for the exact input path it was designed to guard, and still be irrelevant if there's an unrelated code path that reaches the same sensitive state through a different door. `filter_var_name` and `filter_value` were both correctly scoped to prevent *code execution through variable read/write* — but nobody asked whether variable *write* itself should be scoped to a safe subset of names in the first place.

I also want to flag something I initially treated as a dead end but that actually confirmed part of the mechanism: reading `win` as a variable and getting back `<function win at 0x7fbf69676dc0>` wasn't a failure — it was proof that `eval('print(win)')` evaluates `win` as an *expression* referencing the function object, not as a *call*. That distinction (expression vs. call — needing `()` to actually invoke) is exactly what made both filters' focus on `(` and `)` make sense in the first place, and exactly what made forging the table (rather than forging a call) the way through.

Next time I hit a "restricted function table" pattern, my first question is going to be: *what else in this program can write to the table's underlying storage, and is that write path checking the right things?*