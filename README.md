# embedded-boot-time-instrumentation

**© 2026 Matt Chang (matt.chang1118@gmail.com) • https://github.com/mattchang-engr**  
**License: Apache-2.0 (with NOTICE)**  
**Release date: 2026-02-07 11:28 (Taipei, GMT+08:00)**

---

## Summary
**This project implements an intrinsic, early‑boot hardware timer—activated at the PBL/PBI phase prior to Bootloader execution—to quantify end‑to‑end boot latency with microsecond‑level resolution on embedded multicore platforms, validated on the NXP LS1043A‑RDB.**

---

## Overview

This repository shows **how to enable the EL1 Secure Physical Timer at the PBL/PBI stage** on **NXP LS1043A‑RDB** (IFC NOR boot), so timing starts **before U‑Boot**. The approach is portable as a design pattern for early‑boot instrumentation on high‑end SoCs where traditional U‑Boot‑only timing misses critical initialization time.

**Primary audience:** firmware developers, embedded learners, and NXP Layerscape users.  
**Posting:** This project will also be shared on the **NXP Community Forum** (link to be added).

---

## Motivation

### Problem statement
In safety‑sensitive and latency‑sensitive systems, **reboot time is a reliability risk**. Measuring boot only from U‑Boot or OS ignores substantial early‑boot work (reset, RCW load, PLL lock, etc.). We need a **timer that starts as early as possible**—inside the PBL/PBI window—so the measurement reflects **true end‑to‑end** boot latency.

### Technical contribution
- **Earliest‑point enable** of the **EL1 Secure Physical Timer** during **PBI** (right after RCW load).
- Correct handling of **endianness** for the CNTCR payload at this stage so the timer actually starts.
- Use of **ALTCBAR** to open the **alternate CCSR window** (0x02xx_xxxx) required to reach the Secure System Counter block.
- A concise, reproducible **pair of PBI commands** that (1) program ALTCBAR, then (2) enable CNTCR.
- Brief validation with **UART samples** showing staged times and an example full boot of ~**7.947 s**.

*(The motivation and validation are distilled from Matt’s science project report.)*

---

## System Architecture (top‑to‑bottom)

```
+-------------------------------+
| T0: System Reset / Power-On   |
+-------------------------------+
              |
              v
+-------------------------------+
| PBL: Read PBL from IFC NOR    |
|      Load RCW (includes PBI   |
|      source selection)        |
+-------------------------------+
              |
              v
+-------------------------------+
| Platform clocks:              |
| - PLL lock                    |
| - Clock switching             |
+-------------------------------+
              |
              v
+-------------------------------+
| PBI phase (first programmable |
| point after RCW):             |
|   1) Write ALTCBAR            |
|   2) Write CNTCR (EN/CNTCLKEN)|
| -> EL1 Secure timer starts    |
+-------------------------------+
              |
              v
+-------------------------------+
| System Ready (reset complete; |
| SoC released from reset)      |
+-------------------------------+
              |
              v
+-------------------------------+
| U-Boot                        |
|  T1: Console init             |
|  T2: Board checks complete    |
|  T3: U-Boot boot complete     |
+-------------------------------+
              |
              v
+-------------------------------+
| Linux boot (kernel/userspace) |
+-------------------------------+
```

> The terms **PBL**, **PBI**, **RCW**, and **CCSR** in the next section refer back to this diagram.

---

## Hardware Test Environment (top‑to‑bottom)

```
+----------------------------+
| Ubuntu 20.04 Host          |
| - LSDK tools / TFTP server |
| - UART console @115200-8N1 |
| - IP: 192.168.50.114       |
+----------------------------+
              |
              |  Ethernet
              v
+----------------------------+
| Router / Switch            |
| IP: 192.168.50.1           |
+----------------------------+
              |
              |  Ethernet
              v
+----------------------------+
| LS1043A-RDB (IFC NOR boot) |
| UART1 console              |
| Board IP: 192.168.50.112   |
+----------------------------+
```

- **Board / SoC:** LS1043A‑RDB, **SoC stepping:** from boot log → `LS1043AE Rev1.0 (0x87920010)`  
- **Environment:** Ubuntu 20.04, LSDK 21.08 (FlexBuild), U‑Boot 2021.04

---

## Background (brief)

- **PBL (Pre‑Boot Loader):** Early ROM logic that fetches the **PBL structure + RCW** from the boot source (IFC NOR here).
- **PBI (Pre‑Boot Initialization):** Micro‑ops executed **immediately after** RCW load—**earliest programmable point** to touch SoC registers.
- **RCW (Reset Configuration Word):** Encodes reset‑time options (e.g., `PBI_SRC`); consumed by PBL and mirrored into CCSR for visibility.
- **CCSR (Configuration/Control/Status Registers):** SoC‑internal register space; includes **SCFG/ALTCBAR** and **Secure System Counter**.

*(All four map to the steps shown in the System Architecture diagram.)*

---

## Deep Dive: ALTCBAR (Alternate Configuration Base Address Register)

**Why ALTCBAR is needed**  
During PBL/PBI, the SoC exposes **two CCSR windows**:

- **Default Configuration Space** (base ≈ `0x0100_0000`)  
- **Alternate Configuration Space** (base **defined by ALTCBAR**)

Many early‑boot registers (including the **Secure System Counter** where **CNTCR** lives at `0x02B0_0000`) are **outside** the default CCSR window. To reach them from a **PBI command**, you must:

1) **Program ALTCBAR** (from the default window) to define the **alternate** CCSR base; then  
2) Issue subsequent PBI writes with **`ACS=1`** so the PBI engine uses the **alternate base** + `SYS_ADDR` offset.

### What ALTCBAR actually does

- **Register:** `ALTCBAR` (Alternate Configuration Base Address Register)  
- **Address (in default CCSR):** `0x0157_0158` (so you can reach it with `ACS=0`)  
- **Programmed value (this project):** `0x0000_0200`  

When you write **`ALTCBAR = 0x0000_0200`**, you are telling the PBI engine:

> “When a PBI header sets **`ACS=1`**, interpret the 24‑bit `SYS_ADDR` as an **offset** from **`0x0200_0000`** (the alternate CCSR base).”

That creates an **alternate CCSR window** of `0x0200_0000 .. 0x02FF_FFFF`.  
From there, you can reach **CNTCR @ `0x02B0_0000`** by using **ACS=1** and **`SYS_ADDR = 0x00B0_0000`**.

### How PBI selects which base to use

```
PBI header (1 byte) encodes: [ACS | BYTE_CNT | CONT]  (plus 3 bytes of SYS_ADDR)
    - ACS=0  → use Default CCSR Base (≈ 0x0100_0000)
    - ACS=1  → use Alternate CCSR Base (  0x0200_0000 .. set by ALTCBAR)

Effective target address = (ACS ? ALT_BASE : DEF_BASE) + SYS_ADDR
```

**In this project:**

- **Step 1 (ACS=0):** Write `ALTCBAR` from **default** base  
  - Header: `0x0957_0158`  → `ACS=0`, `BYTE_CNT=4`, `CONT=1`, `SYS_ADDR=0x01570158`  
  - Data:   `0x0000_0200`  → Set **ALT_BASE = 0x0200_0000`

- **Step 2 (ACS=1):** Write `CNTCR` from **alternate** base  
  - Header: `0x89B0_0000`  → `ACS=1`, `BYTE_CNT=4`, `CONT=1`, `SYS_ADDR=0x00B0_0000`  
  - Data:   `0x0100_0000`  → **byte‑swapped payload** so the register reads `0x0000_0001`

### Visual: base selection and offsets (≤80 cols)

```
Default CCSR window (ACS=0)                Alternate CCSR window (ACS=1)
Base: 0x0100_0000                          Base: 0x0200_0000  (set by ALTCBAR)
       |                                            |
       +--[SYS_ADDR=0x01570158]--> ALTCBAR          +--[SYS_ADDR=0x00B0_0000]--> CNTCR
                                                   (ALT_BASE + 0x00B0_0000 = 0x02B0_0000)
```

### Ordering, scope, and lifetime

- **Order matters:** Program **ALTCBAR first** (ACS=0), then issue any **ACS=1** writes.  
- **Scope:** ALTCBAR configures how the **PBI engine** resolves ACS=1 **during PBL execution**.  
- **Lifetime:** The alternate mapping remains valid for the remainder of PBI execution. After hand‑off to
  later stages (e.g., U‑Boot/OS), those stages use their own MMU/driver views of the address space.

### Common mistakes (and how ALTCBAR prevents them)

1) **Targeting CNTCR with ACS=0**  
   - Result: You write into the **wrong address range** (default base), and CNTCR never changes.  
   - **Fix:** Write **ALTCBAR** first (ACS=0), then write CNTCR with **ACS=1**.

2) **Programming the wrong ALTCBAR value**  
   - If you program `ALTCBAR=0x0000_0100` (or forget a zero), **CNTCR won’t be in the window**.  
   - **Fix:** Use **`0x0000_0200`**, which yields **ALT_BASE = `0x0200_0000`** so CNTCR is reachable at
     `ALT_BASE + 0x00B0_0000 = 0x02B0_0000`.

### Endianness reminder (ALTCBAR vs. payload)

- The **PBI header and SYS_ADDR fields** are consumed as a defined **byte stream** by the PBL logic—use the
  canonical encodings shown above.  
- The **BYTEn payload** for **CNTCR** must be **byte‑swapped at this stage** so the **register value** becomes
  `0x0000_0001`. That’s why you write **`0x0100_0000`** in the CNTCR PBI **data** field.

### Quick checklist for ALTCBAR use

- ✅ Write **ALTCBAR = `0x0000_0200`** with **ACS=0** (default base).  
- ✅ Then write **CNTCR** at `SYS_ADDR=0x00B0_0000` with **ACS=1**.  
- ✅ Use **BYTEn = `0x0100_0000`** so CNTCR reads **`0x0000_0001`** (EN/CNTCLKEN=1).  
- ✅ Keep all PBI encodings ≤80 cols in the README for readability.

---

## Timer Mechanics Summary: CNTCR, CNTCLKEN, Endianness

**Goal:** Start the **EL1 Secure Physical Timer** as soon as possible—**inside PBI**—so it measures the entire boot, not just U‑Boot/OS.

- **CNTCR (Secure System Counter @ 0x02B0_0000)**  
  **Enable semantics**: document **CNTCR.EN (aka CNTCLKEN)** → **bit 0 = 1** enables the counter.  
  *Desired value in the register:* `0x0000_0001`.

- **Why BYTEn = `0x0100_0000` (endianness)?**  
  At the PBI path/phase, the byte ordering for the **payload** must be **swapped** so that the **target register** ends up as `0x0000_0001`.  
  That’s why the PBI data for the CNTCR write is **`0x0100_0000`** even though the register must read `0x0000_0001`.

---

## PBI Command Encoding (two writes)

**1) Program ALTCBAR (ACS=0, default CCSR window)**

- **Header:** `0x0957_0158`  
  - `ACS=0`, `BYTE_CNT=4`, `CONT=1`, `SYS_ADDR=0x01570158`  
- **Data:** `0x0000_0200`  
  - Sets **ALTCBAR = 0x0000_0200** (alt base at 0x0200_0000)

**2) Enable CNTCR (ACS=1, alternate CCSR window @ 0x02B0_0000)**

- **Header:** `0x89B0_0000`  
  - `ACS=1`, `BYTE_CNT=4`, `CONT=1`, `SYS_ADDR=0x02B00000`  
- **Data:** `0x0100_0000`  
  - **Byte‑swapped payload** so the **register reads 0x0000_0001** (EN/CNTCLKEN=1)

> **Field primer (compact):**  
> `ACS` selects default(0)/alternate(1) CCSR base; `BYTE_CNT=4` for 32‑bit writes; `CONT=1` continues;  
> `SYS_ADDRn` is the 24‑bit offset from the chosen base; `BYTEn` are the write payload bytes.

---

## Common Pitfalls (quick)

- **Endianness at PBL/PBI**  
  Writing BYTEn as `0x0000_0001` keeps the timer **disabled**.  
  **Fix:** write **`0x0100_0000`** so the register value becomes `0x0000_0001`.

- **Wrong window / ACS mismatch**  
  Writing CNTCR with **ACS=0** or before ALTCBAR is programmed targets the **wrong region**.  
  **Fix:** write **ALTCBAR first** (ACS=0), then write **CNTCR with ACS=1**.

---

## Validation (brief UART samples)

```
U-Boot 2021.04-dirty (Mar 12 2024 - 11:51:50 +0800)
SoC: LS1043AE Rev1.0 (0x87920010)
Project: PBI start to u-boot console init: 757 ms
Project: u-boot check board complete: 820 ms
Project: u-boot boot process complete: 1270 ms
...
[ 6.673863] random: crng init done
TinyLinux login: root
```

**Worked example (context):** with these samples plus Linux bring‑up, total boot time observed ≈ **7.947 s**.

---

## License

This project is licensed under the **Apache License 2.0**.

**README banner (top):**
```
**embedded-boot-time-instrumentation**
© 2026 Matt Chang (matt.chang1118@gmail.com) • https://github.com/mattchang-engr
License: Apache-2.0 (with NOTICE)
Release date: 2026-02-07 11:28 (Taipei, GMT+08:00)
```

**SPDX header for source/snippets:**
```c
/* SPDX-License-Identifier: Apache-2.0
 * embedded-boot-time-instrumentation
 * © 2026 Matt Chang <matt.chang1118@gmail.com> (https://github.com/mattchang-engr)
 */
```

**NOTICE (short):**
```
embedded-boot-time-instrumentation
© 2026 Matt Chang

This project references public vendor documentation (e.g., NXP LSDK, Arm TRMs, U-Boot).
No proprietary documentation is redistributed here. Consult vendor sites for licenses.
```

---

## Attribution & Community

This repository references public vendor documentation (NXP LSDK, Arm TRMs, U‑Boot).  
No proprietary materials are redistributed here.  
A discussion thread will be posted on the **NXP Community Forum** (link TBA).

---

## Author

**Matt Chang** • https://github.com/mattchang-engr  
Contact: **matt.chang1118@gmail.com**
