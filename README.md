# ratpak for ESP32

ratpak is the arbitrary-precision rational arithmetic library of [Microsoft Calculator](https://github.com/microsoft/calculator) (`src/CalcManager/Ratpack`), packaged as an ESP-IDF component.

The sources are kept as close to upstream as possible. Every change is listed below, and every non-mechanical change is marked in the code with an `ESP32 port:` comment.

- Upstream base: microsoft/calculator `d125246` (2026-09-28). The last upstream ratpak commit included is `ffd0519` (2025-12-23).
- License: MIT, see [LICENSE](LICENSE) (Microsoft's original license). The ESP32 changes are released under the same license.

## Layout

The folder mirrors upstream so that all upstream include paths work unchanged:

```
ratpak_esp32/
├── CMakeLists.txt              ESP-IDF component
├── idf_component.yml
├── .clang-format               disables reformatting of the sources
├── LICENSE
├── sal_cross_platform.h        verbatim from src/CalcManager/
├── winerror_cross_platform.h   verbatim from src/CalcManager/
└── Ratpack/                    from src/CalcManager/Ratpack/
```

## Usage

Copy or clone the folder into `components/ratpak` of an ESP-IDF project, or reference it as a path dependency. Then add `REQUIRES ratpak` to the consuming component and `#include <ratpak.h>`.

C++ exceptions must be enabled (`CONFIG_COMPILER_CXX_EXCEPTIONS=y`), because ratpak reports errors by throwing `uint32_t` error codes (see `Ratpack/CalcErr.h`).

## Unchanged files

`CalcErr.h`, `basex.cpp`, `itrans.cpp`, `itransh.cpp`, `logic.cpp`, `num.cpp`, `ratconst.h`, `trans.cpp`, `transh.cpp`, `sal_cross_platform.h`, `winerror_cross_platform.h`.

## Build settings instead of source changes

In `CMakeLists.txt`:

- `-Wno-unknown-pragmas`: MSVC `#pragma warning` in `ratpak.h`. ESP-IDF builds with `-Werror=all`, which would turn this warning into an error.
- `-Wno-switch`: `sinanglerat`/`cosanglerat`/`tananglerat` in `trans.cpp` switch over `AngleType` without a `Radians` case.

## Changes to the sources

### 1. Narrow strings (`conv.cpp`, `support.cpp`, `ratpak.h`)

The string API uses `char` / `std::string` / `std::string_view` instead of `wchar_t` / `std::wstring` / `std::wstring_view`. Embedded applications work with narrow strings, and `wchar_t` is 4 bytes with GCC.

This change is purely mechanical and can be re-applied to a newer upstream with these replacements:

| Upstream | Port |
|---|---|
| `wstring_view` | `string_view` |
| `wstring` | `string` |
| `wchar_t` | `char` |
| `wostream`, `wcout` | `ostream`, `cout` |
| `towupper` | `toupper` (plus `#include <cctype>` in `conv.cpp`) |
| `L'x'`, `L"…"` | `'x'`, `"…"` |

Not applied to the `DEBUG_RATPAK` block in `ratpak.h`. It is Windows-only debug code (`OutputDebugString`) and is not compiled unless `DEBUG_RATPAK` is defined.

### 2. CPU detection (`conv.cpp`)

`__XTENSA__` and `__riscv` are added to the 32-bit branch of the `Calc_UInt32x32To64` architecture check. Without them, upstream stops with `#error Must define a target architecture.`

### 3. `<cmath>` include (`support.cpp`)

`ceil` and `log2` are used without an include. Upstream gets them indirectly on Windows; the ESP-IDF toolchain doesn't.

### 4. Memory leaks on exception paths (`rat.cpp`, `exp.cpp`, `fact.cpp`)

Temporaries were not freed when a called function throws. On a device with a small heap that runs for months, these leaks add up.

| Function | Fix |
|---|---|
| `rootrat` | free `oneovern` if `powrat` throws |
| `powratNumeratorDenominator` | free temporaries if any of the three `powratcomp` calls throws |
| `powratcomp` | free `iy`, `pxint`, `podd` if `rattoi32` throws; free `podd`, `pxint` before throwing for a bad exponent |
| `_gamma` | free temporaries if any of the three `powratcomp` calls throws; **`ratRadix` and `mpy` also leaked on every normal call** (`ratRadix` is no longer `const` so it can be destroyed) |
| `factrat` | free `fact`, `frac`, `neg_rat_one` before throwing for negative integers |

Not covered: other calls in `_gamma` that can throw (for example `rattoi32` when the precision bump is computed) still leak their temporaries, as upstream.

### 5. `ratpowi32` with `INT32_MIN` (`conv.cpp`)

Upstream handles negative powers by calling itself with `-power`. For `INT32_MIN`, `-power` is `INT32_MIN` again and the function recurses until the stack overflows. The port works on the unsigned magnitude and inverts the result at the end. All other powers give the same result as before.

### 6. Zero-snapping threshold (`rat.cpp`, `_snaprat`)

Upstream snaps a result to 0 if |r| < max(|a|, |b|) · 10^−precision. For large inputs that threshold is large: at precision 32, `lograt(1e40)` has a threshold of 1e8 and the correct result 92.1 is snapped to 0. The port additionally requires |r| < 10^−precision. Tiny residues (√2.25 − 1.5) are still snapped, real results never are.

### 7. `NumberToString` (`conv.cpp`)

- 0 in scientific/engineering format is shown as `0.e+0` instead of `0.e-1`.
- Scientific/engineering format with |x| < 1 rounds at full precision. Upstream shifted the rounding position by the exponent and shortened the output, dropping significant digits.

### 8. Added functions (`conv.cpp`, `ratpak.h`)

`RatToScientificString` and `NumberToScientificString` return scientific notation **without rounding**, for applications that round to their own display width. They are appended at the end of `conv.cpp` (they need the file-local `DIGITS` table) and declared at the end of `ratpak.h`.

## Updating to a newer upstream

1. Copy the new upstream `src/CalcManager/Ratpack/*`, `sal_cross_platform.h` and `winerror_cross_platform.h` over the old files.
2. Re-apply change 1 with the replacement table, then changes 2–8. Search the old version for `ESP32 port:` to find them.
3. Check whether upstream fixed any of changes 4–7 in the meantime, and drop those.
