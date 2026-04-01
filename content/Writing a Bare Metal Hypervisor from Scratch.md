My idea behind this blog is to condense and simplify all of the information that I have come across for the past three months to build a go to guide for someone who wants to write their own hypervisor for the very first time.  

This blog is written wit the ARM architecture in mind, but the ideas and principles are transferable between instruction sets. 

Stripped down to its basic core, a hypervisor is just highly privileged traffic police. Technically, it is a management layer that enables multiple operating systems to run on the same physical hardware by managing resources like memory and processing. 

Before we start writing our own hypervisor, we need to understand the bare minimum operations every hypervisor should be able to perform: 
1. Isolation: every partition (guest os) should be running in isolation from one other
2. Resource Management: the hypervisor should provide appropriate amount of resources to each partition that it needs to run efficiently, without excessive overhead.
3. Hardware Abstraction: the hypervisor needs to abstracts the hardware to provide a virtualised environment where partitions can run regardless of the actual hardware.

That is all that's required for a functioning bare metal hypervisor. 

This can be achieved by following a series of easy steps. 

# 1. Hardware Initialisation
The hypervisor is the first piece of software to run after the firmware on boot of a machine. Firmware is embedded in the hardware and is run automatically on start up, once it is done doing its job of initialising hardware, it also sets an address such as 0x00000000 in the secure address space for the initial instruction fetch. 

We need to create a boot stub, which is basically a small piece of code whose job is to prepare the environment and hand off control to our hypervisor. We can do this by writing some assembly code and a linker script that ensures our assembly code is loaded at the boot address.

```
ENTRY(_boot)
. = HYP_BASE;  # = 0x40000000
```

The stub performs the following tasks:
1. Park all secondary cores in a spin loop for a WFE instruction to make sure all critical data structures like page tables are initialised only once before parallel execution begins. 
2. It configures all of the special registers required to set up processor’s execution environment. We also need to differentiate between secure and non secure states. An important bit to note is the RW bit of the SCR_EL3 register, this ensures that further execution happens in 64bit rather than 32. 
3. It needs to set an entry point when we jump from EL3 to EL2. By default, the firmware code is run at highest exception level but the hypervisor runs at EL2 so we need to make sure our rust code runs when we jump to EL2 by setting ELR_EL3 (Exception Link Register at EL3) to point to the address of our rust code. This process of jumping can be achieved by using `eret` instruction. 

```
1 def bootEL3toEL2():
2   # Park secondary cores
3   if cpuCoreID() != 0:
4     while True: wfe()
5
6   # Configure control registers for EL2
7   SCR_EL3.NS = 1 # Enable non-secure
8   SCR_EL3.RW = 1 # AArch64 execution
9   SPSR_EL3.M = 0b01001 # EL2 mode
10  SPSR_EL3.DAIF = 0xF # Mask exceptions
11  ELR_EL3 = addressOf(_start_rust)
12  Atomic transition to EL2
13  eret() # Drops privilege, jumps to ELR_EL3
```

# 2. Setting up the runtime
Before we can actually execute the hypervisor code written in rust, we need more assembly code that sets up a runtime since the rust standard library is absent in our low level environment. The assembly portion handles stack setup, while the Rust code immediately establishes foundational runtime infrastructure that all subsequent
code depends on.

Traditionally, the loader is responsible for clearing the .bss region of the memory. Block Started by Symbol (.bss) is the region of the executable where all uninitialised static variables are declared. The linker defines a `__bss_start` and a `__bss_end region` and our assembly code is responsible for clearing it to zero by iterating over it. This is done to ensure past variables do not remain in a new run. 

The assembly code also makes sure the stack pointer points to the top of a preallocated stack region defined in the linker script and allocated in the data segment. 

We setup Universal Asynchronous Receiver/Transmitter (UART), which is a serial communication device responsible for visualising whatever is happening with the hypervisor onto the display monitor. We initialize the UART by writing configuration bytes to these memory
addresses, setting baud rate, data width, stop bits, and enable flags. This is done by writing to the specific registers. 

ARM processors maintain an exception vector table at a
fixed location defined by the VBAR_EL2 (Vector Base Address Register at EL2). We set this register to point to our exception table code. There are four exception types (synchronous/instruction-level, IRQ, FIQ, and system error) and they occur in two modes: either from a lower exception level, or from the same exception level.

```
1 def initializeRuntimeEL2():
2 # Clear BSS section (uninitialized data)
3 for addr in range(__bss_start, __bss_end):
4   memory[addr] = 0
5
6 # Initialize stack pointer
7 SP = __stack_top # Grows downward
8
9 # Configure UART for debug logging
10 UART_CR = 0x301 # Enable TX/RX
11 UART_IBRD = 26 # Baud rate
12 UART_FBRD = 3
13 UART_LCR_H = 0x60 # 8-bit, 1 stop bit
14
15 # Install exception vector table
16 VBAR_EL2 = addressOf(__exception_vectors_el2)
17
18 # Continue to hypervisor main
19 callHypervsorMain()
```

# 3. Memory Translation
In a typical OS, every memory access is translated from a virtual address to physical address using a page table that is managed by the OS. In our case, this translation is done to a Intermediary Physical Address (IPA) instead and the hypervisor maintains its own page table to actually translate it to the physical address. This is made possible by stage two translation by the CPU. The flow is as follows: Guest VA → (via guest Stage 1 page tables) → IPA → (via hypervisor Stage 2 page tables) → PA.

There are a few important points to note before that: 
1. The hypervisor memory needs to be unmapped. This ensures that the partitions can not potentially interfere with the hypervisor code. 
2. Guest OS code is identity mapped. This ensures that critical code of the operating system is directly translated to physical address. This is done for performance and simplicity.
3. The UART memory is also unmapped. Similar to the hypervisor code, the UART should be inaccessible for the partitions. 

This is achieved by writing to the HCR_EL2 (Hypervisor Configuration Register at EL2) and set the VM bit to 1. This tells the CPU to enable stage two translation. 


```
1 def setupStage2Translation():
2   # Create and initialize page table structure
3   stage2_table = createPageTable()
4
5 # Hypervisor memory: unmapped (accesses trigger abort)
6 # stage2_table[0x00000:0x10000] = UNMAPPED
7
8 # Guest memory: identity-mapped (IPA == PA)
9 for ipa in range(0x10000, 0x50000):
10   entry = createPageTableEntry(
11   physAddr=ipa, readable=True, writable=True,
12   executable=True)
13   stage2_table[ipa] = entry
14
15 # UART region: unmapped (prevent guest access to I/O)
16 # stage2_table[0x9000000:0x9001000] = UNMAPPED
17
18 # Activate Stage 2 translation
19 VTTBR_EL2 = addressOf(stage2_table)
20 HCR_EL2.VM = 1 # Enable Stage 2
21 isb() # Instruction sync barrier
```
# 4. Interrupt Routing
On ARM systems, the Generic Interrupt Controller (GIC) is a dedicated hardware module responsible for collecting interrupt signals from various sources, prioritizing them, and routing them to CPU cores. In our setup, we want the interrupts to be managed and routed through the hypervisor. To achieve this we need to configure the GIC by writing to its memory-mapped registers. 

By default, in EL2, the IRQ and FIQ exception masks are set, which means that the interrupts would not raise exceptions. However, we do not want this and need to write to the DAIF (Disable Asynchronous Interrupt Flags) register in EL2. 

We use these interrupts to schedule different processes, specifically when an IRQ occurs, our exception vector table points to our rust scheduler code. 

# 5. Scheduling
The generic hardware timer is configured to fire an interrupt periodically (every 1 millisecond), this is done using the **GICD_ISENABLER** register. When this fires, it automatically raises an exception handled by our rust code which calls our scheduler. The handler saves the current guest partition’s trap frame (all
registers, program counter, exception status, and PSTATE flags) into a kernel managed array partition_frames. The trap frame is a 272-byte structure containing all the CPU state needed to resume execution. A round robin pointer also selects the partition for scheduling. Following this the context is restored with the trap frame of the selected partition. 

```
1 partition_frames = [] # trap frames
2 current_partition = 0 # Currently running partition
3
4 def timerInterruptHandler():
5   # Save current partition’s full context
6   trap_frame = readAllRegisters()
7   partition_frames[current_partition] = trap_frame
8
9 # Select next partition (round-robin)
10 current_partition = (current_partition + 1) % numPartitions
11
12 # Restore next partition’s context
13 next_trap_frame = partition_frames[current_partition]
14 writeAllRegisters(next_trap_frame)
15
16 # Return to guest at restored PC
17 eret() # Exception return, resumes guest
```

With this, we have a functioning hypervisor. We need to build our rust code with the linker so that it loads the boot stub at the right address. 

The next part of this series is to make a distributed system of hypervisors to build a fault tolerant system. [[Distributed Hypervisor]]