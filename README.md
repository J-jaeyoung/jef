## Table of Contents
* [What is this?](#what-is-this)
* [Setup](#setup)
    * [Install (Ubuntu 22.04 or before)](#install-ubuntu-2204-or-before)
    * [Install (Ubuntu 23.04 or after)](#install-ubuntu-2304-or-after)
    * [Upgrade](#upgrade)
    * [Uninstall](#uninstall)
    * [Dependency](#dependency)
* [Supported environment](#supported-environment)
* [Supported mode](#supported-mode)
* [Added / improved features](#added--improved-features)
    * [Qemu-system cooperation - General](#qemu-system-cooperation---general)
    * [Qemu-system cooperation - Arch specific](#qemu-system-cooperation---arch-specific)
    * [Qemu-system cooperation - Linux specific - Basic](#qemu-system-cooperation---linux-specific---basic)
    * [Qemu-system cooperation - Linux specific - Symbol](#qemu-system-cooperation---linux-specific---symbol)
    * [Qemu-system cooperation - Linux specific - Allocator](#qemu-system-cooperation---linux-specific---allocator)
    * [Qemu-system cooperation - Linux specific - Advanced](#qemu-system-cooperation---linux-specific---advanced)
    * [Qemu-system cooperation - Linux specific - Other](#qemu-system-cooperation---linux-specific---other)
    * [Qemu-user cooperation](#qemu-user-cooperation)
    * [Heap dump features](#heap-dump-features)
    * [Improved features](#improved-features)
    * [Added features](#added-features)
    * [Other](#other)
* [FAQ](#faq)

## What is this?
This is a fork of [GEF](https://github.com/hugsy/gef).
However, there are two major improvements.

1. Added many heuristic commands for kernel debugging __WITHOUT symboled vmlinux__ (for qemu-system; linux kernel 3.x ~ 6.7.x).
2. Added support for [many architectures](https://github.com/bata24/gef/blob/dev/docs/QEMU-USER-SUPPORTED-ARCH.md) (for qemu-user).

Many other commands have been added and improved. Enjoy!

## Setup

### Install (Ubuntu 22.04 or before)
```bash
# Run with root user (sudo is NOT recommended, since considering debian)
wget -q https://raw.githubusercontent.com/bata24/gef/dev/install.sh -O- | sh
```

GEF is installed under `/root` to simplify the installation script.
If you want to change the location, please modify accordingly.

### Install (Ubuntu 23.04 or after)
```bash
# Ubuntu 23.04 restricts global installation with pip3, so you need --break-system-packages option.
wget -q https://raw.githubusercontent.com/bata24/gef/dev/install.sh -O- | sed -e 's/\(pip3 install\)/\1 --break-system-packages/g' | sh
```

### Upgrade
```bash
python3 /root/.gdbinit-gef.py --upgrade
```

### Uninstall
```bash
rm -f /root/.gdbinit-gef.py /root/.gef.rc
sed -i -e '/source \/root\/.gdbinit-gef.py/d' /root/.gdbinit
```

### Dependency
See [install.sh](https://github.com/bata24/gef/blob/dev/install.sh) or
[install-minimal.sh](https://github.com/bata24/gef/blob/dev/install-minimal.sh).

## Supported environment
- Tested on ubuntu 23.10.
- It may work under ubuntu 20.04, 22.04, 23.04, debian 10.x or after.

## Supported mode
* Normal debugging (start under `gdb`)
* Attach to the process
* Attach to the process in another pid namespace (e.g., attaching from outside `docker`)
* Connect to `gdbserver`
* Connect to the gdb stub of `qemu-system` (via localhost:1234 etc.)
* Connect to the gdb stub of `qemu-user` (via localhost:1234 etc.)
* Connect to the gdb stub of `Intel Pin` (via localhost:1234 etc.)
* Connect to the gdb stub of `Intel SDE` (via localhost:1234 etc.)
* Connect to the gdb stub of `qiling framework` (via localhost:1234 etc.)
* Connect to the gdb stub of `KGDB` (over the serial; currently, only gdb 12.x~ is supported)
* Connect to the gdb stub of `VMWare` (via ipaddr:port)
* Record and replay debugging (start under `rr replay`)

See [docs/SUPPORTED-MODE.md](https://github.com/bata24/gef/blob/dev/docs/SUPPORTED-MODE.md) for detail.

## Added / improved features

### Qemu-system cooperation - General
* `qreg`: displays the register values from qemu-monitor (allows to get like `$cs` even under qemu 2.x).
    * It is shortcut for `monitor info registers`.
    * It also prints the details of the each bit of the system register when x64/x86.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/qreg.png)
* `sysreg`: pretty prints system registers.
    * It is the result of `info registers` with filtering general registers.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/sysreg.png)
* `pagewalk`: displays the page table from scanning physical memory.
    * x64 (Supported: 4-Level/5-Level Paging)
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-x64.png)
    * x86 (Supported: PAE/Non-PAE)
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-x86.png)
    * ARM64 (Supported: EL1&0-stage1/EL1&0-stage2/EL2&0-stage1/EL2-stage1/EL3-stage1)
        * ARM v8.7 base. 32bit mode is NOT supported.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-arm64.png)
        * Each level pagewalk sample from HITCON CTF 2018 `super_hexagon`. Parsing of stage2 translation table (`VTTBR_EL2`) is also supported.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-arm64-el123.png)
        * Secure memory scanning is supported, but you have to break in the secure world.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-arm64-secure.png)
        * Pseudo page tables without detailed flags and permission can be output even in the normal world (when uses OP-TEE).
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-arm64-secure-pseudo.png)
    * ARM (only Cortex-A, LPAE/Non-LPAE, PL0/PL1)
        * ARM v7 base. PL2 is NOT supported.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-arm.png)
        * Secure memory scanning is supported, you don't have to break in the secure world (use register with `_S` suffix).
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-arm-secure.png)
* `v2p`/`p2v`: displays transformation virtual address <-> physical address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/v2p-p2v.png)
* `xp`: is a shortcut for physical memory dump.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/xp.png)
* `qemu-device-info`: dumps device information for qemu-escape (WIP).

### Qemu-system cooperation - Arch specific
* `msr`: displays MSR (Model Specific Registers) values by embedding/executing dynamic assembly.
    * Supported on only x64.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/msr.png)
* `uefi-ovmf-info`: dumps addresses of some important structures in each boot phase of UEFI when OVMF is used.
    * Supported on only x64.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/uefi-ovmf-info.png)
* `xsm`: dumps secure memory when gdb is in normal world.
    * Supported on only ARM64 and ARM.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/xsm.png)
* `wsm`: writes the value to secure memory when gdb is in normal world.
    * Supported on only ARM64 and ARM.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/wsm.png)
* `bsm`: sets the breakpoint to secure memory when gdb is in normal world.
    * Supported on only ARM64 and ARM.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/bsm.png)
* `optee-break-ta`: sets the breakpoint to the offset of OPTEE-Trusted-App when gdb is in normal world.
    * Supported on only ARM64 and ARM.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/optee-break-ta.png)
* `pac-keys`: pretty prints ARM64 PAC keys.
    * Supported on only ARM64.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pac-keys.png)

### Qemu-system cooperation - Linux specific - Basic
* `kbase`: displays the kernel base address.
* `kversion`: displays the debugged kernel version.
* `kcmdline`: displays the debugged kernel startup cmdline.
* `kcurrent`: displays current task address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kbase-kversion-kcmdline-kcurrent.png)

### Qemu-system cooperation - Linux specific - Symbol
* `ksymaddr-remote`: displays kallsyms information from scanning kernel memory.
    * Supported kernel versions are not only before v6.1, but also after v6.2 (slightly changed structure in memory).
    * Supported kernel after v6.4 (changed structure in memory again).
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ksymaddr-remote.png)
* `ksymaddr-remote-apply`/`vmlinux-to-elf-apply`: applies kallsyms information obtained by `ksymaddr-remote` or `vmlinux-to-elf` to gdb.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ksymaddr-remote-apply.png)
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vmlinux-to-elf-apply.png)
    * Once you get symboled pseudo ELF file, you can reuse and apply it automatically even after rebooting qemu-system.
    * `vmlinux-to-elf-apply` and `ksymaddr-remote-apply` provide almost the same functionality.
        * `vmlinux-to-elf-apply`: Requires installation of external tools. Create `vmlinux` with symbols.
        * `ksymaddr-remote-apply`: Requires no external tools. Create an blank ELF with only embedded symbols.

### Qemu-system cooperation - Linux specific - Allocator
* `slub-dump`: dumps slub free-list.
    * Supported on x64/x86/ARM64/ARM + SLUB + no-symbol + kASLR.
    * Supported on both `CONFIG_SLAB_FREELIST_HARDENED` is `y` or `n`.
    * It supports to dump partial pages (`-v`) and NUMA node pages (`-vv`).
    * Since `page_to_virt` is difficult to implement, it will heuristically determine the virtual address from the freelist.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/slub-dump.png)
* `slab-dump`: dumps slab free-list.
    * Supported on x64/x86/ARM64/ARM + SLAB + no-symbol + kASLR.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/slab-dump.png)
* `slob-dump`: dumps slob free-list.
    * Supported on x64/x86/ARM64/ARM + SLOB + no-symbol + kASLR.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/slob-dump.png)
* `slub-tiny-dump`: dumps slub-tiny free-list.
    * Supported on x64/x86/ARM64/ARM + SLUB-TINY + no-symbol + kASLR.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/slub-tiny-dump.png)
* `slab-contains`: resolves which `kmem_cache` certain address (object) belongs to (for SLUB/SLUB-TINY/SLAB).
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/slab-contains.png)
    * For `SLUB/SLUB-TINY`, if all chunks belonging to a certain `page` are in use, they will not be displayed by `slub-dump/slub-tiny-dump` command.
    * Even with such an address (object), this command may be able to resolve `kmem_cache`.
* `buddy-dump`: dumps zone of page allocator (buddy allocator) freelist.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/buddy-dump.png)
* `vmalloc-dump`: dumps vmalloc used list and freed list.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vmalloc-dump.png)
* `page`: displays transformation struct page <-> virtual/physical address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/page.png)
    * There are shortcuts: `virt2page`, `page2virt`, `phys2page` and `page2phys`.
* `kmalloc-tracer`: collects and displays information when kmalloc/kfree.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kmalloc-tracer.png)
* `kmalloc-allocated-by`: calls a predefined set of system calls and prints structures allocated by kmalloc or freed by kfree.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kmalloc-allocated-by.png)

### Qemu-system cooperation - Linux specific - Advanced
* `kmagic`: displays useful addresses in kernel.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kmagic.png)
* `kchecksec`: checks kernel security.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kchecksec.png)
* `kconfig`: dumps kernel config if available.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kconfig.png)
* `syscall-table-view`: displays system call table.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/syscall-table-view.png)
    * It also dumps ia32/x32 syscall table under x64.
    * It also dumps compat syscall table under ARM64.
* `ksysctl`: dumps sysctl parameters.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ksysctl.png)
* `ktask`: displays each task address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktask.png)
    * It also displays the memory map of the userland process.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktask-maps.png)
    * It also displays the register values saved on kstack of the userland process.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktask-regs.png)
    * It also displays the file descriptors of the userland process.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktask-fd.png)
    * It also displays the signal handlers of the userland process.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktask-sighands.png)
    * It also displays the namespaces of the userland process.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktask-namespaces.png)
* `kmod`: displays each module address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kmod.png)
    * It also displays each module symbols.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kmod-syms.png)
* `kops`: displays each operations member.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kops.png)
* `kcdev`: displays each character device information.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kcdev.png)
* `kbdev`: displays each block device information.
    * If there are too many block devices, detection will not be successful.
    * This is because block devices are not managed in one place, so I use the list of `bdev_cache` obtained from the slub-dump results.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kbdev.png)
* `kfilesystems`: dumps supported file systems.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kfilesystems.png)
* `kclock-source`: dumps clocksource list.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kclock-source.png)
* `kdmesg`: dumps the ring buffer of dmesg area.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kdmesg.png)
* `kpipe`: displays each pipe information.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kpipe.png)
* `kbpf`: dumps bpf information.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kbpf.png)
* `ktimer`: dumps timer.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ktimer.png)
* `kpcidev`: dumps PCI devices.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kpcidev.png)
* `kipcs`: dumps IPCs information (System V semaphore, message queue and shared memory).
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kipcs.png)
* `kdevio`: dumps I/O-port and I/O-memory informations.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kdevio.png)
* `kdmabuf`: dumps DMA-BUF information.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kdmabuf.png)
* `kirq`: dumps irq information.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/kirq.png)

### Qemu-system cooperation - Linux specific - Other
* `ksearch-code-ptr`: searches the code pointer in kernel data area.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ksearch-code-ptr.png)
* `pagewalk-with-hints`: prints pagetables with description.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/pagewalk-with-hints.png)
* `thunk-tracer`: collects and displays the thunk function addresses that are called automatically (only x64/x86).
    * If this address comes from RW area, this is useful for getting RIP.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/thunk-tracer.png)
* `usermodehelper-tracer`: collects and displays the information that is executed by `call_usermodehelper_setup`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/usermodehelper-tracer.png)

### Qemu-user cooperation
* `si`/`ni`: are the wrapper for native `si`/`ni`.
    * On OpenRISC architecture, branch operations don't work well, so use breakpoints to simulate.
    * On Cris architecture, `stepi`/`nexti` commands don't work well, so use breakpoints to simulate.
    * If you want to use native `si`/`ni`, use the full form `stepi`/`nexti`.
* `c`: is the wrapper for native `c`.
    * When connecting to gdb stub of qemu-user or Intel Pin, gdb does not trap SIGINT during `continue`.
    * If you want to trap, you need to issue SIGINT on the qemu-user or pin side, but switching screens is troublesome.
    * This command realizes a pseudo SIGINT trap by trapping SIGINT on the python side and throwing SIGINT back to qemu-user or Intel Pin.
    * It works only local qemu-user or Intel Pin.
    * If you want to use native `c`, use the full form `continue`.

### Heap dump features
* `partition-alloc-dump`: dumps partition-alloc free-list.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/partition-alloc-dump.png)
    * This command is reserved for the implementation of latest version of chromium.
        * Currently tested: v121.x / 1225457 / dce8d804887f7f941b8bb78ae6ad6419a04038f2
        * https://commondatastorage.googleapis.com/chromium-browser-snapshots/index.html?prefix=Linux_x64/1225457/
    * Supported on only x64 (maybe it works on x86/ARM/ARM64, but not tested).
    * It will try heuristic search if binary has no symbol.
    * How to test:
        * See [dev/partition-alloc-dump/downloader.py](https://github.com/bata24/gef/blob/dev/dev/partition-alloc-dump/downloader.py).
* `tcmalloc-dump`: dumps tcmalloc free-list.
    * Supported on only x64, based on gperftools-2.9.1 (named `libgoogle-perftools{4,-dev}`)
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/tcmalloc-dump.png)
    * How to test:
        * Execute as `LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libtcmalloc.so ./a.out`.
* `musl-heap-dump`: dumps musl-libc heap chunks.
    * Supported on x64/x86, based on musl-libc v1.2.4.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/musl-heap-dump.png)
    * How to test:
        * Get and extract latest source, then `./configure && make install`.
        * Build as `/usr/local/musl/bin/musl-gcc test.c`.
* `uclibc-ng-heap-dump`: dumps uClibc-ng heap chunks.
    * Supported on x64/x86, based on uClibc-ng v1.0.42 malloc-standard.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/uclibc-ng-heap-dump.png)
    * How to test (x64):
        * Download and extract `x86-64--uclibc--stable-2022.08-1.tar.bz2` from https://toolchains.bootlin.com/
        * Add `/PATH/TO/x86_64-buildroot-linux-uclibc/bin` to `$PATH`, then `x86_64-linux-gcc test.c`.
        * Fix interpreter by `patchelf --set-interpreter /PATH/TO/x86_64-buildroot-linux-uclibc/sysroot/lib/ld64-uClibc.so.0 a.out`.
* `uclibc-ng-visual-heap`: is colorized heap viewer for uClibc-ng.
* `optee-bget-dump`: dumps bget allocator of OPTEE-Trusted-App.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/optee-bget-dump.png)

### Improved features
* `vmmap`: is improved.
    * It displays the memory map information even when connecting to gdb stub like qemu-user.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vmmap-qemu-user.png)
    * Intel Pin is supported.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vmmap-pin.png)
    * Intel SDE is supported.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vmmap-sde.png)
    * It is redirected to `pagewalk` when connecting to gdb stub of qemu-system.
    * It supports detection and coloring of `Writable`, `ReadOnly`, `None` and `RWX` regions.
    * It shows the area each register points to.
* Glibc heap commands are improved.
    * It changes the color.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/heap-bins.png)
    * They print bins information if the chunk is in free-list.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/heap-if-in-freelist.png)
    * Thread arena is supported for all `heap` commands.
        * Use `-a` option.
    * It supports new modes `heap arenas` and `heap top`.
    * `find-fake-fast`: searches for a memory with a size-like value that can be linked to the fastbin free-list.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/find-fake-fast.png)
    * `visual-heap`: is colorized heap viewer.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/visual-heap.png)
    * `extract-heap-addr`: analyzes tcache-protected-fd introduced from glibc-2.32.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/extract-heap-addr.png)
* `registers`: is improved.
    * It also shows raw values of flag register, current ring, exception level, secure state, etc.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/registers-x64.png)
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/registers-arm64.png)
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/registers-arm.png)
* `context`: is improved.
    * It supports automatic display of system call arguments when calling a system call.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/context-syscall-args.png)
    * It supports automatic display of address and value when accessing memory.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/context-memory-access.png)
    * It supports smart symbol printing for cpp function.
        * ex: `std::map<int, std::map<int, int>>` will be replaced by `std::map<...>`.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/smart-cpp-function-name.png)
        * command: `gef config context.smart_cpp_function_name true` or `smart-cpp-function-name` (later is used to toggle).
* `telescope`: is improved.
    * It displays ordinal numbers as well as offsets.
    * It displays if there are canary and ret-addr on the target area.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/telescope.png)
    * It supports blacklist address features (to avoid dying when touching the address mapped to the serial device).
    * It also shows the symbol if available.
    * It supports some new options: `--is-addr`, `--is-not-addr`, `--uniq`, `--depth`, `--slab-contains`, `--slab-contains-unaligned`, `--phys` and `--tag`.
* `proc-info`: is improved.
    * It displays some additional informations.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/proc-info.png)
* `elf-info`: is improved.
    * It displays Program Header and Section Header.
    * It supports parsing from memory.
    * It supports parsing remote binary (if download feature is available).
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/elf-info.png)
* `xinfo`: is improved.
    * It shows more information.
    * It also supports kernel debugging.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/xinfo.png)
* `checksec`: is improved.
    * It shows whether Static or Dynamic.
    * It shows whether Stripped or not.
    * It detects canary against static stripped binary.
    * It shows whether Intel CET instructions (`endbr64`/`endbr32`) is found or not.
    * It shows whether Intel CET IBT/SHSTK is enabled or not.
    * It shows whether ARMv8 PAC / MTE is enabled or not.
    * It shows whether RPATH/RUNPATH is set or not.
    * It shows if Clang CFI/SafeStack is used or not.
    * It shows whether System-ASLR is enabled or not.
    * It shows whether GDB ASLR setting is enabled or not.
    * It supports parsing remote binary (if download feature is available).
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/checksec.png)
* `got`: is improved.
    * It displays not only GOT address but also PLT address.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/got.png)
    * It scans `.plt.sec` section if Intel CET is enabled.
    * It can also display the GOT of the library.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/got-libc.png)
    * It can also display `type`, `offset`, `reloc_arg`, `section` and `permission`.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/got-v.png)
* `canary`: is improved.
    * It displays all canary positions in memory.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/canary.png)
* `edit-flags`: is improved.
    * It displays the meaning of each bit if `-v` option is provided.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/edit-flags-x64.png)
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/edit-flags-arm.png)
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/edit-flags-arm64.png)
* `unicorn-emulate`: is improved.
    * It reads and writes correctly to the address pointed to by `$fs`/`$gs`.
    * It supports a new mode to stop after executing N instructions (`-g`).
    * It shows changed memories.
        * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/unicorn-emulate.png)
* `ropper`: is improved.
    * It does not reset autocomplete settings after calling imported ropper.
* `hexdump`: is improved.
    * It supports physical memory if under qemu-system.
    * It will retry with adjusting read size when failed reading memory.
    * By default, the same line is omitted.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/hexdump.png)
* `patch`: is improved.
    * It supports physical memory if under qemu-system.
    * Added some new modes: `pattern`, `hex`, `history`, `revert`, `nop`, `inf`, `trap`, `ret`, and `syscall`.
    * `nop` command has been integrated into `patch` command.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/patch.png)
* `search-pattern`: is improved.
    * It supports when under qemu-system (in short, it works without `/proc/self/maps`)
    * It supports hex string specification, aligned search, search interval and search limit.
    * It also searches UTF-16 string if target string is ASCII.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/search-pattern.png)
* `mprotect`: is improved.
    * It supports more architectures.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/mprotect.png)
* `hijack-fd`: is improved.
    * It supports more architectures.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/hijack-fd.png)
* `format-string-helper` is improved.
    * It supports more printf-like functions.
* `theme` is improved.
    * Supports many colors.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/theme.png)
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/theme-colors-sample.png)
* `up`/`down`: are the wrapper for native `up`/`down`.
    * It shows also backtrace.

### Added features
* `pid`/`tid`: prints pid and tid.
* `filename`: prints filename.
* `auxv`: pretty prints ELF auxiliary vector.
    * Supported also under qemu-user.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/auxv.png)
* `argv`/`envp`: pretty prints argv and envp.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/argv-envp.png)
* `dumpargs`: dumps arguments of current function.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/dumpargs.png)
* `vdso`: disassembles the text area of vdso smartly.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vdso.png)
* `vvar`: dumps the area of vvar.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/vvar.png)
* `gdtinfo`: pretty prints GDT entries. If userland, show sample entries.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/gdtinfo.png)
* `idtinfo`: pretty prints IDT entries. If userland, show sample entries.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/idtinfo.png)
* `tls`: pretty prints TLS area. Some architectures only support glibc.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/tls.png)
* `fsbase`/`gsbase`: pretty prints `$fs_base`, `$gs_base`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/fsbase_gsbase.png)
* `libc`/`ld`/`heapbase`/`codebase`: displays each of the base address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/base.png)
* `break-rva`: sets a breakpoint at relative offset from codebase.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/break-rva.png)
* `command-break`: sets a breakpoint which executes user defined command if hit.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/command-break.png)
* `main-break`: sets a breakpoint at `main` with or without symbols, then continue.
    * This is useful when you just want to run to `main` under using qemu-user or pin, or debugging no-symbol ELF.
* `break-only-if-taken`/`break-only-if-not-taken`: sets a breakpoint which breaks only branch is taken (or not taken).
* `distance`: calculates the offset from its base address.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/distance.png)
* `fpu`/`mmx`/`sse`/`avx`: pretty prints FPU/MMX/SSE/AVX registers.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/fpu-mmx-sse-avx.png)
* `xmmset`: sets the value to xmm/ymm register simply.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/xmmset.png)
* `mmxset`: sets the value to mm register simply.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/mmxset.png)
* `exec-until`: executes until specified operation.
    * Supported following patterns of detection.
        * call
        * jmp
        * syscall
        * ret
        * indirect-branch (only x64/x86)
        * all-branch (call || jmp || ret)
        * memory-access (detect just `[...]`)
        * specified-keyword-regex
        * specified-condition (expressions using register or memory values)
        * user-code
        * libc-code
        * secure-world
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/exec-until.png)
* `xuntil`: executes until specified address.
    * It is slightly easier to use than the original until command.
* `until-next`: executes until next address.
    * This is useful for the operation with `rep` prefix.
* `add-symbol-temporary`: adds symbol information from command-line.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/add-symbol-temporary.png)
* `errno`: displays errno list or specified errno.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/errno.png)
* `u2d`: shows cast/convert u64 <-> double/float.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/u2d.png)
* `unsigned`: shows unsigned value.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/unsigned.png)
* `convert`: shows various conversion.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/convert.png)
* `walk-link-list`: walks the link list.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/walk-link-list.png)
* `hexdump-flexible`: displays the hexdump with user defined format.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/hexdump-flexible.png)
* `hash-memory`: calculates various hashes/CRCs.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/hash-memory.png)
* `memcmp`: compares the contents of the address A and B, whether virtual or physical.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/memcmp.png)
* `memset`: sets the value to the memory range, whether virtual or physical.
* `memcpy`: copies the contents from the address A to B, whether virtual or physical.
* `memswap`: swaps the contents of the address A and B, whether virtual or physical.
* `meminsert`: inserts the contents of the address A to B, whether virtual or physical.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/meminsert.png)
* `is-mem-zero`: checks the contents of address range is all 0x00 or 0xff or not.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/is-mem-zero.png)
* `ii`: is a shortcut for `x/50i $pc` with opcode bytes.
    * It prints the value if it is memory access operation.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ii.png)
* `version`: shows software versions that gef used.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/version.png)
* `arch-info`: shows architecture information used in gef.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/arch-info.png)
* `context-extra`: manages user specified command to execute when each step.
* `comment`: manages user specified temporary comment.
* `seccomp`: invokes `seccomp-tools`.
* `onegadget`: invokes `one_gadget`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/onegadget.png)
* `rp`: invokes `rp++` with commonly used options.
* `call-syscall`: calls system call with specified values.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/call-syscall.png)
* `mmap`: allocates a new memory by `call-syscall`.
* `killthreads`: kill specific or all pthread.
* `constgrep`: invokes `grep` under `/usr/include`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/constgrep.png)
* `proc-dump`: dumps each file under `/proc/PID/`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/proc-dump.png)
* `time`: measures the time of the GDB command.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/time.png)
* `multi-line`: executes multiple GDB commands in sequence.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/multi-line.png)
* `cpuid`: shows the result of cpuid(eax=0,1,2...).
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/cpuid.png)
* `capability`: shows the capabilities of the debugging process.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/capability.png)
* `dasm`: disassembles the code by capstone.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/dasm.png)
* `asm-list`: lists up instructions. (only x64/x86)
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/asm-list.png)
    * This command uses x86data.js from https://github.com/asmjit/asmdb
* `syscall-search`: searches system call by regex.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/syscall-search.png)
* `dwarf-exception-handler`: dumps the DWARF exception handler informations.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/dwarf-exception-handler.png)
* `magic`: displays useful addresses in glibc etc.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/magic.png)
* `dynamic`: dumps the `_DYNAMIC` area.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/dynamic.png)
* `link-map`: dumps useful members of `link_map` with iterating.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/link-map.png)
* `dtor-dump`: dumps some destructor functions list.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/dtor-dump.png)
* `ptr-mangle`: shows the mangled value will be mangled by `PTR_MANGLE`.
* `ptr-demangle`: shows the demangled value of the value mangled by `PTR_MANGLE`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ptr-mangle-demangle.png)
* `search-mangled-ptr`: searches the mangled value from RW memory.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/search-mangled-ptr.png)
* `strings`: searches ASCII string from specific location.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/strings.png)
* `read-system-register`: reads system register for old qemu (ARM32 only).
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/read-system-register.png)
* `v8`: displays v8 tagged object.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/v8.png)
    * It also loads more commands from latest gdbinit for v8.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/v8-load.png)
* `follow`: changes `follow-fork-mode` setting.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/follow.png)
* `smart-cpp-function-name`: toggles `context.smart_cpp_function_name` setting.
* `ret2dl-hint`: shows the structure used by return-to-dl-resolve as hint.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/ret2dl-hint.png)
* `srop-hint`: shows the code for sigreturn-oriented-programming as hint.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/srop-hint.png)
* `sigreturn`: displays stack values for sigreturn syscall.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/sigreturn.png)
* `smart-memory-dump`: dumps all regions of the memory to each file.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/smart-memory-dump.png)
* `search-cfi-gadgets`: searches CFI-valid (for CET IBT) and controllable generally gadgets from executable area.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/search-cfi-gadgets.png)
* `symbols`: lists up all symbols with coloring.
    * It is shortcut for `maintenance print msymbols`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/symbols.png)
* `saveo`/`diffo`: saves and diffs the command outputs.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/saveo-diffo.png)
* `seq-length`: detects consecutive length of the same sequence.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/seq-length.png)
* `gef arch-list`: displays defined architecture information.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/gef-arch-list.png)
* `gef pyobj-list`: displays defined global python object.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/gef-pyobj-list.png)
* `dt`: is wrapper for `ptype /ox TYPE` and `p ((TYPE*) ADDRESS)[0]`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/dt.png)
* `mte-tags`: displays the MTE tags for the specified address.
    * Supported on only ARM64.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/mte-tags.png)
* `xs`: dumps string like `x/s` command, but with hex-string style.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/xs.png)
* `syscall-sample`: shows the syscall calling sample for specified architecture.
* `iouring-dump`: dumps the area of iouring (only x64).
    * This area is mapped to userland, but cannot be accessed from gdb.
    * Therefore, it executes the assembly code and retrieve the contents.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/iouring-dump.png)

### Other
* The category is introduced in `gef help`.
    * ![](https://raw.githubusercontent.com/bata24/gef/dev/images/gef-help.png)
* Combined into one file (from gef-extra). The followings are moved from gef-extras.
    * `peek-pointers`, `current-stack-frame`, `xref-telescope`, `bytearray`, and `bincompare`.
    * This is because a single file is more attractive than ease of maintenance.
* The system-call table used by `syscall-args` is moved from gef-extras.
    * It was updated up to linux kernel 6.7.4 for each architecture.
* Removed some features I don't use.
    * `$`, `ida-interact`, `gef-remote`, `pie`, `pcustom`, `ksymaddr`, `trace-run`, `bufferize`, `output redirect` and `shellcode`.
* Many bugs fix / formatting / made it easy for me to use.

## FAQ
* See [docs/FAQ.md](https://github.com/bata24/gef/blob/dev/docs/FAQ.md).
