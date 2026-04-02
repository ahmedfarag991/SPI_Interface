# SPI Slave with Single-Port RAM

**Author:** Ahmed Farag

---

## Project Overview

This project implements a complete **SPI Slave with Single-Port RAM** digital system using Verilog HDL. The goal was to design a hardware module that allows a master device — such as a microcontroller — to communicate serially with an internal memory block using the SPI (Serial Peripheral Interface) protocol. The master can write data to specific memory addresses and read data back, all over just a few wires using the SPI bus.

The design was fully verified through RTL simulation, then taken through the complete FPGA design flow: linting, synthesis, implementation, and bitstream generation — all targeting a **Xilinx Artix-7 FPGA (xc7a35ticpg236-1L)** using **Xilinx Vivado**.

---

## What the Design Does

The system allows a SPI master to perform four types of operations with the internal RAM. Each operation is identified by a 2-bit command embedded in the most significant bits of the 10-bit data frame sent over the MOSI line:

**Write Address** — The master sends the memory address it wants to write to. The SPI slave receives the 10-bit frame, extracts the lower 8 bits, and stores them internally as the write address. The RAM holds this address until the next write data command arrives.

**Write Data** — The master sends the actual data byte to be stored. The SPI slave receives it and the RAM writes that byte into the previously stored write address location. This two-step approach (address first, then data) keeps transactions clean and well-defined.

**Read Address** — The master sends the memory address it wants to read from. Similar to the write address command, the slave extracts and stores this as the internal read address. A flag called `READ_FLAG` is set internally to indicate that an address has been registered and the system is ready for a read data transaction.

**Read Data** — The master triggers a read operation. The RAM fetches the byte stored at the previously registered read address and asserts a `tx_valid` signal to notify the SPI slave that data is ready. The slave then serializes the 8-bit byte and sends it out bit by bit over the MISO line across 8 SPI clock cycles, while the master shifts it in.

---

## System Architecture and Modules

The design is organized as a top-level wrapper called **SPI_wrapper** that connects two sub-modules internally.

**SPI_SLAVE** is the core controller of the system. It implements a finite state machine (FSM) that monitors the SS_n (Slave Select) and MOSI signals to decide what operation the master is requesting. Internally it uses a shift register called `MOSI_BUS` to collect incoming serial bits and assemble them into a 10-bit parallel word. Once 10 bits have been received, it pulses `rx_valid` to inform the RAM that new data is available. For read operations, it uses a separate `MISO_BUS` register and a countdown counter to serialize the data byte back to the master one bit at a time on the MISO line.

**RAM** is a synchronous single-port memory with a depth of 256 locations, each 8 bits wide. It decodes the two command bits from `din[9:8]` every time `rx_valid` is asserted, then acts accordingly — either storing a write address, writing data, storing a read address, or outputting data and asserting `tx_valid`. Both the address and data registers (`addr_WR` and `addr_RD`) are held internally between transactions.

---

## Finite State Machine

The SPI_SLAVE is controlled by a 5-state Moore FSM. The states and their roles are as follows:

**IDLE** is the default state. The module sits here when SS_n is high (no active master). All counters are reset, MISO is cleared, and the system waits.

**CHK_CMD** is entered as soon as the master pulls SS_n low. Here the FSM samples the first MOSI bit to determine what kind of operation is being requested. If MOSI is 0, it goes to WRITE. If MOSI is 1 and READ_FLAG is 0, it goes to READ_ADD. If MOSI is 1 and READ_FLAG is 1, it goes to READ_DATA. If SS_n goes high before a decision is made, it returns to IDLE.

**WRITE** handles both write address and write data operations. It shifts in bits one by one on each clock cycle until 10 bits have been collected, then asserts `rx_valid` and passes the full frame to the RAM.

**READ_ADD** also shifts in 10 bits from the master representing the read address. Once complete, it asserts `rx_valid` and sets `READ_FLAG` to indicate that the read address has been registered.

**READ_DATA** is the most involved state. It first collects the incoming 10-bit frame (which contains dummy bits from the master), then asserts `rx_valid`. Once the RAM responds with `tx_valid` and places the data on `tx_data`, the slave loads it into `MISO_BUS` and starts shifting it out on MISO using a countdown counter (`count2`) over 8 cycles.

---

## Simulation and Verification

The testbench module `tb_SPI` was written to verify all four operations end-to-end. Before the test begins, all 256 RAM locations are initialized to `0x00`, and memory address 15 is pre-loaded with the value `0x14` to serve as a known read target.

The test sequence covers a reset check to verify all signals are correctly initialized, followed by a write address transaction targeting address `0x80`, a write data transaction storing the value `0xFF` at that address, a read address transaction pointing to address `0x0F`, and finally a read data transaction that retrieves `0x14` from address 15 and sends it back serially on MISO. All five test cases passed successfully and were confirmed in the ModelSim waveform viewer.

---

## Linting

Static analysis was performed using **Questa Lint 2021.1**. A total of 10 lint checks were flagged across the design. The notable ones include a warning about a flop output being assigned inside an initial block in the RAM module, multiple warnings about signals being assigned more than once inside a sequential always block in the SPI_SLAVE, and a width mismatch warning where the right-hand side of an assignment is wider than the left-hand side. All warnings were reviewed and found to be non-critical to functional correctness.

---

## Synthesis

Synthesis was run in Xilinx Vivado three times — once for each of three FSM encoding styles: **Gray**, **One-Hot**, and **Sequential**. This was done to compare their impact on area and timing.

In terms of resource usage, the One-Hot encoding used the fewest Slice LUTs (71), while the Sequential encoding used the most (87). Gray and Sequential used identical numbers of Slice Registers (62), while One-Hot used slightly more (66). All three used half a Block RAM tile for the memory.

In terms of timing, the Gray encoding achieved the best Worst Negative Slack (WNS) at 6.099 ns, meaning it had the most timing headroom. All three encodings passed timing with zero failing endpoints.

---

## Implementation

After synthesis, full place-and-route implementation was run for all three encodings. The results confirmed the same trends. The **Gray encoding** remained the fastest design with a post-implementation WNS of 5.382 ns, while the **One-Hot encoding** achieved the smallest area at 70 Slice LUTs. The Sequential encoding fell in between on both metrics but still met all timing constraints.

All three implementations completed successfully with no DRC violations and zero failing timing endpoints. The device floorplan for all encodings showed the logic packed into a very small region of the Artix-7 fabric, reflecting the compact nature of the design.

---

## Bitstream Generation

Bitstream generation was completed successfully in Xilinx Vivado for the target device `xc7a35ticpg236-1L`. The design is ready to be programmed onto a physical Artix-7 FPGA board.

---

## Tools Used

The design was developed and verified using **Verilog HDL** as the hardware description language. Simulation was done in **ModelSim**, linting in **Questa Lint 2021.1**, and the full FPGA flow — synthesis, implementation, and bitstream generation — was carried out in **Xilinx Vivado** targeting the Artix-7 platform.

---

## File Structure

```
SPI_Slave_RAM/
├── rtl/
│   └── SPI_PROJECT.V       # All three modules: RAM, SPI_SLAVE, SPI_wrapper
├── tb/
│   └── tb_SPI.V            # Verilog testbench
├── sim/
│   └── waveforms/          # ModelSim simulation screenshots
├── reports/
│   ├── lint/               # Questa Lint results
│   ├── synthesis/          # Synthesis reports (gray / one_hot / sequential)
│   ├── implementation/     # Implementation reports (gray / one_hot / sequential)
│   └── SPI_report.pdf      # Full project documentation
└── README.md
```

---

*Designed and implemented by Ahmed Farag*
