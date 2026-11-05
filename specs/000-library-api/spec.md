# 000 library API

Roadmap: phase 0. The whole public API of the qsim library, as the interface
of one unit, `src/qsim_api.pas`, with a comment on each declaration. Later
phases implement it piece by piece ([plan.md](plan.md)); this phase adds no
code to qsim.

The spec also drafts two parts that wait for later decisions:
[SPIR-V generation](#draft-spir-v-generation), the producing-code part of the
API, and [the Vulkan renderer](#draft-vulkan-renderer) in
`qsim_vulkan_visualizer.inc`.

## Decisions

- One unit, three objects: `TQsimCircuit` is what to run, `TQsimState` is a
  run of it, `TQsimSpirv` (draft) is GPU code made from it. No free functions.
- Circuits are made only from OpenQASM text. A program that builds a circuit,
  like the Sudoku demo, writes OpenQASM and compiles it; there are no builder
  calls.
- A compiled circuit is a flat list of operations: a one-qubit gate on
  `Target` with a mask of control qubits, a swap, or a measurement. User
  gates, `ctrl @`, `inv @` and gates on whole registers are expanded by the
  compiler. The same list is run by `TQsimState`, stepped over by `qdb`, drawn
  by the circuit diagram and turned into SPIR-V.
- Errors: `Boolean` result plus `out TQsimError`, first error only. Running
  cannot fail: compiling checks the qubit and bit limits.
- Results and internal state are public read-only fields: no getters.
  Probabilities, marginals, Bloch vectors and counts are computed by the
  callers (the visualization units) from `Amplitudes` and the outcomes of
  `RunShots`.
- Debugging is `Step`, `Position` and `Reset`. `qdb`'s `back` is `Reset`
  with the same seed and `Step` up to one operation before, so measurements
  repeat.
- Limits: 24 qubits (256 MiB of state, the top of the benchmark), 64
  classical bits (one `UInt64` per shot).

## Coverage

| Need | Phases | API |
|---|---|---|
| Compile a file or a string | 2, 16 | `CompileFile`, `Compile` |
| Errors as `file:line:col: message` | 2 | `TQsimError` |
| Run, print the state and probabilities | 2–5 | `State.Init`, `while State.Step do ;`, `Amplitudes` |
| Seeded shots and counts | 6, 23 | `Reset`, `RunShots`, `BitRegs` |
| Names of registers in output | 14, 22 | `QubitRegs`, `BitRegs` |
| Circuit diagram | 22 | `Ops`, `QSIM_OP_NAME` |
| `qdb`: step, breakpoints on lines, back | 24–27 | `Step`, `Position`, `Ops[...].Line`, `Reset` |
| Probabilities, phases, Bloch vectors, marginals | 17–21, 28 | `Amplitudes` |
| Shor's classical part | 32 | outcomes of `RunShots` |
| Vulkan demo: load and step | 37 | `CompileFile`, `Step`, `Reset` |
| GPU code | idea | `TQsimSpirv.Emit` (draft) |

## API

The interface section of `src/qsim_api.pas`. Phases may add private fields
and internal units; public declarations change only through this spec.

```pascal
unit qsim_api;
//
//  Public API of qsim: compiles a circuit written in a subset of OpenQASM 3,
//  runs it on a state vector, and reads the results and the internal state.
//
// Author:  Doj
// License: Public domain or MIT

{$MODE FPC}
{$MODESWITCH DUPLICATELOCALS}
{$MODESWITCH OUT}
{$MODESWITCH RESULT}

interface

const
  QSIM_MAX_QUBITS = 24;
    // More qubits is a compile error. The state of n qubits takes 16 * 2^n
    // bytes: 256 MiB at the limit.
  QSIM_MAX_BITS = 64;
    // More classical bits is a compile error, so the bits fit one UInt64.

  // Operation codes, see TQsimOp. The comments give the matrix of each
  // one-qubit gate by rows, with e(x) = exp(x). Gates with controls become one
  // of these with TQsimOp.Controls: cx and ccx are X, cz is Z, cp is P, cswap
  // is SWAP; so does ctrl @. inv @ gives the inverse: s becomes SDG, t TDG,
  // rx(a) RX(-a), U(a, b, c) U(-a, -c, -b).
  QSIM_OP_X       = 0;   // [0 1; 1 0]
  QSIM_OP_Y       = 1;   // [0 -i; i 0]
  QSIM_OP_Z       = 2;   // [1 0; 0 -1]
  QSIM_OP_H       = 3;   // [1 1; 1 -1] / sqrt(2)
  QSIM_OP_S       = 4;   // [1 0; 0 i]
  QSIM_OP_SDG     = 5;   // [1 0; 0 -i]
  QSIM_OP_T       = 6;   // [1 0; 0 e(i*pi/4)]
  QSIM_OP_TDG     = 7;   // [1 0; 0 e(-i*pi/4)]
  QSIM_OP_RX      = 8;   // rx(a): [cos(a/2) -i*sin(a/2); -i*sin(a/2) cos(a/2)]
  QSIM_OP_RY      = 9;   // ry(a): [cos(a/2) -sin(a/2); sin(a/2) cos(a/2)]
  QSIM_OP_RZ      = 10;  // rz(a): [e(-i*a/2) 0; 0 e(i*a/2)]
  QSIM_OP_P       = 11;  // p(a): [1 0; 0 e(i*a)]
  QSIM_OP_U       = 12;  // U(a, b, c): [cos(a/2)        -e(i*c)*sin(a/2);
                         //              e(i*b)*sin(a/2) e(i*(b+c))*cos(a/2)]
  QSIM_OP_SWAP    = 13;  // swaps qubits Target and Target2
  QSIM_OP_MEASURE = 14;  // measures Target into Bit, collapses the state

  QSIM_OP_NAME: array[QSIM_OP_X..QSIM_OP_MEASURE] of String[7] = (
    'x', 'y', 'z', 'h', 's', 'sdg', 't', 'tdg', 'rx', 'ry', 'rz', 'p', 'U',
    'swap', 'measure');
    // Name of each operation code in OpenQASM 3.

type
PQsimComplex = ^TQsimComplex;
TQsimComplex = record
  Re, Im: Double;
end;
  // An amplitude or a matrix entry.

PQsimError = ^TQsimError;
TQsimError = record
  Line, Col: Int32;
    // Where the error is in the source, from 1; Col counts bytes. 0 if the
    // error has no position, e.g. the file cannot be read.
  Msg: AnsiString;
    // What is wrong, without the position: 'unknown gate foo'.
end;
  // An error in the input. Tools print 'file:line:col: msg', or 'file: msg'
  // when Line is 0, and exit with a nonzero code.

PQsimRegister = ^TQsimRegister;
TQsimRegister = record
  Name: AnsiString;
    // Name as declared.
  First: Int32;
    // Index of its first qubit or bit. Registers take consecutive indexes in
    // the order they are declared.
  Size: Int32;
    // Number of its qubits or bits.
end;
  // A declared register, e.g. qubit[2] q or bit[2] c.

PQsimOp = ^TQsimOp;
TQsimOp = record
  Code: Int32;
    // One of QSIM_OP_* const.
  Line, Col: Int32;
    // Position of the statement that made the operation. All operations made
    // by one statement, such as a call of a user gate or a gate on a whole
    // register, share it.
  Target: Int32;
    // Qubit the operation acts on; the first qubit of QSIM_OP_SWAP.
  Target2: Int32;
    // The second qubit of QSIM_OP_SWAP.
  Bit: Int32;
    // Classical bit QSIM_OP_MEASURE writes.
  Controls: UInt32;
    // Control qubits, bit i is qubit i: the operation acts only on the basis
    // states where all of them are 1.
  ParamCount: Int32;
    // Number of used Params: 1 for rx, ry, rz and p, 3 for U, 0 otherwise.
  Params: array[0..2] of Double;
    // Angles in radians, after inv @.
  Matrix: array[0..1, 0..1] of TQsimComplex;
    // Matrix[Row, Col] of the one-qubit gate applied to Target; unused by
    // QSIM_OP_SWAP and QSIM_OP_MEASURE.
end;
  // One operation of a compiled circuit.

PQsimCircuit = ^TQsimCircuit;
TQsimCircuit = object
public
  // Public fields - read only!
  QubitCount: Int32;
    // Qubits of all registers, at most QSIM_MAX_QUBITS.
  BitCount: Int32;
    // Classical bits of all registers, at most QSIM_MAX_BITS.
  OpCount: Int32;
    // Number of Ops.
  Ops: PQsimOp;
    // Operations in the order they run. User gates, ctrl @, inv @ and gates
    // on whole registers are expanded.
  QubitRegCount: Int32;
    // Number of QubitRegs.
  QubitRegs: PQsimRegister;
    // Qubit registers in the order they are declared.
  BitRegCount: Int32;
    // Number of BitRegs.
  BitRegs: PQsimRegister;
    // Bit registers in the order they are declared.

  procedure Init;
    // Makes an empty circuit: no qubits, no bits, no operations.
  procedure Done;
    // Frees the circuit.
  function Compile(const Source: AnsiString; out Error: TQsimError): Boolean;
    // Replaces the circuit with the one written in Source. Stops at the first
    // error and leaves the circuit empty. include "stdgates.inc" is built in,
    // other includes are errors.
  function CompileFile(const FileName: AnsiString;
                       out Error: TQsimError): Boolean;
    // Compile with the contents of FileName.
end;
  // A compiled circuit: what to run.

PQsimState = ^TQsimState;
TQsimState = object
public
  // Public fields - read only!
  Circuit: PQsimCircuit;
    // The circuit being run.
  Amplitudes: PQsimComplex;
    // 2^Circuit^.QubitCount amplitudes. Bit i of the index is qubit i, so
    // qubit 0 is the rightmost digit in |...>.
  Bits: UInt64;
    // Classical bits, bit i is classical bit i.
  Position: Int32;
    // Index in Circuit^.Ops of the next operation; Circuit^.OpCount at the end.

  procedure Init(Circuit: PQsimCircuit; Seed: UInt64);
    // Allocates the state for Circuit and calls Reset(Seed). Circuit must not
    // change until Done.
  procedure Done;
    // Frees the state.
  procedure Reset(Seed: UInt64);
    // Goes back to the start: the state |0...0>, Bits 0, Position 0.
    // Measurements draw from the random sequence of Seed: equal seeds give
    // equal results on every target.
  function Step: Boolean;
    // Applies the operation at Position and moves to the next one. Does
    // nothing and returns False at the end, so 'while State.Step do ;' runs
    // the rest of the circuit.
  procedure RunShots(Shots: Int32; Outcomes: PUInt64);
    // Runs the circuit Shots times from the start, continuing the random
    // sequence, and stores Bits of each shot in Outcomes[0..Shots - 1].
    // Leaves the state of the last shot.
end;
  // A run of a circuit: the state vector, the classical bits, the position.

//
//  Producing code. DRAFT: SPIR-V is the proposed format, to be decided after
//  stage 6.
//

PQsimSpirv = ^TQsimSpirv;
TQsimSpirv = object
public
  // Public fields - read only!
  Words: PUInt32;
    // SPIR-V module in host byte order, as vkCreateShaderModule takes it.
  WordCount: SizeUInt;
    // Number of Words.
  PassCount: Int32;
    // Number of entry points: pass0, pass1, ... run in this order.
  GroupCount: UInt32;
    // Workgroups along X in the dispatch of each pass.

  procedure Init;
    // Makes an empty module: no words, no passes.
  procedure Done;
    // Frees the module.
  function Emit(const Circuit: TQsimCircuit; out Error: TQsimError): Boolean;
    // Replaces the module with compute shaders that apply the operations of
    // Circuit to the amplitudes in a storage buffer. QSIM_OP_MEASURE is an
    // error at its position for now.
end;
  // GPU code of a circuit.
```

### Usage

The library example of phase 16 will look like this:

```pascal
program bell;
//
//  Prints the amplitudes of a Bell state.
//

{$MODE FPC}
{$MODESWITCH OUT}

uses
  qsim_api;

const
  SOURCE =
    'OPENQASM 3.0;'#10 +
    'include "stdgates.inc";'#10 +
    'qubit[2] q;'#10 +
    'h q[0];'#10 +
    'cx q[0], q[1];'#10;

var
  Circuit: TQsimCircuit;
  State: TQsimState;
  Error: TQsimError;
  I: Int32;
begin
  Circuit.Init;
  if not Circuit.Compile(SOURCE, Error) then begin
    WriteLn('bell:', Error.Line, ':', Error.Col, ': ', Error.Msg);
    Halt(1);
  end;
  State.Init(@Circuit, 0);
  while State.Step do ;
  for I := 0 to (1 shl Circuit.QubitCount) - 1 do
    WriteLn(I, ': ', State.Amplitudes[I].Re:0:4, ' ',
            State.Amplitudes[I].Im:0:4);
  State.Done;
  Circuit.Done;
end.
```

Shots, and `qdb`'s `back`:

```pascal
  GetMem(Outcomes, Shots * SizeOf(UInt64));
  State.Reset(Seed);
  State.RunShots(Shots, Outcomes);

  Back := State.Position - 1;
  State.Reset(Seed);
  while State.Position < Back do
    State.Step;
```

## Draft: SPIR-V generation

`TQsimSpirv.Emit` turns a circuit into one SPIR-V module with one compute
entry point, a pass, per operation; the constants of the operation are baked
into its pass. A runner, an include that calls Vulkan like the visualizer and
maybe sharing its device, runs the module. The runner is not drafted here.

The runner only dispatches the passes in order and knows nothing about
operations, so the emitter can later fuse several operations into one pass
without changing the API.

Module:

- SPIR-V 1.0, for Vulkan 1.0. Capabilities `Shader` and `Float64`: the device
  needs `shaderFloat64`, and results match `TQsimState` within 1e-9.
- One storage buffer at set 0, binding 0: a runtime array of 2^n amplitudes,
  two float64 each, `ArrayStride` 16. It has the layout of `TQsimComplex`, so
  the runner copies `Amplitudes` as is.
- Every pass has local size L × 1 × 1 with L = min(64, 2^(n-1)), and
  `GroupCount` = 2^(n-1) / L. Each invocation handles one pair of amplitudes,
  so there are no bounds checks. A circuit with no qubits gets no passes.

A gate pass, as GLSL for reading (GLSL allows one entry point per module,
SPIR-V many):

```glsl
// cx q[0], q[1]: QSIM_OP_X, Target 1, Controls 1
void pass1() {
  uint i = gl_GlobalInvocationID.x;
  uint i0 = ((i >> 1) << 2) | (i & 1u);  // 0 inserted at bit Target
  uint i1 = i0 | 2u;                     // 1 inserted at bit Target
  if ((i0 & 1u) != 1u)                   // Controls
    return;
  dvec2 a0 = Amp[i0];
  dvec2 a1 = Amp[i1];
  Amp[i0] = cmul(M00, a0) + cmul(M01, a1);  // M is Op.Matrix, constants
  Amp[i1] = cmul(M10, a0) + cmul(M11, a1);
}
```

A swap pass inserts 0 at bit `Target` the same way; if bit `Target2` of `i0`
is 1 and the controls hold, it swaps `Amp[i0]` with the amplitude at `i0` with
both bits flipped. Each pair is visited once.

Runner: a storage buffer of 16 · 2^n bytes filled with the start state, one
compute pipeline per pass, `PassCount` dispatches of `GroupCount` × 1 × 1 in
order with a shader-write to shader-read barrier between them, then a read
back.

Emitter: words appended to a growing buffer, ids given out in order. The
header, capabilities, types, constants, the buffer and the invocation id come
once, then one `OpFunction` per pass. It knows only the instructions the
passes use.

Not covered yet:

- Measurement: `Emit` fails at the first measure. Options: read the state
  back and sample on the host, or a reduction pass on the GPU.
- GPUs without float64.
- One pipeline per operation: creating them takes longer as circuits grow.
  Fusing operations into passes fixes it when it matters.

Tests: `make test` has no GPU and no SPIR-V tools, so a golden test prints the
words of a small circuit in hex; it catches changes, not mistakes. Validity
is checked by hand on Windows: `spirv-val`, and a runner whose amplitudes
match `TQsimState` within 1e-9.

## Draft: Vulkan renderer

`src/qsim_vulkan_visualizer.inc` draws colored triangles and lines in pixels
into one window. It knows nothing of quantum states: the visualization units
compute what to draw, the Windows demo turns it into vertices. 3D views, like
the Bloch spheres, are projected to pixels on the CPU, so the renderer stays
2D: no matrices, no depth buffer.

```pascal
// qsim_vulkan_visualizer.inc
//
//  Draws colored triangles and lines into a window through Vulkan. A Windows
//  program includes it after its uses clause, which brings the windows unit,
//  a Vulkan header and qsim_api into scope.
//

type
PVulkanVertex = ^TVulkanVertex;
TVulkanVertex = packed record
  X, Y: Single;
    // Pixels from the top left corner of the client area.
  Color: UInt32;
    // $AARRGGBB, blended over what is already drawn by its alpha.
end;
  // A vertex of a triangle or a line.

PVulkanVisualizer = ^TVulkanVisualizer;
TVulkanVisualizer = object
public
  // Public fields - read only!
  Width, Height: UInt32;
    // Size of the client area in pixels, updated by BeginFrame.

  function Init(Window: HWND; out Error: TQsimError): Boolean;
    // Creates a Vulkan instance, a surface of Window, a device and a
    // swapchain. On failure Error.Msg names the failed call and its VkResult,
    // and nothing is left to free.
  procedure Done;
    // Waits for the device and frees everything Init created.
  function BeginFrame(ClearColor: UInt32): Boolean;
    // Starts a frame cleared to ClearColor, $AARRGGBB. Returns False if the
    // window has nothing to draw into, e.g. it is minimized; skip the frame
    // then, without EndFrame.
  procedure Triangles(Vertices: PVulkanVertex; Count: Int32);
    // Draws Count div 3 triangles over what is drawn in this frame.
  procedure Lines(Vertices: PVulkanVertex; Count: Int32);
    // Draws Count div 2 lines, one pixel wide, over what is drawn in this
    // frame.
  procedure EndFrame;
    // Shows the frame. Recreates the swapchain if the window was resized.
end;
  // Vulkan renderer of one window.
```

Inside:

- `Init`: an instance with `VK_KHR_surface` and `VK_KHR_win32_surface`, a
  surface from `vkCreateWin32SurfaceKHR`, the first device with a queue that
  can draw and present, a swapchain with FIFO present mode (always available),
  one render pass that clears, two pipelines, triangle list and line list,
  with the same shaders, alpha blending and dynamic viewport.
- `Triangles` and `Lines` copy vertices into CPU memory; `EndFrame` uploads
  them to a host-visible vertex buffer that grows as needed, records the draws
  in call order and presents. One frame in flight.
- Shaders: the vertex shader maps pixels to clip space with the window size
  as a push constant (Vulkan's y axis points down, as pixels do); the fragment
  shader outputs the color. They are `const` arrays of SPIR-V words, each with
  its GLSL source in a comment above it, compiled once outside the build, as
  the build has no shader compiler.
- Errors after `Init`, other than an out-of-date swapchain, end the program
  with a message: a demo has nothing to recover.

Phases: 35 uses `Init`, `BeginFrame`, `EndFrame`, `Done`; 36 adds
`Triangles` for bars; 39 adds `Lines` for the spheres. 37, 38 and 40 need
nothing new: keys, colors and animation are the demo's.

## Out of scope

- The visualization units: their specs come with stage 2.
- The OpenQASM subset: it grows phase by phase.
- SIMD kernels: internal, they do not change the API.
- The GPU runner, `qsim` options, `qdb` commands.

## Open questions

1. Unit name `qsim_api`: `qsim` itself clashes with the program name of the
   CLI.
2. 24 qubits as the limit.
3. SPIR-V: measurement on the GPU, GPUs without float64.
4. Vulkan renderer: text labels (bar values, gate names), for example a small
   bitmap font drawn as triangles.

## Done when

- The listing compiles with FPC 3.2.2, given empty bodies, natively and for
  ppc64, riscv64 and win64, and the example compiles against it; the renderer
  listing compiles for win64. Checked outside the repositories: phase 0 adds
  no code to qsim.
- The owner accepts the spec, with the open questions answered or deferred.
