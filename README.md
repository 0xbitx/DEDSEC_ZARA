
<p align="center">
<img src="https://media3.giphy.com/media/v1.Y2lkPTc5MGI3NjExemFreHN5MnRlYzIybHh0dXlpNWl4Z3VmMXd3ZGphMTYwcjUyNWNociZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/KAGxX1xU8PwGGfki8v/giphy.gif", width="400", height="400">
</p>

<h1 align="center">ZARA</h1>
<p align="center"><code>Advanced Python Malware Obfuscator & Binary Packer for Evasion-Focused Red-Team and Malware-Development</code></p>

---

## DESCRIPTION

ZARA is a multi-stage Python source-to-source obfuscator built for
malware-development research. It takes readable Python (implants, loaders,
post-exploitation agents, keyloggers, droppers, C2 stagers, and persistence
scripts) and turns it into an equivalent but heavily obfuscated form that is
significantly harder to statically analyze, reverse engineer, or reuse. The
obfuscated source is then always compiled into a standalone native ELF binary
using Nuitka, and finally packed and anti-decompression-hardened with UPX. There
is no plain Python output, the end result is a compiled executable.

The project is built around a simple premise: **Python-based malware is only as
stealthy as its source code.** Python payloads ship their logic and a large part
of their intent in plain, recoverable text. Readable identifiers, hardcoded
command strings, obvious imports, and self-describing builtin calls all act as
high-signal breadcrumbs that give away a payload's purpose within seconds of
inspection. This obfuscator exists to systematically strip those breadcrumbs
away, studying and reproducing the exact evasion techniques that have made
Python a persistent and frequently underestimated threat in the wild.

Every transformation in the pipeline is deliberately chosen to target a specific
analysis surface that incident responders, AV/EDR engines, sandboxes, and manual
reverse engineers rely on to triage suspicious code:

- **Readable variable, function, and class names**. Semantic names like
  `download_and_execute`, `steal_cookies`, or `keylog_buffer` instantly expose
  intent. The renamer collapses them into indistinguishable Unicode glyphs (or
  invisible zero-width identifiers) so capability becomes opaque.
- **Plaintext string literals**. C2 URLs, API keys, shell commands, file paths,
  registry entries, user-agents, and mutex names are the single highest-value
  signal for both automated `strings` extraction and manual analysis. The string
  encoder hides them behind runtime-decoded payloads.
- **Direct `import` statements**. `import socket` implies networking, `import
  subprocess` implies command execution, and `import ctypes` implies native API
  access. The import rewrite removes this capability fingerprint by deferring
  resolution to obfuscated `__import__()` calls.
- **Obvious builtin usage**. Calls like `exec(...)`, `eval(...)`, `open(...)`,
  `subprocess.run(...)`, and `socket.connect(...)` are the most dangerous API
  surface a payload exposes. Builtin aliasing replaces these self-describing
  names with generated identifiers that no longer match static signatures.
- **Recoverable numeric constants**. Connection ports, sleep/jitter intervals,
  buffer sizes, magic values, and XOR keys are trivial to extract and correlate.
  Integer and float obfuscation rewrites them as runtime-evaluated expressions.
- **Excess structure**. Comments, clean f-strings, and predictable formatting
  all aid a human analyst tracing control flow. The framework strips comments,
  normalizes f-strings into concatenation trees, and injects opaque predicates
  and dead code to deliberately obscure logic.

Together these stages convert a self-documenting Python implant into a
purpose-built, analysis-resistant artifact: a payload whose strings are invisible
until execution, whose imports and dangerous calls no longer answer to grep, and
whose numeric constants no longer sit in plain sight. Nuitka then compiles that
obfuscated source into a standalone ELF binary, removing the Python source (and
any recoverable `.pyc` bytes) from the equation entirely, and UPX packs the
result to further frustrate static unpacking and reverse engineering.

> This is a **research and defense-orientation** project. It models real evasion
> techniques so analysts can recognize them, engineers can build detections, and
> researchers can generate realistic test samples.

---

## Threat Model & Purpose

This tool exists to model the obfuscation techniques commonly seen in Python-based
malware so that researchers can:

- **Build realistic samples** for detection engineering and sandbox testing.
- **Validate detections** against obfuscated Python and packed binaries.
- **Study evasion techniques** (string encoding, import hiding, name mangling,
  opaque predicates, packing).
- **Train reverse engineers** on unpacking and deobfuscating real-world payloads.

It is a **source-to-source** transformation followed by a **source-to-native**
compilation stage. The obfuscation is runtime-decoded, meaning the original logic
is only reconstructed in memory when the payload executes, a deliberate trade-off
intended to defeat static analysis.

---

## Features

### Core Obfuscation

- **Variable / function / class renaming**: identifiers rewritten to Unicode
  glyphs from 12 scripts (Chinese, Japanese, Korean, Greek, Cyrillic, Arabic,
  Thai, Ethiopic, Runic, Braille). Invisible zero-width names are also available.
- **String obfuscation**: 12 encoding modes, including zero-width invisible
  characters and XOR, to hide C2 URLs, paths, commands, and secrets from `strings`
  and static scanners.
- **Integer & float obfuscation**: numeric literals (ports, timeouts, IDs)
  rewritten as runtime decoder calls so they don't appear verbatim.
- **Import obfuscation**: every `import` / `from ... import` converted into a
  dynamic `__import__()` call with an encoded module name, hiding capabilities.
- **Builtin aliasing**: `exec`, `eval`, `open`, `subprocess`, `socket`, `input`,
  `range`, and other builtins replaced with obfuscated aliases.
- **F-string conversion**: f-strings normalized into concatenation trees so
  their interpolated contents can be obfuscated.
- **List / dict / set obfuscation**: literal collections encoded element-wise.

### Anti-Reverse Engineering

- **Opaque predicates**: injected branches that evaluate to a fixed
  `True`/`False`, hiding dead code and confusing control-flow recovery.
- **Integer splitting**: simple integers expanded into arithmetic expressions.
- **Indirect calls**: call sites wrapped to obscure the callee target.
- **Anti-debug**: `sys.gettrace()` checks injected at entry to detect debuggers.

### Binary Compilation & Packing

- **Nuitka integration**: compiles obfuscated Python into standalone ELF binaries.
- **UPX compression**: auto-detected and applied, with optional signature
  stripping to break naive `strings` inspection and reduce size.
- **Anti-decompression**: corrupts the UPX trailer magic so `upx -d` fails.
- **Disguised build directory**: builds occur inside an invisible-character path
  to prevent path leakage into the binary.
- **Standalone vs light mode**: onefile standalone, or `--static-libpython` with
  Clang + LTO for much smaller payloads.

| Anti-decompression |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/56e6fdfa-1c8d-4651-8074-4228f47df2b4) |

### Usability & Workflow

- **Interactive menu**: 3 preset modes (MAXIMUM to LIGHT).
- **Skip list**: keep select values (keys, URLs) verbatim for testing or
  specific study scenarios.
- **Comment removal**: strips disassembly-aiding comments while preserving strings.
- **Regex-aware processing**: protects regex patterns from being mangled.
- **Spacing normalization**: cleans up `ast.unparse()` output.
- **Loader animations**: progress indicator during compilation.
- **Ctrl+C safe**: graceful exit at any prompt.

---

## Installation

```bash
git clone https://github.com/0xbitx/DEDSEC_ZARA.git
cd DEDSEC_ZARA
sudo pip install tabulate nuitka
sudo apt install upx-ucl
sudo apt install clang
sudo ./dedsec_zara
```

Preset modes:

| # | Mode | Description |
|---|---------------|----------------------------------------------|
| 1 | **MAXIMUM** | Standalone onefile + Compress |
| 2 | **MEDIUM** | Light compile + Compress |
| 3 | **LIGHT** | Light compile, not Compress |

#### Step-by-step flow

1. Select a mode (`1`–`3`), or `0` to exit.
2. Choose an **encoding language** (`1`–`12`).
3. Enter the **input file path** (must end in `.py`).
4. *(Optional)* enter a **skip list** of variable names whose values should not
   be obfuscated (comma-separated, e.g. `c2_url, xor_key, mutex`).
5. The tool processes the file and writes `output`.

---

## Encoding Languages

Obfuscated identifiers and encoded payloads can be generated from 12 Unicode
scripts:

| # | Language | Visual Style |
|---|-----------|--------------|
| 1 | Chinese | 汉字 |
| 2 | Japanese | ひらがなカタカナ |
| 3 | Korean | 한글 |
| 4 | Greek | αβγ |
| 5 | **Invisible** | Zero-width characters |
| 6 | XOR | Hexadecimal encoding |
| 7 | Cyrillic | абв |
| 8 | Arabic | عربي |
| 9 | Thai | ไทย |
| 10 | Ethiopic | ሀለሐ |
| 11 | Runic | ᚠᚢᚦ |
| 12 | Braille | ⠁⠃⠉ (strings only) |

---

## Skip Value Obfuscation (Exclusion List)

Malware-development research often requires comparing obfuscated vs.
non-obfuscated samples, or keeping a specific indicator (a C2 URL, key, or magic
value) visible for detection testing.

The **skip list** lets you name variables whose *assigned literal values* are left
untouched by string / integer / float obfuscation.

## Disassembled Binary: Side-by-Side Comparison (Original vs. Obfuscated)
| Screenshot |
|-------|
| ![screenshot](https://github.com/user-attachments/assets/c9490eca-d263-4e28-be52-a7c60f55e834) |

### How to use it

```text
 [?] SKIP VALUE OBFUSCATION variables (comma-separated, optional): c2_url, xor_key, mutex
```

Separate names with commas (spaces are trimmed). Leave it blank to obfuscate
everything.

### Behavior

```python
c2_url   = "https://example.com/beacon"  # kept in plain text
xor_key  = 0x5A                          # kept in plain text
payload  = "base64_of_shellcode"         # obfuscated
interval = 60                            # obfuscated (unless ignored by other rules)
```

With a skip list of `c2_url, xor_key`, the values of those two variables stay
verbatim, while `payload` and `interval` are still obfuscated.

Composite assignments are handled recursively: if a skipped variable is assigned a
list, tuple, set, or dict, each literal element / value inside is preserved.

### Important notes

- This only affects **value obfuscation**. It does **not** stop the variable
  *name* from being renamed.
- Values already excluded by built-in safety heuristics (common function
  arguments, small ints, common floats, regex patterns, `case` values, `range()`
  bounds, subscripts) remain excluded regardless of the skip list.

---

## How It Works

### Runtime-Decoded Strings

The tool avoids storing plaintext strings by embedding a hidden decoder function
named `ᅟ` (U+115F, Hangul Choseong Filler, invisible in most terminals). All
strings, integers, and floats are rewritten as calls to this decoder, which
reconstructs the original values only when the payload executes:

```python
# Conceptual form (decoder body is generated per-language):
value = ᅟ("⟨encoded payload⟩")
```

This defeats static `strings` extraction and naive pattern matching against URLs,
commands, and keys. The values exist only in plaintext at runtime in memory.

### Invisible Mode (Language 5)

Uses zero-width characters (U+200C ZWNJ and U+200D ZWJ) to bit-encode strings. The
source file appears to contain empty strings but decodes correctly at runtime.

### Import Hiding

Imports leak capability (e.g. `import socket` → networking, `import subprocess` →
command execution). This stage rewrites them to hide that signal:

```python
# Before:
import os
from ctypes import windll

# After (conceptual):
ᅟ_1 = __import__(ᅟ("⟨os⟩"), fromlist=[ᅟ("⟨...⟩")])
ᅟ_2 = getattr(__import__(ᅟ("⟨ctypes⟩"), fromlist=[ᅟ("⟨...⟩")]), ᅟ("⟨windll⟩"))
```

### Builtin Aliasing

Obvious calls like `exec(...)`, `eval(...)`, `open(...)`, or `print(...)` are
replaced with generated aliases so dangerous API usage is no longer searchable by
name.

### Anti-Reverse

- **Opaque predicates** wrap code in always-true / always-false conditions and
  inject dead branches, confounding control-flow reconstruction.
- **Integer splitting** expands constants into arithmetic expressions.
- **Indirect calls** obscure callee targets.
- **Anti-debug** injects `sys.gettrace()` checks at entry.

### Binary Compilation & Packing

- **Standalone mode**: `--standalone --onefile` → single executable.
- **Light mode**: `--static-libpython --clang --lto` → smaller payload.
- **UPX**: auto-detected and applied; optional signature stripping to hinder
  `strings` and static unpacking.
- **Path hiding**: builds in an invisible-character directory to avoid leaking
  build paths into the binary.

---

## Output Examples

### String Obfuscation (Korean mode)

```python
# Original: print("Hello World")
칛쇅훛(f'{쇸뭨("걁겭갆걁")} {쇸뭨("걂겪겳걂겪갸")}')
```

### Import Obfuscation (Invisible mode)

```python
薇鯑旋疂 = __import__(ᅟ(""), fromlist=[ᅟ("")])
# The "empty" strings contain invisible zero-width characters.
```

---

## Requirements

| Dependency | Purpose | Install |
|-----------|---------|---------|
| Python 3.8+ | Runtime | `apt install python3` |
| `tabulate` | Table formatting | `pip install tabulate` |
| Nuitka | Binary compilation | `pip install nuitka` |
| UPX | Binary packing | `apt install upx-ucl` |
| Clang | Light mode LTO | `apt install clang` |

---

## Tested On

- Kali Linux
- Parrot OS
- Ubuntu

---

## Legal Disclaimer

This tool is intended for **educational and security research purposes only**.
Unauthorized access to computer systems is illegal. The author is not
responsible for any misuse of this tool. Only use on systems you own or have
explicit permission to test.
