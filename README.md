# Lab003 — Interrupts and Exception Handling (BeagleBone Black)

This project is a bare-metal ARM (Cortex-A8) program for the BeagleBone Black that demonstrates
timer-driven interrupts.

## Objective / Expected Behavior

- The main program prints random numbers continuously.
- Every ~2 seconds, a Timer2 interrupt fires and prints: `Tick`
- The timer runs in periodic (auto-reload) mode.
- The system keeps running without freezing or crashing.

## Project Structure

- `OS/os.c`
  - `main()` (startup messages, timer init, enable IRQs, random loop)
  - `timer_init()` (DMTIMER2 + INTCPS configuration)
  - `timer_irq_handler()` (clears timer interrupt + acknowledges INTCPS + prints `Tick`)
  - UART helper functions for output/debug
- `OS/os.h`
  - Function declarations and register helpers/macros
- `OS/root.s`
  - Exception Vector Table
  - `irq_handler` (saves registers, calls C handler, restores, returns)
  - `enable_irq()` (unmasks IRQs at CPU level)

## Hardware / Memory Map (BeagleBone Black)

- UART0 base: `0x44E09000`
- DMTIMER2 base: `0x48040000`
- INTCPS base: `0x48200000`
- Clock Manager (CM_PER) base: `0x44E00000`
- RAM load address (used for U-Boot loading): `0x82000000`

## How It Works (IRQ Architecture)

**Flow (end-to-end):**

1. **DMTIMER2** counts down.
2. On overflow, DMTIMER2 raises **IRQ 68**.
3. **INTCPS** routes the interrupt to the CPU (it must be unmasked in `INTC_MIR_CLEAR2`).
4. CPU takes an **IRQ exception** and jumps to the **IRQ vector** (vector table entry at offset `0x18`).
5. `OS/root.s: irq_handler` runs:
   - saves registers (`push {r0-r12, lr}`)
   - calls `timer_irq_handler()` in C
   - restores registers
   - returns from IRQ using `subs pc, lr, #4`
6. `OS/os.c: timer_irq_handler()`:
   - clears the timer overflow flag (`TISR = 0x2`)
   - acknowledges the interrupt controller (`INTC_CONTROL = 0x1`)
   - prints `Tick\n` via UART
7. Execution resumes in `main()` as if nothing happened.

## Build (Host PC)

Requirements:
- Arm GNU Toolchain (`arm-none-eabi-gcc`, `arm-none-eabi-as`, `arm-none-eabi-objcopy`)
- Bash (or WSL on Windows)

Build:
```bash
./build_and_run.sh