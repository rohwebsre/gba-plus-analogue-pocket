# GBA Plus Open Hardware Specification v0.1

## 1. Overview
GBA Plus defines a dual‑mode FPGA core:
- **Legacy Mode**: faithful reproduction of the original GBA.
- **Plus Mode**: extended hardware features for new development.

## 2. Mode Detection
- On reset, scan ROM header at offset 0x0C0.
- If bytes == `"GBAPLUS\0"`, assert `plus_mode = 1`.
- Else, run in Legacy Mode.

## 3. Clocking
- Legacy: 16.78 MHz ARM7TDMI + PPU.
- Plus: 33 MHz option via clock mux.

## 4. Video Pipeline
- **Legacy**: 240×160 → scaled to 1440×960 → centered in 1600×1440.
- **Plus**: native 1600×1440 framebuffer passthrough.

## 5. Memory Map
| Region       | Legacy Mode                  | Plus Mode                        |
|--------------|------------------------------|----------------------------------|
| VRAM         | 96 KB @ 0x0600_0000          | 2 MB banked @ 0x0600_0000        |
| Palette RAM  | 1 KB @ 0x0500_0000           | Extended LUT banks               |
| OAM          | 1 KB @ 0x0700_0000           | Banked OAM segments              |
| DMA          | 4 channels @ 0x0400_00B0     | 6 channels (DMA4/5 at 0xA000_0400)|

## 6. Sprite Engine
- FSM: IDLE → FETCH_ATTR → ISSUE_DMA → WAIT_DMA → DRAW_LINE.
- Legacy: 40 sprites/line. Plus: 64 sprites/line.

## 7. Blending Logic
- Input: BG pixel, SP pixel, priorities.
- Modes:
  - 00: Opaque (SP wins).
  - 01: 50/50 average.
  - 10: 75% SP + 25% BG.
- Transparent pixels (0x0000) skipped.

## 8. Verification Plan
- Run commercial GBA test ROMs in Legacy Mode.
- Benchmarks: DMA throughput, sprite stress test.
- Plus demos: full‑screen tilemap, 64‑sprite blending.

---

*This is a living specification. Contributions welcome.*
