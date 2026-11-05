# 000 library API: plan

Phase 0 adds no code to qsim. The listing in [spec.md](spec.md) is the target;
later phases move it into `src/qsim_api.pas` piece by piece.

## Rules for later phases

- A phase copies the declarations it implements from the listing into
  `src/qsim_api.pas`, with their comments, and writes their bodies.
- A phase adds no public declaration that is not in the listing. If it needs
  a change, its spec says so, and the change goes into spec.md in the same
  spec PR.
- Private fields, internal units and include files are up to each phase.
- The drafts are revised by the phases that implement them: the renderer by
  phase 35, SPIR-V generation after the GPU decision.

## Phase map

| Phase | Declarations it brings |
|---|---|
| 2 | `QSIM_MAX_QUBITS`, `QSIM_OP_X`, `TQsimComplex`, `TQsimError`, `TQsimRegister`; `TQsimOp`: `Code`, `Line`, `Col`, `Target`, `Matrix`; `TQsimCircuit`: all but the bit fields; `TQsimState`: all but `Bits` and `RunShots` |
| 3 | `QSIM_OP_H` |
| 4 | `TQsimOp.Controls`, for `cx` |
| 6 | `QSIM_MAX_BITS`, `QSIM_OP_MEASURE`, `TQsimOp.Bit`, `BitCount`, `BitRegCount`, `BitRegs`, `TQsimState.Bits`, `RunShots`; `Reset` starts the random sequence |
| 7 | `QSIM_OP_Z`, `S`, `SDG`, `T`, `TDG`, `Y` |
| 8 | `QSIM_OP_RX`, `RY`, `RZ`, `TQsimOp.ParamCount`, `Params` |
| 10 | `QSIM_OP_U` |
| 11 | `QSIM_OP_P`, `QSIM_OP_SWAP`, `TQsimOp.Target2` |
| 16 | nothing: the library example shows the API is enough |
| 22 | `QSIM_OP_NAME` |
| 35–40 | `qsim_vulkan_visualizer.inc`, as its draft lists |
| after stage 6 | `TQsimSpirv`, if SPIR-V is chosen |

The other phases change only the compiler, the implementation or the callers
of the API.
