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

# Placing a breakpoint

...

Executing the trap instruction causes a trap exception to be raised. The trap exception handler starts executing which calls `do_trap()`. The trap instruction can be executed in user space or kernel space. If executed in user space, it raises a SIGTRAP that needs to be handled by the concerned application.

Currently, if a trap is caught in kernel space, the kernel simply crashes. We'll change this behaviour so the kernel can call the kprobe handler to handle the trap.

# Setting up the env

## Kernel

Before building the kernel, make sure that the `.config` file has the following lines:
```
CONFIG_KPROBES=y
CONFIG_HAVE_KPROBES=y
```

Once the kernel has been built as explained in *(insert link here)*, run the following to build and install the modules. This is required to build out-of-tree kernel modules for the OpenRISC kernel.

```sh
make -j `nproc` ARCH=openrisc CROSS_COMPILE="or1k-buildroot-linux-gnu-" modules
make -j `nproc` ARCH=openrisc CROSS_COMPILE="or1k-buildroot-linux-gnu-" modules_install
```

# Kretprobes - implementation

In addition to the config options for KProbes, the following options are required:
```
CONFIG_KRETPROBES=y
CONFIG_RETHOOK=y
```

We'll make kretprobes use the newer rethook mechanism for inserting function return hooks.

Similar to KProbes, kretprobes has a `struct kretprobe`. This structure has a `struct kprobe` field. When a kretprobe is registered, the kernel places a KProbe at the probed function's entry point. This KProbe only has a pre-handler (see `register_kretprobe` in `linux/kernel/kprobes.c`) which points to a special kretprobe pre-handler in the kernel.

`struct kretprobe` itself also has two fields for user-defined handlers - `entry_handler` which runs on function entry (because it is invoked by the encapsulated KProbe's pre-handler), and `handler` which is similar to KProbe's post-handler. A non-zero value returned by `entry_handler` implies there's something wrong and that the kretprobe mustn't be processed.

A kretprobe flow looks a little like this:

`kretprobe_register()`
\_ `kprobe_on_func_entry()` checks if addr is at function entry
\_ compare addr with blacklisted functions
\_ set KProbe pre-handler to `pre_handler_kretprobe()`
\_ allocate `struct rethook` and set rethook callback function to `kretprobe_rethook_handler()`
\_ with this set up, register encapsulated KProbe at entry of the function.

On entering the function, the KProbe is triggered and `pre_handler_kretprobe()` is run. If the kretprobe has an `entry_handler()` associated with it, it executes the handler. It then hooks the function return to `arch_rethook_trampoline()`. The original return address is saved in `struct rethook_node`.

On function return, control is transferred to the trampoline. This callback in turn invokes `rethook_trampoline_handler()`. This handler restores the original return address and fixes pt_regs if needed. It then calls `kretprobe_rethook_handler()` which calls the user-defined kretprobe `handler`. Once this is done, the 