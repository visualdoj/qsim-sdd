# qsim mission

qsim is a simulator of quantum computation written in Free Pascal. It builds a
quantum circuit, runs it, and lets you look inside: amplitudes, probabilities,
the state of each qubit, measurement samples.

## Goals

1. **Readable reference.** Small, clear code that is easy to learn from and to
   check results against. Correctness and clarity come before speed.
2. **Pascal library.** A clean, minimal API with as few calls as possible
   that still covers every need: compiling, getting errors, running, reading
   results, producing OpenQASM code, and debugging. `qsim`, `qdb`, the
   visualizations and the demos use only this API; `qdb` and the
   visualizations read the internal state through its debugging part.
3. **`qsim` CLI.** Runs a circuit file (a subset of OpenQASM 3) and prints
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
- All of OpenQASM 3: support grows only as far as tests and demos need.
  Timing and pulse features (`delay`, `box`, `defcal`) are out.
- Graphics on platforms other than Windows.

## Done means

Every demo runs under `make test` and its output is checked on all targets in
[tech-stack.md](tech-stack.md). The Vulkan program is only built there and is
run by hand on Windows.
