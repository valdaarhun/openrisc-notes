Disclaimer:

If you are unfamiliar with KProbes, please read through references [1], [2] and [3] first for a high-level overview of what they are and how they work.

# KProbes - implementation

## init_kprobes

Some functions are executed when the kernel is initialized. These are marked using the `__init` macro. One such function is `init_kprobes()` which calls an arch-specific function - `arch_init_kprobes()`. `arch_init_kprobes()` has to be implemented for OpenRISC for this to work.

## register_kprobe

KProbes are registered using `register_kprobe()`. This function accepts `struct kprobe`. This struct can either have an address or a symbol name. If provided a symbol name, the kernel takes care of fetching the corresponding address and saving it in the struct.

- `get_kprobe(addr)` checks if a KProbe has already been registered at addr. If that's the  case, nothing else needs to be done.
- But if that's not the case, `prepare_kprobe(kprobe)` is called which in turn calls an arch-specific function `arch_prepare_kprobe()`.

`arch_prepare_kprobe()` performs arch-specific checks and validates if a KProbe can be placed at the given address. The relevant preparations are made for KProbe insertion if the address is valid. For example, KProbes can't be supported for l.lwa and l.swa due to their atomic nature. The insertion of a KProbe will result in a trap execution when the target instruction is reached and, according to the specification, the execution of an lwa-swa instruction block is hampered if an exception occurs and this is undesirable.

Assumptions/Questions:
1. Software breakpoint in openrisc? l.trap K -> what is K? What's the equivalent of x86's 0xcc?
2. Instructions are word-aligned? What about 64-bit instructions?
3. Single stepping instructions?
4. Instruction slot?
5. Delay slot?

# Placing a breakpoint

`do_trap()` is responsible for placing a breakpoint at a particular address. It first checks if the processor is in supervisor mode. If it is *not*, then it sends a SIGTRAP signal with 
