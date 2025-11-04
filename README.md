# SDU_EmbeddeSystems_Assignment2
<<<<<<< HEAD
Assignment 2 for the course Embedded Systems

image file type uint8
imge size 320x240
=======

0) What you’ll build (big picture)

Laptop (Python) grabs a webcam frame (OpenCV), converts to 8-bit grayscale, and sends it over UART to the Ultra96-V2.

PS (A53 on Ultra96-V2) receives the bytes, writes them into BRAM_IN, triggers your HLS image filter IP in the PL, then reads results back from BRAM_OUT, and sends the processed frame back over UART.

Laptop shows the processed image.

This mirrors the README’s UART → BRAM → PL → BRAM → UART loop, but for a handcrafted image filter instead of a neural net.

1) Prep (software, board, repo)

Install the same tool versions they used (Vivado/Vitis/Vitis HLS 2021.1); their guide is written against that set.

Clone the ultra96-v2 branch (it has board-specific pieces you’ll reuse for PS config & UART).

git clone --branch ultra96-v2 https://github.com/nhma20/FPGA_AI.git


Cable up JTAG, set SW3 boot switches as shown in their picture (JTAG mode).

2) Choose practical defaults (so everything fits)

To keep it simple and BRAM-friendly:

Image format: 8-bit grayscale (1 byte/pixel).

Frame size: 320×240 (QVGA) → 76,800 bytes per frame (fits easily).

UART baud: 115200 (same vibe as their screen /dev/ttyUSB1 115200 check).

We’ll send a tiny header first (width, height), then raw pixels.

3) HLS: make a tiny image-filter IP

We’ll replace their NN HLS with an img_filter HLS block that:

Reads pixels from one BRAM port (IN),

Writes pixels to another BRAM port (OUT),

Uses the standard ap_ctrl_hs handshake (start/done/idle).

3.1 Create the HLS project (Vitis HLS 2021.1)

Their UI steps still apply; point the design to your files and set device to xczu3eg-sbva484-1-i and a 10 ns goal.

Open HLS → Create Project → Add your source/testbench → Part: xczu3eg-sbva484-1-i, Period: 10 ns → Finish.

3.2 Example HLS top (threshold filter)

Use two BRAM interfaces (one for input, one for output) so we can wire them cleanly in Vivado with two AXI BRAM Controllers (simpler than the NN’s glue modules):

// img_filter.hpp
#pragma once
#include <ap_int.h>
#include <hls_stream.h>
#include <stdint.h>

void img_filter(volatile uint8_t *in_bram,
                volatile uint8_t *out_bram,
                int width, int height, uint8_t thresh);

// img_filter.cpp
#include "img_filter.hpp"

void img_filter(volatile uint8_t *in_bram,
                volatile uint8_t *out_bram,
                int width, int height, uint8_t thresh) {
  #pragma HLS INTERFACE bram port=in_bram
  #pragma HLS INTERFACE bram port=out_bram
  #pragma HLS INTERFACE s_axilite port=width  bundle=CTRL
  #pragma HLS INTERFACE s_axilite port=height bundle=CTRL
  #pragma HLS INTERFACE s_axilite port=thresh bundle=CTRL
  #pragma HLS INTERFACE s_axilite port=return bundle=CTRL  // ap_ctrl_hs

  int N = width * height;

  // Simple per-pixel threshold
  PIXELS: for (int i = 0; i < N; i++) {
    #pragma HLS PIPELINE II=1
    uint8_t p = in_bram[i];
    out_bram[i] = (p > thresh) ? 255 : 0;
  }
}


Notes:

bram interfaces create native BRAM ports (address, enable, we, din/dout). They’ll connect to the B ports of two AXI BRAM Controllers in Vivado.

The AXI-Lite CTRL bundle gives you registers for width, height, thresh, and the control (start/done). This mirrors the README’s use of ap_ctrl_hs.

3.3 Test quickly in C/RTL

Add a tiny C testbench that fills an input array, calls img_filter(...), and checks the output.

Run C Simulation and C Synthesis, then Export RTL to produce an IP you can import in Vivado. The README’s flow is the same here.

4) Vivado: simple, robust block design

We’ll follow their overall Vivado steps, but replace the NN IP and custom glue with your img_filter IP + two AXI BRAM subsystems.

4.1 Project + sources

Create a new RTL Project; pick Ultra96-V2 board.

Add your exported HLS IP to Tools → IP → Repository (point to the HLS export dir). Now “img_filter” appears in Add IP.

4.2 Build the diagram

Zynq UltraScale+ MPSoC → Run Block Automation.

Enable UART0 and UART1 in the PS (double-click Zynq, tick UART 0 and UART 1), just like the README. We’ll use UART1 from software.

Add two AXI BRAM Controllers (axi_bram_ctrl_0 and axi_bram_ctrl_1) and let Vivado create BRAMs for them (Run Block Automation when prompted).

Add your img_filter IP.

Connections:

AXI-Lite control of img_filter → connect to PS M_AXI_HPM via an AXI Interconnect (Vivado does this in automation or you can wire it: S_AXI of img_filter → interconnect → PS master).

IN BRAM: connect BRAM Controller 0, Port B to img_filter’s in_bram native BRAM interface (address/data/we/ce).

OUT BRAM: connect BRAM Controller 1, Port B to img_filter’s out_bram native BRAM interface.

Port A of each BRAM Controller stays on the AXI side for the PS; that’s how software will write/read frames.

Clocks/resets: tie img_filter ap_clk/ap_rst_n to PS clock/reset (like the README wires NN IP clocks).

(Optional) Expose a few GPIO LEDs like they do for fun; not needed here. The README shows how they assign pins if you want that.

Validate design, Generate Bitstream, then Export Hardware including bitstream. Same clicks as their steps 9–13.

Why two BRAMs? It simplifies your life: PS writes to BRAM_IN, IP reads there and writes to BRAM_OUT, PS reads back from BRAM_OUT. No custom “fix_address” or controller FSMs needed. (They used helper Verilog modules for the NN path; we avoid that here.)

5) Vitis: platform + app (UART1 + BRAM + IP control)

We’ll follow their Vitis steps (create platform; then an app), and keep the crucial “use UART1 instead of UART0” BSP edits exactly like they describe.

5.1 Create projects

Platform project: point it at the exported XSA with bitstream; Finish.

Application project: “Hello World” template is fine as a starter; we’ll replace helloworld.c.

In BSP settings, switch psu_uart0 → psu_uart1 everywhere they list (standalone BSPs for FSBL, A53, PMU). This makes XUartPs use UART1.

5.2 Software responsibilities

Open UART1 at 115200.

Receive header (width, height, thresh), then receive width*height bytes.

Write input bytes into BRAM_IN’s mapped base address (AXI port A).

Program img_filter registers (width, height, thresh) via AXI-Lite and start (set ap_start).

Poll for done (read ap_done), or wait for ap_idle.

Read back width*height bytes from BRAM_OUT and tx over UART.

The README shows where to change UART to UART1 and how to deploy/flash and open a serial session while flashing (their screen /dev/ttyUSB1 115200 tip is handy).

Tip: In Vitis, include the xparameters.h from the BSP to get the base addresses for the two BRAMs and the AXI-Lite base for your img_filter. You’ll do Xil_Out8/Xil_In8 (or 32-bit bursts) to BRAM, and Xil_Out32/Xil_In32 to the control regs.

6) Host (Laptop): small Python sender/receiver

Adapt the README’s UART test idea, but use your webcam frame instead of a dataset example. They already send an image and get data back over UART—same structure.

Minimal plan (Python 3 + OpenCV + pyserial):

Open serial on /dev/ttyUSB1 (or the one you see with ls /dev/ttyUSB*).

Grab a frame from the webcam, convert to grayscale, resize to 320×240.

Send a small header (e.g., 2 bytes width, 2 bytes height, 1 byte thresh), then raw pixel bytes.

Read back width*height bytes and display.

7) Build, flash, and run (like the README)

Build app in Vitis, Run As → Launch on Hardware. They mention the blue LED during flashing and using screen to watch text—use that to confirm programming is good.

Run your Python script. You should see the processed frame pop up.

8) Troubleshooting & gotchas

Clocking: If HLS reports tight timing, relax the clock (e.g., 10–15 ns) or pipeline less. The README’s HLS advice about UNROLL and timing/latency trade-offs also applies if you later optimize.

Cache: If you enable caches, flush/invalidate around BRAM transfers. If not, you can keep standalone BSP defaults and you’ll be fine for a first demo.

Endian/word alignment: You’re moving bytes. Using Xil_Out8/Xil_In8 is straightforward; or write in aligned 32-bit words if you pack/unpack in software.

UART throughput: 320×240 is only ~77 KB; 115200 is okay for demos. If you bump resolution, increase baud or switch to USB/Ethernet later.

Addresses: Confirm in Vivado’s Address Editor that both BRAMs and img_filter control registers have unique non-overlapping address ranges; note them for Vitis code.

9) Stretch options

Swap the threshold for a 3×3 box blur or Sobel—same BRAM wiring; just change the loop body.

Add an AXI-Lite reg for mode (0=threshold, 1=blur, 2=sobel) and branch inside the kernel.

Try HLS #pragma HLS PIPELINE II=1 (already used) and selective UNROLL for small 3×3 windows. The README’s optimization notes are relevant.

Appendix — Mapping from README steps to your steps

README 2) Create HLS project → You still do this, but your design files are img_filter.cpp/.hpp. Use ap_ctrl_hs, BRAM interfaces, AXI-Lite params. Export RTL to get an IP.

README 3) Create Vivado project → Same flow: Zynq MPSoC + UART0/1 enabled + AXI BRAM. Instead of their NN IP + custom modules (nn_ctrl, fix_address, not_gate), you wire two AXI BRAM controllers to your HLS IP’s two BRAM ports. Generate bitstream & Export Hardware (with bitstream).

README 4) Create Vitis project → Same: make platform from exported hardware, make an app, switch to UART1 in BSP exactly as shown, build, run on hardware. Your main() replaces their helloworld.c to do UART↔BRAM↔IP control.

README 5) UART test → Same spirit: a Python script (like their uart_test_nn.py) that sends bytes over UART and reads the result. Your script uses webcam frames instead of MNIST.
>>>>>>> 2e29e8b4fc0f5fab31b235a6a29342e5cae0a278


--- Instractions for capturing and converting an image

sudo apt install v4l-utils	#install dignostics tool to see camera port and name

v4l2-ctl --list-devices	#run the diagnostics

sudo apt install fswebcam	#install the capturing tool

fswebcam -d /dev/video2 -r 1280x720 --jpeg 95 --no-banner /home/youruser/webcam_shot.jpg	#capture an image

sudo apt install imagemagick	#install conversions tool

convert webcam_shot.jpg -colorspace Gray -resize 320x240 grayscale_small.jpg		#convert image to greyscale and 320x240 resolution

convert grayscale_small.jpg -depth 8 GRAY:output_image.raw	#convert image to raw
