# qsim tech stack

## Language and build

- Free Pascal 3.2.2, `{$MODE FPC}` with explicit modeswitches.
- Code style: `agent-instructions/pascal_style.md`. In short: `object`/`record`
  with `Init`/`Done`, no `class`, no exceptions, `Boolean` + `out` for errors,
  sized integer types, `UPPER_SNAKE` constants instead of enums.
- GNU make. The root `Makefile` has `build`, `test` and `clean`, and takes
  `TARGET_CPU`, `TARGET_OS` and `EXEEXT` so `fpc-ci` can drive it.
- The build and tests need only `fpc`, `make`, `sh` and `diff`.

## Dependencies

- None. Only `System` and the RTL subset allowed by the style guide: no
  `SysUtils`, `Classes`, `Math`. `Sin`, `Cos`, `Sqrt`, `ArcTan` and `Pi` are in
  `System`.
- One exception: Vulkan rendering lives in `src/qsim_vulkan_visualizer.inc`,
  an include file that calls the Vulkan API. A program that includes it brings
  a Vulkan header (such as `visualdoj/dvulkan`) into scope, so the library
  units do not depend on Vulkan. The Windows demo includes it and also uses
  the `windows` unit. How the demo gets the header is decided in its spec.
- The build has no shader compiler, so the include carries its shaders as
  SPIR-V.

## Simulation model

- Full state vector: 2^n amplitudes of `record Re, Im: Double end`.
- Qubit 0 is the least significant bit of the basis index and the rightmost
  digit in `|...>`, as in Qiskit, so results can be checked against it.
- Memory is 16 × 2^n bytes: 20 qubits take 16 MiB. Demos stay within about
  20 qubits. qsim has no qubit limit of its own: a state the OS cannot give
  memory for is an error, not a crash.
- Measurement uses qsim's own seeded RNG built on integer operations only, so
  one seed gives the same samples on every target.
- SIMD kernels sit behind a per-CPU include, with the plain Pascal kernels as
  the `_no` fallback. The fallback is the reference SIMD results are tested
  against, and the only backend on ppc64 and riscv64. x86_64 kernels must
  work with both the Linux and the win64 calling conventions.
- Visualization units compute what to draw (bars, phases, Bloch vectors,
  circuit layout) apart from how to draw it. `qrun` and `qdb` are CLI tools
  and draw it as text; only the Windows demo includes
  `qsim_vulkan_visualizer.inc`.

## Input and output

- Circuit files: a subset of OpenQASM 3 with the gates of `stdgates.inc`,
  extended phase by phase
  ([roadmap.md](roadmap.md)).
- Input errors print `file:line:col: message` and exit with a nonzero code.
- Text output only, no binary files, so byte order never matters.
- Numbers are printed with a fixed number of digits, and `-0` as `0`, so
  output is identical on all targets.
- Plain ASCII by default; ANSI colors are opt-in.

## Layout of the qsim repository

    Makefile
    src/            library units: qsim.pas, the public API, and qsim_<topic>.pas
    src/qsim_vulkan_visualizer.inc  Vulkan rendering, for programs that include it
    tools/qrun/     qrun CLI
    tools/qdb/      qdb debugger
    demos/<name>/   demos
    tests/<name>/   test programs, circuit files, expected output

## Tests

- Assertion tests: `{$ASSERTIONS ON}` and `Assert`; amplitudes are compared
  with a tolerance of 1e-9.
- Golden-output tests: stdout of a program or a `qrun` run is compared with a
  committed `.can` file by `diff`.
- `qdb` is tested by piping commands to its stdin.

## Targets

| Target | Run by | Checked |
|---|---|---|
| x86_64-linux | native | build + test |
| powerpc64-linux, big-endian | `fpc-ci ppc64`, qemu | build + test |
| riscv64-linux, FPC 3.3.1 | `fpc-ci riscv64`, qemu | build + test |
| x86_64-win64 | `fpc-ci win64`, Wine | build only |

- The Windows demo, and with it `qsim_vulkan_visualizer.inc`, builds only for
  win64.
- qsim is a private repository and GitHub Actions minutes are paid: the local
  checks above are the acceptance bar.
