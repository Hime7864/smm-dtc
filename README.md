# About

This project is my attempt to detect the presence of SMM (System Management Mode) abuse from a kernel driver by leveraging the interplay of #SMI ↔ #NMI on AMD and Intel systems.

A few months ago, I got a message from a friend telling me he was cheating in FACEIT. When I asked what he was doing, he told me it was this new thing called an "SMM cheat". From all my free time reading the APM/PPR, this should be impossible for many reasons. For one, ASP, TPM, and Secure Boot should all catch this — but they don't. This led me down the path of figuring out how they were actually doing it. The short version: motherboard manufacturers shipped slop code.

## Why this problem exists
SMM is intentionally opaque to the rest of the system:
- **SmmLock** Typically set early in DXE, this bit locks SMRAM so it can no longer be read from or written to from outside of SMM.
- **SMRAM** This is just normal DRAM that has been marked as SMRAM (via SMM_BASE and SMM_MASK). The CPU can only access it via #SMI. Outside of SMM, DMA can also access this range.

Motherboard Manufacutre's incompetence:
- **TPM 2.0** Either doesn't work properly or PCRs 0-7 are garbage.
- **Secureboot** Outside of the OS loader, virtually no images executing in DXE are validated.

So if malicious software is implanted into the SMM handler, you're left with heuristics or directly calling the handler yourself. In either case, the solution is a highly platform tailored approach

---

## Core idea
The x86 interrupt priority hierarchy is well-defined: Machine Check (MC) > System Management Interrupt (SMI) > Non-Maskable Interrupt (NMI) > Maskable Interrupt (INTR).
When a higher-priority interrupt arrives while a lower-priority interrupt handler is executing, the processor immediately preempts the current handler to service the higher-priority one.
This preemption behavior can be exploited to detect the servicing of the SMI from within an NMI handler. The technique involves creating an SMI honeypot using a PAUSE loop, while monitoring the instruction retirement counts via MSR 0xC00000E9 (IRPerfCount). Since this counter is read-only outside of SVM and cannot be directly modified, the only control available is enabling or disabling it via MSR 0xC0010015 (IRPerfEn). As a result, any unexpected delta in the counter value reliably indicates that an SMI has occurred.
This detection method is effective on processors that expose an instructions-retired counter (AMD `IRPerfCount`, Intel fixed-counter `INST_RETIRED`), provided the expected instruction delta is calibrated for each manufacturer and major firmware revision.

---

## AMD vs Intel

The NMI honeypot logic is the same on both vendors: spin in a `PAUSE` loop inside an NMI handler and compare instructions retired before and after. Vendor-specific code is limited to how that counter is discovered, enabled, and read.

| | AMD | Intel |
|---|---|---|
| Feature check | CPUID `0x80000008`, `InstRetCntMsr` | CPUID `0x0A`, version > 0 |
| Counter | `MSR 0xC00000E9` (`IRPerfCount`) | `IA32_FIXED_CTR0` (`INST_RETIRED`) |
| Enable | `MSR 0xC0010015` (`HWCR.IRPerfEn`) | `IA32_FIXED_CTR_CTRL` + `IA32_PERF_GLOBAL_CTRL` |
| Per-core context | Indexed by extended local APIC ID (CPUID `0x0B`) | Indexed by processor number |
| NMI callback extras | — | `mfence`/`lfence` around counter reads; scores within ±10 of the calibrated delta are treated as zero (Intel’s retired-count signal is noisier than AMD’s) |

`DriverEntry` dispatches to `AMD::Service()` or `Intel::Service()` based on the CPU vendor string from CPUID leaf 0.

---

## Tested systems

Validated on these machines:

`note: delta is the min and max per NMI, detection uses lowest of a batch of 50 NMIs per core`

**Desktop (AMD)** - uncompromised
- Processor: AMD Ryzen 7 9800X3D (8 cores, 16 logical processors @ 4700 MHz)  
- Baseboard: ROG STRIX B850-F GAMING WIFI  
- delta ~20000 +- 12500

**Laptop (AMD)** - uncompromised
- Processor: AMD Ryzen 9 270 w/ Radeon 780M Graphics, 4001 Mhz, 8 Core(s), 16 Logical Processor(s)
- Baseboard: GA403UM
- delta ~20000 +- 12500

**Laptop (Intel)** - uncompromised
- Processor: Intel Core Ultra 9 275HX (24 cores, 24 logical processors @ 2700 MHz)  
- Baseboard: LNVNB161216  
- delta 1.5m - 400k (laptop are not a concern for smm abuse because of the lack of a flash-back button & pc unuseable at this delta)

**Desktop (Intel)** - uncompromised
- Processor: 13th Gen Intel(R) Core(TM) i5-13600KF, 3500 Mhz, 14 Core(s), 20 Logical Processor(s)  
- Baseboard: B760M GAMING WIFI6 PLUS GEN5  
- delta ~25000 +- 20000

## Final Notes

This code was tested via my own driver base using llvm, the time value of 15014/12 in the nmi hander will need to be change when compiled under WDK. why I don't do this myself? I hate so many of the design choices of WDK that I took it upon myself to rewrite most of the parts I use from WDK and part that arn't supported e.g. all the unsafe stuff msdn dosn't have documented and amd things too
