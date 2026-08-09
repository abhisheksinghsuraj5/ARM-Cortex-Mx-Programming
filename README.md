# ARM Cortex-M4 Programming on STM32F407

This repository contains a number of small projects that I used while learning and practicing **ARM Cortex-M4 programming** on the **STM32F407G-DISC1** board.

My main aim with these examples was to understand what is actually happening inside the Cortex-M processor, especially for topics such as interrupts, exceptions, stack pointers, privilege levels and context switching. Because of this, many of the examples use direct register access and some inline assembly instead of relying completely on HAL functions.

I also implemented a small task scheduler to understand how concepts such as **SysTick, PendSV, PSP/MSP and context saving/restoring** are used in an RTOS.

> Some examples intentionally generate processor faults. They are included for learning and debugging purposes.

---

## Hardware and tools used

- **Board:** STM32F407G-DISC1 / STM32F4 Discovery
- **MCU:** STM32F407VGTx
- **Core:** ARM Cortex-M4
- **IDE:** STM32CubeIDE
- **Compiler:** GNU Arm Embedded GCC (`arm-none-eabi-gcc`)
- **Debugger:** On-board ST-LINK

Most of the projects are independent STM32CubeIDE projects and can be imported separately.

---

## Repository structure

```text
CortexMxProgramming/
├── Documents/
│   └── CortexM4-generic-user-guide.pdf
│
├── Source_code/
│   ├── USART3_int_pend/
│   ├── access_levels/
│   ├── bit_banding/
│   ├── cmsis_task_scheduler/
│   ├── exception_entry_exit/
│   ├── fault_gen/
│   ├── inline_1/
│   ├── interrupt_priority/
│   ├── operation_modes/
│   ├── stack/
│   ├── t-bit/
│   └── task_scheduler/
│
└── README.md
```

---

## Projects

### `operation_modes`

This project shows the difference between **Thread mode** and **Handler mode** in Cortex-M.

A software interrupt is generated using the NVIC Software Trigger Interrupt Register (STIR). Normal code runs in Thread mode and the interrupt handler runs in Handler mode.

Main topics:
- Thread mode
- Handler mode
- NVIC
- Software-triggered interrupt

---

### `access_levels`

Here I experimented with **privileged and unprivileged execution**.

The `CONTROL` register is modified to change Thread mode from privileged to unprivileged mode. This helps to understand which processor resources can be accessed depending on the current privilege level.

Main topics:
- Privileged mode
- Unprivileged mode
- `CONTROL` register
- Access restrictions

---

### `stack`

This example is mainly about the two Cortex-M stack pointers:

- **MSP - Main Stack Pointer**
- **PSP - Process Stack Pointer**

The PSP is initialized manually and Thread mode is changed to use PSP. An SVC exception is also generated to see the behavior of the stack during an exception.

This example was useful before moving to the task scheduler implementation.

---

### `exception_entry_exit`

This project is for understanding what happens when the processor enters an exception and then returns from it.

During exception entry, Cortex-M automatically stores part of the processor context on the stack:

```text
R0, R1, R2, R3, R12, LR, PC, xPSR
```

The project also shows the relation between MSP and PSP when normal code and exception handlers are running.

---

### `USART3_int_pend`

This example shows the **pending and enable state of an interrupt**.

The USART3 interrupt is first set to pending using the NVIC and is then enabled. After it becomes enabled, the processor can service the pending interrupt.

Main registers used here are the NVIC pending and interrupt-set-enable registers.

---

### `interrupt_priority`

This project demonstrates **interrupt priority and preemption**.

Two interrupts are configured with different priorities. While one interrupt handler is executing, another higher-priority interrupt is made pending so that the preemption can be observed in the debugger.

One important point is that, for Cortex-M configurable interrupts, a **smaller priority number means a higher priority**.

---

### `fault_gen`

I used this project to understand Cortex-M fault exceptions and fault debugging.

The code intentionally creates faults such as a **UsageFault**. Inside the fault handler, the stacked processor registers are inspected so that the instruction which caused the fault can be identified.

Registers automatically stored by the processor include:

```text
R0
R1
R2
R3
R12
LR
PC
xPSR
```

The project also checks fault status information such as the UsageFault Status Register.

> **Note:** This project intentionally creates processor faults.

---

### `t-bit`

This example is related to the **Thumb-state T-bit**.

Cortex-M executes Thumb/Thumb-2 instructions. In this example, a function address is modified so that bit 0 is cleared. This creates an invalid execution state and causes a fault.

It is a small example, but it helped me understand why the least significant bit of Cortex-M function addresses is important.

> **Note:** This project intentionally causes a fault.

---

### `bit_banding`

This project demonstrates **SRAM bit-banding**.

A bit is first modified using a normal read-modify-write operation and then using the bit-band alias address.

The basic bit-band address calculation used is:

```text
alias_address = alias_base
              + (byte_offset * 32)
              + (bit_number * 4)
```

This was useful for understanding how a single bit can be accessed through the Cortex-M memory map.

---

### `inline_1`

This project contains some basic examples of **GCC inline assembly inside C code**.

I used instructions such as:

- `MOV`
- `MRS`
- `LDR`

The examples show how C variables can be passed to assembly instructions and how special Cortex-M registers such as `CONTROL` can be read.

---

### `task_scheduler`

This is one of the larger examples in the repository. I implemented a small preemptive task scheduler mainly to understand how an RTOS performs context switching on Cortex-M.

The scheduler includes:

- Separate PSP stack for each task
- MSP for scheduler/exception handling
- Task Control Blocks (TCBs)
- SysTick interrupt
- PendSV exception
- Saving and restoring `R4-R11`
- Ready and blocked task states
- Idle task
- Round-robin task selection
- Critical section handling using `PRIMASK`

The application tasks also toggle the LEDs available on the STM32F4 Discovery board at different rates.

This scheduler is only for learning purposes. It is not intended to replace a real RTOS.

---

### `cmsis_task_scheduler`

This is another version of the task scheduler, but here I used **CMSIS-Core** definitions instead of accessing every Cortex-M register through a manually defined address.

For example:

```c
SysTick_Config(...);
__set_MSP(...);
SCB->ICSR;
SCB->SHCSR;
```

It is useful for comparing direct register programming with the CMSIS approach.

---

## A small note about the task scheduler

For the scheduler, each task has its own stack and uses the **PSP**. Exceptions are handled using the **MSP**.

The basic idea of the context switch is:

```text
Running task
    |
    v
SysTick / task delay
    |
    v
PendSV becomes pending
    |
    v
PendSV_Handler
    |
    +--> Save R4-R11
    +--> Save current PSP in TCB
    +--> Select next ready task
    +--> Load next task PSP
    +--> Restore R4-R11
    |
    v
Exception return
    |
    v
Next task continues
```

During exception entry the processor already saves `R0-R3`, `R12`, `LR`, `PC` and `xPSR`. The remaining registers `R4-R11` are saved by software in the PendSV handler.

This was one of the main reasons I created the scheduler example: to understand the connection between **exceptions and RTOS context switching**.

---

## How to run the projects

1. Install **STM32CubeIDE**.
2. Connect the **STM32F407G-DISC1** board using USB/ST-LINK.
3. Clone or download this repository.
4. In STM32CubeIDE go to:

```text
File -> Import -> General -> Existing Projects into Workspace
```

5. Select the project you want from the `Source_code` folder.
6. Build the project.
7. Start a debug session and flash it to the board.

I normally recommend importing one example at a time because every folder is a separate project.

---

## Debugging

For most of these examples, using the debugger is important because many things are easier to understand by directly checking:

- Core registers
- NVIC registers
- Stack memory
- Disassembly
- MSP and PSP values
- Fault status registers

Some examples also use `printf()` with **ITM/SWV** output. For these projects, the STM32CubeIDE SWV ITM Data Console can be used to see the messages.

---

## Suggested order

If you are new to Cortex-M, I would suggest going through the projects approximately in this order:

```text
operation_modes
    ↓
access_levels
    ↓
stack
    ↓
exception_entry_exit
    ↓
USART3_int_pend
    ↓
interrupt_priority
    ↓
fault_gen
    ↓
t-bit
    ↓
bit_banding
    ↓
inline_1
    ↓
task_scheduler
    ↓
cmsis_task_scheduler
```

For me, understanding **MSP/PSP, exception entry, NVIC, SysTick and PendSV** first made the scheduler part much easier to understand.

---

## Reference

I have also included the Cortex-M4 generic user guide in the `Documents` folder:

```text
Documents/CortexM4-generic-user-guide.pdf
```

The ARM Cortex-M4 documentation and STM32F407 reference manual are very useful while going through the examples, especially for NVIC, SCB, SysTick, fault registers and the memory map.

---

## Why I made this repository

I mainly used these projects to improve my understanding of low-level embedded programming instead of only using high-level APIs.

The topics covered here are especially useful for understanding:

- Interrupt handling
- Exception handling
- Embedded debugging
- Cortex-M registers
- RTOS basics
- Context switching
- Fault analysis
- Startup and low-level firmware behavior

I will keep adding or improving examples as I continue learning Cortex-M and embedded systems.
