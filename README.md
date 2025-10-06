# GBA Plus Specification

**GBA Plus** is a open specification for a dual-mode FPGA core targeting the Analogue Pocket.

- **Legacy Mode**: Cycle-accurate GameBoy Advance (240x160, 16.78MHz, 4 DMA, 40 sprites).
- **Plus Mode**: Extended execution environment (1600x1440 framebuffer, 33MHz CPU, 6 DMA, 64 sprites, 2MB VRAM).

```
      +-------------------+
      |  ROM Header Scan  |
      +---------+---------+
                |
                v
      +-------------------+
      |   Mode Select     |
      |   (Legacy/Plus)   |
      +----+---------+----+
           |         |
      Legacy Mode  Plus Mode
           |         |
           v         v
 +---------+----+  +-+------------+
 | Legacy Clock |  |  Plus Clock  |
 |   16.78 MHz  |  |    33 MHz    |
 +-------+------+  +-------+------+
         |                 |
         v                 v
 +---------+----+  +-+------------+
 | Legacy Video |  |  Plus Video  |
 |    240x160   |  | 1600x1440 FB |
 +-------+------+  +-------+------+
         |                 |
         +--------+--------+
                  v
            +-----------+
            |  LCD Out  |
            +-----------+
```

This repo contains the specification only - no implementation yet. The goal iss to proovide a clear, open foundation so FPGA developers and retro-computing enthusiasts can build on it.

## Quick Links
- [Full Specification](GBAPlus_Spec.md)
- DOI (Zenodo archive): https://doi.org/10.5281/zenodo.17274535
- Preprint: *arXiv link once uploaded*

## Roadmap

- [] Mode Detection FSM
- [] Clock Generation & BUFGMUX
- [] Legacy Scaler Pipeline
- [] Plus Passthrough Pipeline
- [] Banked VRAM Controller
- [] Sprite Engine FSM
- [] Blending Logic Module
- [] AHB Addr Decoder
- [] Testbench for header-scan, memory map, sprite cases
- [] Example Plus-Mode Demo (tilemap + sprite stress)

## How to contribute
- Open issues for questions, clarifications or suggestions.
- Submit pull requests with corrections, diagrams or HDL stubs.
- Use the "help wanted" label to find tasks.

## License
MIT License - free to use, modify, and share.
