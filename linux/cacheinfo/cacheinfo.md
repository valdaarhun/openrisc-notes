# Cacheinfo

This page gives an overview of the cache model in the OpenRISC architecture. It also documents the usage of the cacheinfo API in the linux kernel to report information related to the CPU and cache.

## Cache model

A processor implementation might have either of the two cache implementations:
1. Harvard cache model, i.e., a separate cache for data and instructions.
2. Stanford cache model, i.e., a unified cache for data and instructions.

For a program to execute correctly, it is enough to assume that the processor implements the Harvard cache model even if this is not the case, or if the processor implements no cache. In such cases, the processor simply ignores writes to the relevant cache registers.

## Reporting CPU and cache attributes in Linux

The linux kernel enables descriptive information pertaining to the CPU and cache to be exposed in the sysfs and procfs pseudo-filesystems. CPU-related info can be found in `/proc/cpuinfo` while cache attributes are exposed in `/sys/devices/system/cpu/cpu*/cache/index*`. At the time of writing this doc, the kernel doesn't have the facility to report CPU cache attributes for OpenRISC. The remainder of this doc walks through how the cachinfo API can be leveraged to implement this functionality.

## Cacheinfo for OpenRISC

In OpenRISC, some pieces of information related to the CPU and cache are logged in the kernel ring buffer during startup. This is implemented in `arch/openrisc/kernel/setup.c:print_cpuinfo()`. Use `dmesg` to print these messages. Some cache attributes are also exposed in `/proc/cpuinfo` along with other CPU-related information. We'll modify the implementation so that these cache attributes are exposed in sysfs instead of procfs.

The generic cacheinfo API is located at `drivers/base/cacheinfo.c`. We are mainly interested in two functions - `init_cache_level` and `populate_cache_leaves`. These functions are used to extract cache attributes and populate sysfs. The implementation provided by the generic API does nothing by itself and they are defined as weak symbols (`__attribute__((weak))`). This is because extraction of cache attributes is architecture-specific. These functions need to be implemented explicitly for each architecture which will then override the dummy implementation in the generic API. An architecture port that does not implement these functions cannot expose cache attributes.

For OpenRISC, we'll implement these functions in `arch/openrisc/kernel/cacheinfo.c`.

### init_cache_level

This function is used to pull cache attributes. They can be pulled from special-purpose registers (SPR) in the CPU, the device tree, etc. The generic API provides several functions to extract attributes from the device tree.

For OpenRISC, the instruction cache and data cache attributes are stored in SPRs. At the time of writing this, OpenRISC processors only support one level of cache. The dcache and icache are also local to each processor.

To begin with, we check if the cache components exist. This can be done by checking the DCP and ICP bits in the UPR register. The architecture manual does not provide a mechanism to detect if the dcache and icache are unified. In such a case, the attributes will still be exposed as though the caches are not unified.

The dcache and icache attributes can be pulled from the DCCFGR and ICCFGR registers respectively.

```
DCCFGR

Bits      31----15    14      13     12     11     10     9    8    7   6-3  2-0
Purpose   Reserved  CBWBRI  CBFRI  CBLRI  CBPRI  CBIRI  CCRI  CWS  CBS  NCS  NCW
```

```
ICCFGR

Bits      31----13    12     11     10     9       8      7   6-3  2-0
Purpose   Reserved  CBLRI  CBPRI  CBIRI  CCRI  Reserved  CBS  NCS  NCW
```

```
NCW  -> Number of ways
NCS  -> Number of sets
CBS  -> Cache line size
CWS  -> Cache write strategy (write-though or write-back)
```

The total size of the cache can be calculated as

```
cache size = cache line size * number of ways * number of sets
```

Each cache is a cache leaf and are at level 1.

### populate_cache_leaves

This function is responsible for populating the `cacheinfo` structure. This structure is then used by the generic cacheinfo API to populate sysfs.

```c
struct cpu_cacheinfo {
	struct cacheinfo *info_list;
	unsigned int per_cpu_data_slice_size;
	unsigned int num_levels;
	unsigned int num_leaves;
	bool cpu_map_populated;
	bool early_ci_levels;
};
```

`info_list` in the above structure is a pointer to an array of `struct cacheinfo`. Each cache leaf will have a `struct cacheinfo` structure associated with it and this structure stores the actual cache attributes. For each cache leaf, the cacheinfo structure is populated.

One cache attribute that we haven't talked about yet is `shared_cpu_map`. This is a bitmask associated with each cache leaf denoting the list of *logical* CPUs that share this cache component. Since the dcache and icache are local to each CPU, and OpenRISC does not support hyperthreading at the time of writing this doc, the mapping in `shared_cpu_map` will be 1-to-1 between the CPU core and the cache leaf.

Once the above functions are implemented, we should be able to see a cache directory under /sys/devices/system/cpu/cpuN.

In order to test this in QEMU, we'll have to make a few changes in QEMU as well. This is because upstream QEMU does not emulate the cache SPRs. Since our changes center around reading cache SPRs, simply adding the relevant SPRs and setting their values accordingly should suffice.

First, we'll enable the dcache and icache components in `target/openrisc/cpu.c`.

```diff
@@ -201,7 +201,8 @@ static void or1200_initfn(Object *obj)
     OpenRISCCPU *cpu = OPENRISC_CPU(obj);
 
     cpu->env.vr = 0x13000008;
-    cpu->env.upr = UPR_UP | UPR_DMP | UPR_IMP | UPR_PICP | UPR_TTP | UPR_PMP;
+    cpu->env.upr = UPR_UP | UPR_DMP | UPR_DCP | UPR_ICP | UPR_IMP |
+                   UPR_PICP | UPR_TTP | UPR_PMP;
     cpu->env.cpucfgr = CPUCFGR_NSGF | CPUCFGR_OB32S | CPUCFGR_OF32S |
                        CPUCFGR_EVBARP;
 
@@ -220,7 +221,8 @@ static void openrisc_any_initfn(Object *obj)
     cpu->env.vr2 = 0;           /* No version specific id */
     cpu->env.avr = 0x01030000;  /* Architecture v1.3 */
 
-    cpu->env.upr = UPR_UP | UPR_DMP | UPR_IMP | UPR_PICP | UPR_TTP | UPR_PMP;
+    cpu->env.upr = UPR_UP | UPR_DMP | UPR_DCP | UPR_ICP | UPR_IMP |
+                   UPR_PICP | UPR_TTP | UPR_PMP;
     cpu->env.cpucfgr = CPUCFGR_NSGF | CPUCFGR_OB32S | CPUCFGR_OF32S |
                        CPUCFGR_AVRP | CPUCFGR_EVBARP | CPUCFGR_OF64A32S;

```

Next, we'll register the DCCFGR and ICCFGR registers and define what the various bits in these SPRs mean.

In `target/openrisc/cpu.h`:

```diff
@@ -113,6 +113,31 @@ enum {
     IMMUCFGR_HTR = (1 << 11),
 };
 
+/* Data cache configure register */
+enum {
+    DCCFGR_NCW = (7 << 0),
+    DCCFGR_NCS = (15 << 3),
+    DCCFGR_CBS = (1 << 7),
+    DCCFGR_CWS = (1 << 8),
+    DCCFGR_CCRI = (1 << 9),
+    DCCFGR_CBIRI = (1 << 10),
+    DCCFGR_CBPRI = (1 << 11),
+    DCCFGR_CBLRI = (1 << 12),
+    DCCFGR_CBFRI = (1 << 13),
+    DCCFGR_CBWBRI = (1 << 14),
+};
+
+/* Instruction cache configure register */
+enum {
+    ICCFGR_NCW = (7 << 0),
+    ICCFGR_NCS = (15 << 3),
+    ICCFGR_CBS = (1 << 7),
+    ICCFGR_CCRI = (1 << 9),
+    ICCFGR_CBIRI = (1 << 10),
+    ICCFGR_CBPRI = (1 << 11),
+    ICCFGR_CBLRI = (1 << 12),
+};
+
 /* Power management register */
 enum {
     PMR_SDF = (15 << 0),
@@ -272,6 +297,8 @@ typedef struct CPUArchState {
     uint32_t avr;             /* Architecture version register */
     uint32_t upr;             /* Unit presence register */
     uint32_t cpucfgr;         /* CPU configure register */
+    uint32_t dccfgr;          /* Data cache configure register */
+    uint32_t iccfgr;          /* Instruction cache configure register */
     uint32_t dmmucfgr;        /* DMMU configure register */
     uint32_t immucfgr;        /* IMMU configure register */
```

We'll also have to register these SPRs in the `VMStateDescription` structure in `target/openrisc/machine.c`:

```diff
@@ -101,6 +101,8 @@ static const VMStateDescription vmstate_env = {
         VMSTATE_UINT32(cpucfgr, CPUOpenRISCState),
         VMSTATE_UINT32(dmmucfgr, CPUOpenRISCState),
         VMSTATE_UINT32(immucfgr, CPUOpenRISCState),
+        VMSTATE_UINT32(dccfgr, CPUOpenRISCState),
+        VMSTATE_UINT32(iccfgr, CPUOpenRISCState),
         VMSTATE_UINT32(evbar, CPUOpenRISCState),
         VMSTATE_UINT32(pmr, CPUOpenRISCState),
         VMSTATE_UINT32(esr, CPUOpenRISCState),
```

Next, we'll add handlers that return the values stored in these SPRs in mfspr's helper routine. The helper routine is in `target/openrisc/sys_helper.c`:

```diff
@@ -251,6 +251,12 @@ target_ulong HELPER(mfspr)(CPUOpenRISCState *env, target_ulong rd,
     case TO_SPR(0, 4): /* IMMUCFGR */
         return env->immucfgr;
 
+    case TO_SPR(0, 5): /* DCCFGR */
+        return env->dccfgr;
+
+    case TO_SPR(0, 6): /* ICCFGR */
+        return env->iccfgr;
+
     case TO_SPR(0, 9): /* VR2 */
         return env->vr2;
```

Our cache components are ready to be emulated. For example, to emulate a 4-way 16KB dcache and direct-mapped 8KB icache, we can add the following in `target/openrisc/cpu.c`:

```diff
@@ -205,6 +205,19 @@ static void or1200_initfn(Object *obj)
                    UPR_PICP | UPR_TTP | UPR_PMP;
     cpu->env.cpucfgr = CPUCFGR_NSGF | CPUCFGR_OB32S | CPUCFGR_OF32S |
                        CPUCFGR_EVBARP;
+    
+    /* 4-way 16KB data cache */
+    cpu->env.dccfgr = 0 | (DCCFGR_NCW & (1 << 1))
+                    | (DCCFGR_NCS & (8 << 3))
+                    | (DCCFGR_CBS & (0 << 7))
+                    | DCCFGR_CBIRI
+                    | DCCFGR_CBFRI;
+
+    /* Direct-mapped 8KB instruction cache */
+    cpu->env.iccfgr = 0 | (ICCFGR_NCW & (0 << 2))
+                    | (ICCFGR_NCS & (9 << 3))
+                    | (ICCFGR_CBS & (0 << 7))
+                    | ICCFGR_CBIRI;
 
     /* 1Way, TLB_SIZE entries.  */
     cpu->env.dmmucfgr = (DMMUCFGR_NTW & (0 << 2))
```
