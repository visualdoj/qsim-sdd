# qsim roadmap

Thin end-to-end slices. From phase 2 on, a circuit file goes through the
library and the `qsim` CLI to printed output, and every later phase adds one
small thing through all layers, with its tests and its input errors.

Each phase becomes one spec in `specs/NNN-name/` and is done when
`make build test` passes on every target in [tech-stack.md](tech-stack.md).

## Stage 0: library API

0. Design the whole library API: the interface of the public unit with a
   comment on each declaration, no implementation. It covers compiling,
   getting errors, running, reading results, producing code (SPIR-V or
   another format, once decided), and debugging (stepping and reading the
   internal state, for `qdb` and the visualizations). Later phases implement
   it piece by piece; producing code waits for the format decision.

## Stage 1: simulator core

1. Skeleton: `make build test` builds `qsim`, which prints usage; one test
   passes.
2. A file with `qubit[1] q;` and `x q[0];` prints the state `|1>`.
3. `h`: prints `|+>`; fixed digits, no `-0`.
4. Several qubits and `cx`: Bell state.
5. Probability of each basis state.
6. Seeded RNG, `bit`, `measure`, shots and counts.
7. `z`, `s`, `sdg`, `t`, `tdg`, `y`.
8. `rx`, `ry`, `rz` with numeric angles.
9. Angle expressions: `pi`, `+ - * /`, parentheses.
10. `U`, the general single-qubit gate.
11. `cz`, `cp`, `swap`.
12. `ccx` (Toffoli) and `cswap` (Fredkin).
13. A gate applied to a whole register: `h q;`.
14. Several qubit and bit registers.
15. User `gate` definitions.
16. Library example: a Pascal program compiles a Bell circuit from a string,
    runs it and reads the result.

## Stage 2: visualizations

17. Probability bars in the terminal.
18. Phase of each amplitude, as an angle and as a glyph.
19. Opt-in ANSI colors for phase.
20. Bloch vector of each qubit, from its reduced state.
21. ASCII picture of each qubit's Bloch vector.
22. ASCII circuit diagram.
23. Histogram of measurement counts.

## Stage 3: qdb

24. `qdb file.qasm` shows the circuit and the state before the first gate;
    `step` applies one gate; commands come from stdin.
25. `continue` and breakpoints on lines.
26. Pick any stage 2 view per command: `show probs`, `show bloch`, ...
27. `back`: one gate back, by rerunning from the start.
28. Focus on a subset of qubits: their marginal probabilities.

## Stage 4: demos

29. Grover search on 2 qubits.
30. Grover search on 3 qubits, with `ctrl @` for the multi-controlled Z.
31. QFT, and inverse QFT with `inv @`, checked on known states.
32. Shor for N = 15, a = 7: quantum period finding, then continued fractions
    find 3 × 5.
33. Sudoku solver: Grover on a 2×2 binary Sudoku; a Pascal program builds the
    oracle from the rules.
34. Sudoku solver: a 4×4 puzzle with a few empty cells, within about 20 qubits.

## Stage 5: Vulkan visualizer for Windows

35. The Windows demo opens a window and includes `qsim_vulkan_visualizer.inc`,
    which clears it to a color through Vulkan.
36. Probability bars of a fixed state.
37. Load a circuit file and step through it with keys.
38. Phase shown as color.
39. Bloch spheres in 3D.
40. Animated transitions between steps.

## Stage 6: SIMD

41. Benchmark program: run time of fixed circuits at 16 to 24 qubits, run
    natively by hand; records a baseline.
42. Gates go through a few kernels (one-qubit matrix, controlled, diagonal)
    behind a per-CPU include; only the plain Pascal `_no` backend so far.
43. SSE2 kernel for one-qubit gates on x86_64, tested against `_no` on random
    circuits.
44. SSE2 kernels for controlled and diagonal gates.
45. AVX2 kernels, chosen at run time by CPUID; SSE2 otherwise.

## Ideas, not decided

- GPU simulation: the producing-code part of the API generates SPIR-V compute
  shaders for gates, which run through Vulkan, possibly sharing the setup with
  `qsim_vulkan_visualizer.inc`. Generating SPIR-V from Pascal keeps the build
  free of shader compilers. Decide after stage 6.
