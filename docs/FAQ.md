# FAQ


# About the environment

## Does GEF work properly on OS other than Ubuntu?
Yes, it probably works fine for regular Linux.

I have used it on debian. Also it seems to be working fine on WSL2 (ubuntu) so far.
Some users are running it on Arch Linux.
However, I have not confirmed that all commands work correctly.

## What Linux kernel versions does GEF support as guests in qemu-system?
I have confirmed that most commands work on versions 3.x ~ 6.7.x.

However, I have not verified every kernel version.
For example, certain symbols in some versions may not be supported by heuristic symbol detection.
Also, the structure may differ depending on the build config and the compiler that built the kernel.
So there may be environments where GEF does not work.
If you have any trouble, please report it on the issue page.

## GDB will not load GEF.
This is probably because gdb does not support cooperation with python3.
Consider building gdb from source with `./configure --enable-targets=all --with-python=/usr/bin/python3 && make && make install`.

## Where is `gef.py`?
GEF (`gef.py`) is placed in `/root/.gdbinit-gef.py` by default. GEF is one file.

## What is `~/.gef.rc`?
This is the GEF config file. Not present by default.

Executing the `gef save` command saves the current settings to disk (`~/.gef.rc`).
The next time GEF is started, it will be automatically loaded and the settings will be reflected.
This includes the current values of items configurable with `gef config` and alias settings for commands.

## What is `/tmp/gef`?
This is the directory where GEF temporarily stores files.

Since it is used for caching, there is no problem in deleting it.
It will be created automatically the next time GEF starts.

## Will this GEF work as a plugin for `hugsy/gef`?
No, it doesn't work. It replaces `hugsy/gef`.

The compatibility with `hugsy/gef` has already been lost.
Think of it as a completely different product.

## Will each GEF command be more accurate if I have `vmlinux` with debug symbols?
No, whether `vmlinux` includes debug information has no effect on GEF behavior.

GEF always uses its own resolved address with `ksymaddr-remote`.
It also performs its own heuristic structure member detection in each command.

## Does GEF support i386 16-bit mode (real mode)?
No, GEF does not support real mode.

Please consider using other scripts, such as [here](https://astralvx.com/debugging-16-bit-in-qemu-with-gdb-on-windows/).

## Does GEF support to debug Android?
I have never tried it, so I don't know.

I think it will work for userland debugging.
However, Android does not use glibc, so the heap structure is different.
Therefore, I think at least `heap` related commands will not work.

Regarding kernel debugging, I haven't been able to confirm how much the structure is different.

## Does GEF support TEE OS other than OP-TEE?
No, GEF does not support it.

If there is publicly available test image, I consider developing to support that OS.

## Is it possible to debug userland with GEF when using qemu-system?
Partially yes.

I think it can be used when you want to track before and after a system call.
However, of course, I do not recommend continually debugging userland with qemu-system.
This is because many commands are restricted for various reasons.
Consider setting up `gdbserver` in the guest and connecting from the outside.

Note: If KPTI is enabled, many kernel-related commands cannot be used.
The reason is that most memory access to kernel space is unavailable if KPTI is enabled.

## How do I break in userland when using qemu-system?
Use a hardware breakpoint.

When you are stopped inside the kernel, is it in the intended process context? If so, just use `break *ADDRESS` as usual.
But if you're stuck in the kernel context of a different user process than you expected, or in a kernel thread like `swapper/0`,
the virtual address of the process you wanted isn't mapped.
For this reason, software breakpoints that embed `0xcc` in virtual memory cannot be used in some situations.
However, hardware breakpoints can be used without any problems.

## Does GEF work with the latest version of gdb?
Yes, it probably works.

The example of build commands are shown below.
```
git clone --depth 1 https://github.com/bminor/binutils-gdb && cd binutils-gdb
./configure --disable-binutils --disable-ld --disable-gold --disable-gas --disable-sim --disable-gprof --disable-gprofng \
--enable-targets=all --with-python=/usr/bin/python3 --with-system-zlib --with-system-readline
make && make install
```


# About commands

## How does GEF implement kernel analysis related commands without symbols?
Internally, it consists of several steps.

1. Enumerate memory map informations from the page table structure.
2. Detect `.rodata` area of kernel from memory map informations.
3. Scan `.rodata` to identify the kernel version.
4. Parse the structure of `kallsyms` in `.rodata` and get all "symbol and address" pairs.
5. If global variable symbols are available at this point, use it. (= `CONFIG_KALLSYMS_ALL=y`).
    * If not, GEF disassembles the function which uses specified global variable.
    * By parsing the result, GEF obtains the address of the required global variable.
    * This is implemented at `KernelAddressHeuristicFinder` class and `KernelAddressHeuristicFinderUtil` class.
6. Detect the offset of the member of the structure, if necessary.
    * To identify it heuristically, GEF use facts such as whether a value in memory is an address or whether a structure in memory has a specific structure.
    * At this time, GEF take into account the presence or absence of members and changes in their order due to differences in kernel versions.
7. Parse and display the value in memory using all the information detected so far.

As you can see, it doesn't work well if structure members are arranged randomly (`CONFIG_RANDSTRUCT=y`).
Also, depending on the assembly output by the compiler, it may not be possible to parse it correctly.

## What command should I start with when debugging the kernel?
Try `pagewalk` , `ks-apply` and `kchecksec`.
After that, try `slub-dump`, `ktask` and `ksysctl` as well.
Other commands are less important, so check them with `gef help` if necessary.

## I prefer the AT&T style.
Please specify each time using the `set disassembly-flavor att` command.
Or, since the `set disassembly-flavor intel` command is executed in the GEF's main function, it may be a good idea to comment it out.
However, since GEF does not take AT&T syntax parsing into consideration, so some commands may do not work fine.
If you find a case where it doesn't work, please report it on the issue page.

## I don't like the color scheme.
Customize it using the `theme` command, then `gef save`. The config is saved to `~/.gef.rc`.
There is another option is disable colors. Try `gef config gef.disable_color True`.

## I don't want to add `-n` to every command to disable pager.
Try `gef config gef.always_no_pager True` then `gef save`.

## Is GEF possible to re-display the results of a command (for using less-pager)?
Basically you can't.

Please save as appropriate with `|$cat > /tmp/foo.txt` while the less-pager is running.
Or try `gef config gef.keep_pager_result True` then `gef save`. From next time onwards, temporary files will no longer be deleted.

Note: The `pagewalk` command and `pagewalk-with-hints` command can be redisplayed by `-c`.

## `ktask` (or other kernel related commands) does not work.
The kernel you are debugging may have been built with `CONFIG_RANDSTRUCT=y`.

In this case, except for a few commands, they will not work correctly.
Currently, at least following commands do not work.
* `ktask`
* `kmod`
* `kbdev`
* `kcdev`
* `kops`
* `kpipe`
* `ksysctl`
* `kmalloc-tracer`
* `kmalloc-allocated-by`
* `kfiles`
* `kregs`
* `ksighands`
* `kpcidev`
* `knamespaces`
* `kipcs`
* `kfilesystems`

## `vmmap` command does not recognize option.
Try `pagewalk` command.

When connected to qemu-system or vmware's gdb stub, the `vmmap` command is just redirected to the `pagewalk` command.
All options are ignored at this time. If you want to use some options, please use the `pagewalk` command instead of `vmmap` command.

## Is GEF possible to pass the result of a command to a shell command?
Yes, you can use built-in `pipe` command.

For example, `pipe elf-info -n |grep .data` or `|pdisas |grep call`.

## `vmlinux-to-elf-apply` command causes an error of creating ELF.
Please update `vmlinux-to-elf` to the latest version.

If the problem persists, try using the `ks-apply` command. The logic is different a little, so it might work.
If it still doesn't work, please report it on the issue page.

## `got` command does not display PLT address well.
This problem is probably caused by an outdated version of `binutils`.

The `got` command uses `objdump` internally to obtain the PLT address.
However, with certain combinations of `binutils` and `glibc` versions, `objdump` does not display the PLT address.

The currently known combinations are as follows.
* `binutils 2.38` (Ubuntu 22.04 default) + `glibc 2.37 or later`

This problem occurs when you try to use newer glibc in an Ubuntu 22.04 environment using `patchelf` etc.
The workaround is to build and install new `binutils` from source code.

## Can I switch to a mode that references physical memory?
Yes. It is possible if you are using qemu-system. You can switch with `pi enable_phys()` and `pi disable_phys()`.

GEF uses this function internally to switch.
If the mode remains switched due to an interruption during command execution, etc., you will need to fix it manually.

## `magic` command has few valid results.
This is because libc symbols are not loaded.

Unlike kernel symbols, userland symbols do not undergo heuristic detection (with some special exceptions).
Therefore, missing symbols may not be detected by the `magic` command.

If you're referring to system-wide glibc, you can resolve it with these steps:
1. Install the symbols with `apt install libc6-dbg`.
2. Add `set debug-file-directory /usr/lib/debug` in `~/.gdbinit`.


# About python

## Can I access each GEF command object from `python-interactive`?
Yes, you can access it using `__LCO__` that means loaded command objects.

For example, `pi __LCO__["vmmap"]`.

## The class name `KernelAddressHeuristicFinder` is too long.
You can access it using `KF`.

For example, `pi KF.get_slab_caches()` instead of `pi KernelAddressHeuristicFinder.get_slab_caches()`.

## Can I revert the output of `python-interactive` back to decimal from hex?
Yes, you can get it back by executing `pi hexoff()`.

## How can I get the instruction object?
You can get instruction object by `pi get_insn(addr=None)`.

There are also similar functions. Here are the list.
* `get_insn(addr=None)`
* `get_insn_next(addr=None)`
* `get_insn_prev(addr=None)`
* `gef_instruction_n(addr, n)`

## Are there any other globally accessible functions that are useful?
* Memory access
    * `write_memory(addr, data)`, `read_memory(addr, length)`
    * `is_valid_addr(addr)`
    * `read_int_from_memory(addr)`
    * `read_cstring_from_memory(addr, max_length=None, ascii_only=True)`
    * `read_physmem(paddr, size)`, `write_physmem(paddr, data)`
    * `read_physmem_secure(paddr, size)`, `write_physmem_secure(paddr, data)`
* Register access
    * `get_register(regname, use_mbed_exec=False, use_monitor=False)`
* Other
    * `str2bytes(x)`, `bytes2str(x)`
    * `slicer(data, n)`, `slice_unpack(data, n)`
    * `p8`, `p16`, `p32`, `p64`
    * `u8`, `u16`, `u32`, `u64`, `u128`

If you want the complete list, run `gef pyobj-list`.

## I want to add a command, how do I get started?
Copy and paste the `TemplateCommand` class and edit it as you like.

Followings are some notes.
* Class name
    * Rename the newly added command class to any name you like.
    * Make sure to end it with `...Command`.
* Inheritance
    * Make sure you inherit the `GenericCommand` class.
    * This is the condition for registering the command.
* Help strings
    * `_cmdline_`, `_category_`, `_aliases_`, `_syntax_`, `_example_` and `_note_` are used by `help` or `gef help`.
* `__init__()`
    * This method is executed only once, when GEF starts.
    * There is usually no need to override this method.
    * Delete it if you don't need to do anything special.
* `do_invoke()`
    * It is important to override this method.
    * When a command is executed, it starts from this method.
* Arguments to command
    * They should be controlled with the `argparse` module.
    * They are handled by the `parse_args` decorator of the `do_invoke()` method.
* Command execution conditions
    * Add decorators to the `do_invoke()` method as needed.
    * You can check the list of decorators that can be added with `gef pyobj-list`.
* Other
    * If you do not want to execute the same command again when you press ENTER on a blank line after executing the command, please write `self.dont_repeat()`.
    * Use the `gef_print()` function instead of the `print()` function whenever possible.


# About development schedule

## Are there any plans to support kernels for other architectures?
There are no plans.

## Are there any plans to support more architectures with qemu-user?
Yes. However, it is becoming difficult to find new support targets.

This is because three things are required:
1. toolchain
    * linux-headers, binutils, gcc, glibc (or uclibc) are needed.
    * Prebuilt tarball is prefer.
2. qemu-user
    * It needs implementation of gdb-stub.
3. gdb
    * It needs python3-support.


# About reporting, etc.

## I found a bug.
Please feel free to report it on the issue page. I will respond as soon as possible.

## Can you please add this feature? / I don't like a certain feature, so please fix it.
I will consider it, so please report it on the issue page.
But this is a personal development, so I have the final decision. I appreciate your understanding.

## What information should I provide when reporting a issue?
You will need a screenshot or a copy of the terminal output when the problem occurred.
In addition, I am glad if there are the results of the `version` command and `arch-info` command.

Additionally, if the issue is related to kernel debugging, please provide a set of environments (`run.sh`, `bzImage`, `rootfs`, etc.) or where to get them.

## Is it okay to fork and modify?
Yes. However, please follow the license.


# About me

## What kind of environment are you developing in?
I don't use anything special.
I am using a test environment built with buildroot while viewing the Linux kernel source with Bootlin.
I also use images of kernel exploit task from various CTFs.

## Why is there no master branch?
I'm a `git` beginner, so one `dev` branch is the best I can do.

## Are you bad at English?
Yes, I mostly use Google Translate. This is the reason for the inconsistent writing style.
If the expression in English is strange, please feel free to correct it on the issue page.

## I would like to contact you about a GEF?
Find `bata___` on Discord of official GEF's server or Pwndbg's server.
Or send DM to [`@bata_24` on X](https://twitter.com/bata_24).


# Other memo (Japanese)
* Why I decided to make this
    * [gefを改造した話](https://hackmd.io/@bata24/rJVtBJsrP)
* The story behind each command, etc.
    * [bata24/gefの機能紹介とか](https://hackmd.io/@bata24/SycIO4qPi)
