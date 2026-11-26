# Device methods 0x1c8 and 0x218 (investigation notes)

Status: **0x218 (compute) implemented**, 0x1c8 (graphics) not. The title
calls both during start-up. 0x218 now builds a shader container from the
pieces (`victory_shader::console_compute_container`) and makes the pipeline
as `CreateComputePipelineState` does (`pipeline::create_console_compute`;
`VICTORY_CONSOLE_PIPELINES=0` turns it off). 0x1c8 still answers
`E_NOTIMPL`, and the title goes without those pipelines (`device.rs`,
`unknown_pipeline`). These notes record what the descriptions hold, as read
from dumps of a real run (`VICTORY_DUMP_PIPELINES=<folder>` writes them; see
`dump_unknown_pipeline`).

Offline check on the 23 dumped 0x218 programs of one run (containers built
from the dumps, then `victory translate --validate --pipelines` on an
RTX 4070 Ti SUPER): 23 translated to IR, 19 with their resources traced, 18
emitted and validated by `dxil.dll`, 5 host pipelines created. Of the
rest, 13 are 64-lane programs with cross-lane code (`quad_perm`, `row_ror`,
`v_permlane16`) that the host refuses on 32-lane hardware, as for the
title's own wave64 compute containers; 4 stop at "a descriptor changed
after loading", 1 at an unhandled `s_pack_ll_b32_b16`.

These programs build their immediate samplers in the code (`s_movk_i32`,
`s_bfm_b32`, `s_mov_b64` of a literal): the runtime bakes the root
signature's static samplers in. `resource.rs` now folds `s_bfm_b32` of two
constants and `s_mov_b64` of a 32-bit literal (zero-extended, as the
lowering reads it), which took the traced programs from 1 to 19.

## Shape of a call

Both are `(this, description, iid, out)` asking for `ID3D12PipelineState`.
The description is an array of records; the title passes a pointer to one
of them. Every record starts with the root signature object (one of ours:
its vtable is in our image), followed by `(pointer, size)` pairs. The
pieces are **not** a shader container: they are a program already split up
and loaded, with the machine code at a GPU-heap address (`0xd49…`, 256-byte
aligned) and the rest in CPU memory (`0xd04…`).

### 0x218: compute, 0x60-byte records

| word | what |
|------|------|
| 0 | root signature |
| 1, 2 | registers, 0x1c bytes (below) |
| 3, 4 | 0x14 bytes: a 32-bit value, then `3`, then zeros |
| 5, 6 | GFX10 machine code; begins with a prologue that embeds the 16-byte hash of word 7's tag 0x43 as literals; ends with `s_endpgm` then `s_code_end` padding |
| 7, 8 | resource table (below) |
| 9, 10 | two pointers to other records of the same array (links, not needed) |
| 11 | a 64-bit hash |

Registers (seven u32), checked against every dumped call:

| offset | value | meaning |
|--------|-------|---------|
| 0x00 | `0x4005cf00` | constant |
| 0x04 | `0x6c090102` | constant |
| 0x08 | 0 | |
| 0x0c | u16 x, u16 y | `numthreads` x, y |
| 0x10 | u16 z, u16 flags | `numthreads` z; flags 0 or 0x10 (meaning unknown; wave size?) |
| 0x14 | `COMPUTE_PGM_RSRC1` | e.g. `0x402c0343`: FLOAT_MODE 0xC0, DX10_CLAMP, MEM_ORDERED |
| 0x18 | `COMPUTE_PGM_RSRC2` | e.g. `0xb8a`: 5 user SGPRs, TGID X/Y/Z, 2 thread-id VGPRs; agrees with `numthreads` and the code in every call |

### 0x1c8: graphics (vertex + pixel), 13 words then zeros

| word | what |
|------|------|
| 0 | root signature |
| 1, 2 | pipeline registers, 0x184 bytes (render state, `0x2c…` RSRC1-like values) |
| 3, 4 | vertex input layout, e.g. 0xbc bytes with `POSITION`, `COLOR`, `TEXCOORD` names |
| 5, 6 | vertex program code |
| 7, 8 | vertex program resource table |
| 9, 10 | pixel program code |
| 11, 12 | pixel program resource table |

### Resource tables (both methods)

A header (0x10 to 0x18 bytes: counts, a size, a mask), then 0x20-byte entries
`u32 tag, then five u32, then 0xffffffff, 0`:

- tag `0x41`: a used-resource entry in exactly the FSSM layout that
  `native::UsedResource` is read from: `u8 resource type (0 SRV, 1 sampler,
  2 CBV, 3 UAV), u8 root type (0 constants, 3 table, 4 immediate sampler),
  u16 0, u32 table offset (dwords), u32 lower bound (register), u32 stride,
  u8 descriptor dwords, u8 first user-data dword, u8 root slot, u8 0`. It
  agrees with the code (a table read through `s4` has user data 4).
- tag `0x42`: a root signature range: `u8 1 (2: an SM 6.6 heap), u8 D3D12
  range type (0 SRV, 1 UAV, 2 CBV, 3 sampler), u8 dimension, u8 0`, the base
  register, the register space; not needed to translate.
- tag `0x43`: the end, with a 16-byte hash (also embedded in the code's
  prologue).

The FSSM root layout (`native::RootParam`) is not in the pieces; it is made
from the used entries (see `console_compute_container`). Root constants take
the user SGPRs up to the next parameter's or the last one (the title's
containers use 1 to 5).

The `numthreads` flags bit 4 is the wave size: set, 32 lanes. Checked on the
dumped programs that touch `exec`: the three with it set use only 32-bit
`exec` operations, the one without it 64-bit ones.

## Looking for the original containers

The code was searched for in every container of every game file up to
16 GB (`shaders.rpf`, `packs.rpf`, `pack/*.rpf`, the executable), as
decompressed native programs, by 16-byte windows of non-padding code. No
program of 29 sampled calls was found (one hit was a common 16-byte
sequence in an unrelated pixel shader). Not searched: `levels.rpf` (76 GB)
and `pack/story000/story000.rpf` (20 GB). Either the programs live there,
or the title builds them at run time.

The title's call sites (title+0x2477578 for 0x1c8, title+0x24800d5 for
0x218) only store the root signature in word 0 and pass a record they found
in a table of their own: the records come ready-made, so the title's code
says nothing about their fields.

## Next steps

1. Graphics (0x1c8): the register block (0x184 bytes) and the input layout
   still need reading, to give the vertex inputs and outputs and the pixel
   inputs, interpolation and export patches the translator wants
   (`StageIo::Vs`, `StageIo::Ps`), and the signature parts (`ISG1`,
   `OSG1`) a graphics container carries.
2. A run of the title with 0x218 on, to see the compute pipelines bound and
   whether the frame changes.

## First run in the title

Handing out every 0x218 pipeline turned the world black: some 2,000 were
made, 881 of their programs did not translate (554 "a descriptor changed
after loading", 95 "an address with two computed terms", 94 "an address
not from a root parameter", 47 partial image stores, …), and their
dispatches were dropped, 3,100 draws and dispatches a frame against 330
with 0x218 refused. With the call refused, the title takes another way to
the same work; with it answered, it relies on pipelines that do nothing.
So `create_console_compute` now makes the host pipeline at once
(`make::now`) and answers only when the host can run it, refusing the
call as before otherwise.
