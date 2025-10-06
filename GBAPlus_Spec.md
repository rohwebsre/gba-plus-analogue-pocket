# GBA Plus Open Hardware Specification v1.0

## 1. Overview
GBA Plus is a dual‑mode FPGA core specification for the Analogue Pocket.  
It balances preservation (cycle‑accurate GBA emulation) with extension (new hardware features).

- **Legacy Mode**: 240×160 framebuffer, 16.78 MHz ARM7TDMI, 96 KB VRAM, 4 DMA channels, 40 sprites/line.  
- **Plus Mode**: 1600×1440 framebuffer, 33 MHz CPU option, 2 MB banked VRAM, 6 DMA channels, 64 sprites/line, extended blending.

```
                +-------------------+
                |   ROM Header Scan |
                +---------+---------+
                          |
                          v
                +---------+---------+
                |   Mode Select     |
                | (Legacy / Plus)   |
                +----+---------+----+
                     |         |
     Legacy Mode ----+         +---- Plus Mode
                     |         |
                     v         v
        +------------+--+   +--+------------+
        | Legacy Clock |   | Plus Clock     |
        | 16.78 MHz    |   | 33 MHz         |
        +------+-------+   +-------+--------+
               |                   |
               v                   v
        +------+-------+   +-------+-------+
        | Legacy Video |   | Plus Video    |
        | Scaler 240x160|  | 1600x1440 FB  |
        +------+-------+   +-------+-------+
               |                   |
               +---------+---------+
                         v
                  +------+------+
                  |   LCD Out   |
                  +-------------+
```

This document defines the interfaces, memory maps, state machines, and test methodology needed for implementation.

## 2. Mode Detection
- On reset, the ROM header at offset 0x0C0 is scanned.  
- If bytes == "GBAPLUS\0", assert plus_mode = 1.  
- Otherwise, default to Legacy Mode.

Signal:  
plus_mode (1 bit, global) → configures clock mux, video pipeline, memory map.

## 3. Clocking
- Legacy Mode: 16.78 MHz (matches original GBA).  
- Plus Mode: 33 MHz (doubled throughput).  
- Generated via PLL/MMCM from 50 MHz reference.  
- BUFGMUX selects active clock.  
- Timing constraints: false paths across domains, multi‑cycle paths for AHB crossings.

## 4. Video Pipeline
Two parallel pipelines, selected by plus_mode:

- Legacy Scaler  
  - Input: 240×160 framebuffer (5:5:5 RGB).  
  - Upscale: nearest‑neighbor → 1440×960.  
  - Output: centered in 1600×1440 with black borders.

- Plus Passthrough  
  - Input: 1600×1440 framebuffer (8:8:8 RGB).  
  - Output: direct to LCD controller.

Pixel clock: 33 MHz in both modes for consistency.

## 4. Clock Generation & Switching
- MMCM/PLL generates 16.78 MHz (CLKOUT0) and 33 MHz (CLKOUT1) from clk_50MHz.  
- BUFGMUX selects lcdclk = plusmode ? CLKOUT1 : CLKOUT0.  

Timing Constraints
- False-paths between 16.78 MHz and 33 MHz domains.  
- Multi-cycle paths for AHB crossing.

## 5. Video Pipeline

### 5.1 Legacy Scaler
- Input: 240×160 @ 5:5:5 RGB  
- Upscale to 1440×960 via nearest-neighbor.  
- Center within 1600×1440, black border.

### 5.2 Plus Passthrough
- Input: 1600×1440 @ 8:8:8 RGB from extended VRAM.  
- Output directly to LCD.

Mux controlled by plus_mode.

## 6. Memory Map & Banked VRAM
| Region          | Addr Range (Legacy)          | Addr Range (Plus)                     |
|-----------------|------------------------------|---------------------------------------|
| VRAM            | 0x06000000–0x06017FFF (96KB) | 0x06000000–0x061FFFFF (2MB, banked)   |
| Palette RAM     | 0x05000000–0x050003FF (1KB)  | same + extended LUT banks             |
| OAM             | 0x07000000–0x070003FF (1KB)  | same + banked OAM segments            |
| DMA Channels    | 4 ch @ 0x040000B0–0x040000BC | 6 ch (DMA4/5 @ 0xA0000400–0xA000040C) |

Bank-select register: 0xA000_04F0 (3-bit value → map 256 KB banks).

```
+----------------------+-----------------------------+
| Address Range        | Legacy Mode   | Plus Mode   |
+----------------------+---------------+-------------+
| 0x0600_0000          | 96 KB VRAM    | 2 MB VRAM   |
| 0x0500_0000          | 1 KB Palette  | Extended    |
| 0x0700_0000          | 1 KB OAM      | Banked OAM  |
| 0x0400_00B0–00BC     | 4 DMA         | 4 DMA       |
| 0xA000_0400–040C     | N/A           | DMA4/5      |
| 0xA000_04F0          | N/A           | Bank Select |
+----------------------+---------------+-------------+
```

## 7. Sprite Engine FSM
State diagram (Appendix B) with five states:
1. IDLE  
2. FETCH_ATTR  
3. ISSUE_DMA  
4. WAIT_DMA  
5. DRAW_LINE  

```
   +-------+
   | IDLE  |
   +---+---+
       |
       v
+------+------+
| FETCH_ATTR |
+------+------+
       |
       v
+------+------+
| ISSUE_DMA  |
+------+------+
       |
       v
+------+------+
| WAIT_DMA   |
+------+------+
       |
       v
+------+------+
| DRAW_LINE  |
+------+------+
       |
       v
   +---+---+
   | IDLE  |
   +-------+
```

## 8. Blending Logic
- Input: bgpixel[14:0], sppixel[14:0], spprio, blendmode[1:0].  
- Transparent pixels (0x0000) skipped.  
- Modes:  
  - 00: Opaque (sprite replaces BG).  
  - 01: (BG+SP)>>1 - 50/50 average.
  - 10: (BG>>2)+(SP−(SP>>2)) - 75% sprite + 25% BG.  
- Combinatorial logic with optional pipeline register for 33 MHz timing.
- Priority compare: spprio < bgprio.

```
 BG Pixel ----+
              |        +-------------------+
 SP Pixel ----+------->| Blend Logic Unit  |----> Output Pixel
              |        | Modes:            |
 Priority ----+------->| 00 Opaque         |
 Blend Mode --+------->| 01 50/50 Average  |
                       | 10 75/25 Mix      |
                       +-------------------+
```

## 9. Top-Level I/O and Ports
| Signal         | Dir   | Width     | Description                                |
|--------------- |------ |---------- |------------------------------------------- |
| `clk_50MHz`    | in    | 1         | 50 MHz reference clock                     |
| `reset_n`      | in    | 1         | Active-low global reset                    |
| `rom_header`   | in    | [7:0]×256 | ROM header bytes 0x000–0x0FF               |
| `rom_clk`      | in    | 1         | ROM data strobe                            |
| `rom_data`     | in    | 16        | ROM data bus                               |
| `lcd_hsync`    | out   | 1         | LCD horizontal sync                        |
| `lcd_vsync`    | out   | 1         | LCD vertical sync                          |
| `lcd_rgb`      | out   | 24        | 8:8:8 RGB pixel                            |
| `lcd_clk`      | out   | 1         | Pixel clock to LCD                         |
| **AHB Bus**    |       |           |                                            |
| `hclk`         | in    | 1         | AHB bus clock                              |
| `haddr`        | in    | 32        | AHB address                                |
| `htrans`       | in    | 2         | AHB transfer type                          |
| `hwrite`       | in    | 1         | AHB write enable                           |
| `hwdata`       | in    | 32        | AHB write data                             |
| `hrdata`       | out   | 32        | AHB read data                              |
| `hready`       | out   | 1         | AHB ready                                  |

## 10. Boot-Time Mode Detection
1. On reset, sample `rom_header` bytes at offset 0x0C0.  
2. If bytes == `"G" "B" "A" "P" "L" "U" "S" 0x00"`, assert `plus_mode`.  
3. Otherwise `plus_mode = 0` (legacy).

```verilog
// header_scan.v (simplified)
module header_scan(...);
  // see detailed FSM and compare logic in spec Appendix A
endmodule
```

## 11. Toolchain & SDK
- Header patcher: inserts "GBAPLUS\0" magic string.  
- libgba_plus: exposes APIs for VRAM banks, DMA4/5, Plus PPU registers.  
- Examples:  
  - Legacy demo (letterboxed).  
  - Plus demo (full‑screen tilemap, sprite stress test).

## 12. Verification Plan
- Compatibility: run commercial GBA test ROMs in Legacy Mode.  
- Performance: measure DMA throughput, sprite rendering at 33 MHz.  
- Case Studies:  
  - Full‑screen 1600×1440 demo.  
  - 64‑sprite blending stress test.

## 13. Implementation Roadmap
- [ ] Mode detection FSM.  
- [ ] Clock mux + PLL.  
- [ ] Legacy scaler pipeline.  
- [ ] Plus passthrough pipeline.  
- [ ] VRAM controller + bank select.  
- [ ] DMA4/5 integration.  
- [ ] Sprite FSM.  
- [ ] Blending logic.  
- [ ] Testbenches + verification scripts.  
- [ ] Example demos.

End of Spec v1
