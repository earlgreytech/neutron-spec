# NARM Subset of ARM

Due to various toolchain problems as well as extra complexities that come with x86, an ARM VM is proposed as an alternative. An ARMv6-M VM has already been implemented as the "narm" project, but ARMv6-M is missing several key opcodes necessary for efficient smart contracts. Thus, ideally narm would be expanded into ARMv7 or ARMv8. 

The memory map and interface for narm is similar. Instead of interrupts being used for hypervisor communication, narm uses the `SVC` opcode. The subset would have all privileged opcodes removed, use a similar "split" of readable/writeable memory, and share a similar memory map. 

Further details are TBD