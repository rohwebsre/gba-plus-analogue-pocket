# GBA Plus Open Hardware Specification v0.1

## 1. Overview
This document defines the electrical, logical, and timing interfaces of the GBA Plus dual-mode FPGA core.  
– Legacy Mode: Cycle-accurate Game Boy Advance replication.  
– Plus Mode: Extended 1600×1440 framebuffer, banked VRAM, 6 DMA channels, optional 33 MHz CPU clock.

## 2. Top-Level I/O and Ports
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

## 3. Boot-Time Mode Detection
1. On reset, sample `rom_header` bytes at offset 0x0C0.  
2. If bytes == `"G" "B" "A" "P" "L" "U" "S" 0x00"`, assert `plus_mode`.  
3. Otherwise `plus_mode = 0` (legacy).

```verilog
// header_scan.v (simplified)
module header_scan(...);
  // see detailed FSM and compare logic in spec Appendix A
endmodule
```

4. Clock Generation & Switching
- MMCM/PLL generates 16.78 MHz (CLKOUT0) and 33 MHz (CLKOUT1) from clk_50MHz.  
- BUFGMUX selects lcdclk = plusmode ? CLKOUT1 : CLKOUT0.  

Timing Constraints
- False-paths between 16.78 MHz and 33 MHz domains.  
- Multi-cycle paths for AHB crossing.

5. Video Pipeline

5.1 Legacy Scaler
- Input: 240×160 @ 5:5:5 RGB  
- Upscale to 1440×960 via nearest-neighbor.  
- Center within 1600×1440, black border.

5.2 Plus Passthrough
- Input: 1600×1440 @ 8:8:8 RGB from extended VRAM.  
- Output directly to LCD.

Mux controlled by plus_mode.

6. Memory Map & Banked VRAM
| Region          | Addr Range (Legacy)       | Addr Range (Plus)                  |
|-----------------|---------------------------|------------------------------------|
| VRAM            | 0x06000000–0x06017FFF     | 0x06000000–0x061FFFFF              |
| Palette RAM     | 0x05000000–0x050003FF     | same + extended LUT banks          |
| OAM             | 0x07000000–0x070003FF     | same + banked OAM segments         |
| DMA Channels    | 4 @ 0x040000B0–0x040000BC | 6 (DMA4/5 @ 0xA0000400–0xA000040C) |

Bank-select register: 0xA000_04F0 (3-bit value → map 256 KB banks).

7. Sprite Engine FSM
State diagram (Appendix B) with five states:
1. IDLE  
2. FETCH_ATTR  
3. ISSUE_DMA  
4. WAIT_DMA  
5. DRAW_LINE  

See Verilog pseudocode in Appendix B.

8. Blending Logic
- Input pixels: bgpixel[14:0], sppixel[14:0] (0 = transparent).  
- Blend modes (2-bit):  
  - 00 = opaque  
  - 01 = (BG+SP)>>1  
  - 10 = (BG>>2)+(SP−(SP>>2))  
- Priority compare: spprio < bgprio.

Verilog in Appendix C.

9. Verification & Testbench Interface
- Test vectors for header detection, clock switching, memory map.  
- OAM dumps and expected pixel outputs.  
- Scripts: make testbench, make verify.

10. Appendix
- A. Header-scan FSM state chart  
- B. Sprite FSM pseudocode  
- C. Blending logic equations and timing  

End of Spec v0.1
