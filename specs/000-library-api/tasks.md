# 000 library API: tasks

- [x] T1. Write [spec.md](spec.md): the API listing, its coverage of the
  roadmap, the SPIR-V and Vulkan renderer drafts.
- [x] T2. Check the listings in a scratch directory, not in qsim: the unit
  with empty bodies and the two usage examples build with FPC 3.2.2 natively
  and with `fpc-ci` for ppc64, riscv64 and win64; the renderer listing builds
  for win64.
- [x] T3. Roadmap: phase 0 keeps the API and the drafts in its spec.
- [x] T4. Owner answers: the unit is `qsim` and the CLI `qrun`; no qubit limit
  of qsim's own; float32 on the GPU and no measurement there; text in the
  renderer.
- [ ] T5. Owner accepts the spec; then phase 0 is done.
