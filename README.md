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

## Build outputs

After building, you should have:

- `bin/program.elf`
- `bin/program.bin`

> Note: On real hardware (BeagleBone Black), `build_and_run.sh` should **only build** (no QEMU).

## Run on BeagleBone Black (U-Boot + YMODEM)

Connect the serial console (115200 8N1) and stop at the U-Boot prompt.

1) Receive the ELF into RAM using YMODEM:
```text
U-Boot# loady 0x82000000

2) Send `bin/program.elf` from your terminal program using **YMODEM**.

3) Boot the ELF:
```text
U-Boot# bootelf 0x82000000

You should see startup messages, random numbers, and `Tick` appearing periodically.  
(`loady` = receive a file over serial using YMODEM; `bootelf` = launch an ELF from memory.) 

## Key configuration notes (Timer2)

High-level configuration (as required by the lab):

- Enable Timer2 clock: `CM_PER_TIMER2_CLKCTRL = 0x2`
- Unmask IRQ 68 (Timer2) in `INTC_MIR_CLEAR2`
- Set interrupt priority/type in `INTC_ILR68` (`0x0` = IRQ mode, priority 0)
- Stop timer: `TCLR = 0`
- Clear pending interrupts: `TISR = 0x7`
- Load ~2 seconds at 24 MHz: `0xFE91CA00` into `TLDR` and `TCRR`
- Enable overflow interrupt: `TIER = 0x2`
- Start + auto-reload: `TCLR = 0x3` (bit0=start, bit1=auto-reload)

## Debugging tips:

- Interrupt never fires:
  - Verify `INTC_MIR_CLEAR2` is set correctly
- System hangs after enabling IRQs:
  - Ensure the ISR writes `INTC_CONTROL = 0x1` to acknowledge
- Timer counts down but does not restart:
  - Ensure periodic mode (AR bit set in `TCLR`)
- Too many ticks:
  - Ensure `TISR = 0x2` is written after servicing the IRQ
- Random numbers stop printing:
  - Check register saving/restoring in the ISR (stack/register corruption)