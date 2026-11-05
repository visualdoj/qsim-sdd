# qsim mission

qsim is a simulator of quantum computation written in Free Pascal. It builds a
quantum circuit, runs it, and lets you look inside: amplitudes, probabilities,
the state of each qubit, measurement samples.

## Goals

1. **Readable reference.** Small, clear code that is easy to learn from and to
   check results against. Correctness and clarity come before speed.
2. **Pascal library.** FPC programs build and run circuits from code.
3. **`qsim` CLI.** Runs a circuit file (a subset of OpenQASM 2) and prints
   amplitudes, probabilities or measurement samples.
4. **Visualization library.** Interesting and cool views of the state:
   probability bars, phases, Bloch spheres, circuit diagrams, histograms.
   `qsim` and `qdb` show these views as text; only the Windows demo draws them
   with Vulkan.
5. **`qdb` debugger.** A CLI tool: steps through a circuit in the terminal and
   shows the internal state after each gate. It never uses Vulkan.
6. **Demos.** Grover search, Shor factoring, a Sudoku solver, and a Vulkan
   visualization program for Windows.

## Non-goals

- Threads and distributed state. SIMD is planned, GPU is an idea for later
  ([roadmap.md](roadmap.md)).
- Noise models, density-matrix simulation, real quantum hardware.
- OpenQASM 3. OpenQASM 2 support grows only as far as tests and demos need.
- Graphics on platforms other than Windows.

## Done means

Every demo runs under `make test` and its output is checked on all targets in
[tech-stack.md](tech-stack.md). The Vulkan program is only built there and is
run by hand on Windows.
