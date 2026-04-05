# 🧠 Write-up — deepwash (Web CTF)

## 🎯 Objective

The goal of this challenge is to send a POST parameter `x` composed of **exactly 3 lines**, such that:

* It passes strict input validation
* It is parsed using `DateTimeImmutable::createFromFormat`
* It satisfies:

  * A specific `sha256`
  * A specific `md5` on `$y`
  * A specific `md5` on `$z`

If all conditions are met → the server returns the flag.

---

## 🔍 Source Code Analysis

The input is parsed as follows:

```php
$p = a('!D d M Y', $l[0]);
$q = a('!Y?z', $l[1]);
$r = a('!U H', $l[2]);
```

Then two arrays are constructed:

```php
$y = [
    $p->format('Y-m-d'),
    $q->format('Y-m-d'),
    $r->format('U'),
];

$z = [
    $p->format('D'),
    $q->format('z'),
    $r->format('H:i:s'),
];
```

Final checks:

```php
md5(json_encode($y)) == 779d88604518d4528cec1539c8ce5fa0
md5(json_encode($z)) == 156e4ca092477c6c57a3cf114acf08fd
sha256(json_encode(lines)) == dca58f177da7427d37b7030b14d77995ccf105051c14f6e5e1ca071e5cc19c93
```

---

## 💣 Key Vulnerability

The challenge relies on the behavior of:

```php
DateTimeImmutable::createFromFormat(...)
```

Important observations:

* It accepts **invalid values**
* It applies **silent normalization (overflow)**
* It does **NOT reject malformed inputs**
* No error checking is performed

---

## ⚠️ Exploitable Behaviors

### Day-of-year overflow

```
2021 366 → 2022-01-02
```

### Hour overflow

```
1609459200 24 → next day 00:00:00
```

### Invalid day accepted

```
Fri 00 Jan 2021 → 2021-01-01
```

---

## 🎯 Target Constraint

From the hash of `$z`, we deduce:

```
["Fri", "1", "00:00:00"]
```

So we need:

* Day = Fri
* Day of year = 1
* Time = 00:00:00

---

## 🧠 Payload Construction

### Line 3 (`!U H`)

```
3600 00
```

* 3600 = 01:00:00
* H=00 forces the hour → becomes 00:00:00
* Overflow normalizes the timestamp

---

### Line 2 (`!Y?z`)

```
2022x366
```

* z=366 overflows into the next year
* becomes 2023-01-02
* day-of-year = 1

---

### Line 1 (`!D d M Y`)

```
Fri 19 November 2011
```

* Adjusted by PHP to a valid Friday
* becomes 2011-11-25

---

## ✅ Final Payload

```
Fri 19 November 2011
2022x366
3600 00
```

---

## 🚀 Exploit

```bash
curl -s -X POST "http://34.175.114.248/" \
  --data-urlencode $'x=Fri 19 November 2011\n2022x366\n3600 00'
```

---

## 🏁 Flag

```
CITEFLAG{b37a508f1b6a4a49a9f2420f8fc23f61}
```

---

## 💡 Conclusion

This challenge is not about hash cracking, but about:

* Understanding PHP date parsing quirks
* Exploiting silent normalization (overflow)
* Constructing a valid payload logically

> Whenever you see `DateTime::createFromFormat` in CTFs, always consider overflows and parser inconsistencies.
