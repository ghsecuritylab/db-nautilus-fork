```

    _   __               __   _  __                
   / | / /____ _ __  __ / /_ (_)/ /__  __ _____    
  /  |/ // __ `// / / // __// // // / / // ___/ 
 / /|  // /_/ // /_/ // /_ / // // /_/ /(__  )     
/_/ |_/ \__,_/ \__,_/ \__//_//_/ \__,_//____/


```

# The Nautilus AeroKernel

Nautilus is an example of an AeroKernel, a very thin kernel-layer exposed 
(much like a library OS, or libOS) directly to the runtime and application. 
AeroKernels are suited particularly well for parallel runtimes that need fine-grained,
explicit control of the machine to squeeze every bit of performance out of it. Note that
an AeroKernel does not, by default, have a user-mode! There are several reasons for this, 
simplicity and performance among the most important. Furthermore, there are no heavy-weight
processes---only threads, all of which share an address space. Therefore, Nautilus is an 
example of a single address-space OS (SASOS). The runtime can implement user-mode features
or address space isolation if this is required for its execution model.

# Hybrid Runtimes (HRTs) and Hybrid Virtual Machines (HVM)

We call the combination of an AeroKernel and the runtime/application using it
a Hybrid Runtime, in that it is both a runtime *and* a kernel, esp. regarding its
ability to use the full machine and determine the proper abstractions to the raw hardware
(if the runtime developer sees a mismatch with his/her needs and the AeroKernel mechanisms, 
they can be overridden). 

If stronger isolation or more complete POSIX/Linux compatibility is required, it is useful
to run the HRT in the context of a Hybrid Virtual Machine. An HVM allows a virtual machine
to split the hardware resources among a regular OS (ROS) and an HRT. The HRT portion of the 
HVM can then be seen as a kind of software accelerator. Note that because of the simplicity 
of the hardware abstractions in a typical HRT, virtualization overheads are much, much less
significant than in, e.g. a Linux guest. 

# Prerequisites

- gcc cross compiler (more on this to come)
- grub version >= 2.02
- xorriso (for CD burning)
- QEMU recommended if you don't want to run on raw hardware

# Hardware Support

Nautilus works with the following hardware:

- x86_64 machines (AMD and Intel)
- Intel Xeon Phi, both KNC and KNL (see http://philix.halek.co for instructions)
- As a Hybrid Virtual Machine (HVM) in the Palacios VMM (see http://v3vee.org/palacios)

# Building

First, configure Nautilus by running 

`make menuconfig`

Select any options you require, then 
run `make` to build the HRT binary image. To make a bootable CD-ROM, 
run `make isoimage`. If you see weird errors, chances are there
is something wrong with your GRUB2 toolchain (namely, `grub-mkrescue`). Make sure `grub-mkrescue`
knows where its libraries are, especially if you've installed the
latest GRUB from source. Use `grub-mkrescue -d`. We've run into issues with naming of
the GRUB2 binaries, in which case a workaround with symlinks was sufficient.

On newer systems, Grub 2 renamed the binaries, so you might want to symlink to
them, e.g. as follows:

```
$> ln -s /usr/bin/grub2-mkrescue /usr/bin/grub-mkrescue
```


# Running and Debugging under QEMU

Here's an example:

[![asciicast](https://asciinema.org/a/xT76jwP0Qe9H7w3nKAk7gkTmv.png)](https://asciinema.org/a/xT76jwP0Qe9H7w3nKAk7gkTmv)

Recommended:

`$> qemu-system-x86_64 -cdrom nautilus.iso -m 2048`

Nautilus has multicore support, so this will also work just fine:

`$> qemu-system-x86_64 -cdrom nautilus.iso -m 2048 -smp 4`

You should see Nautilus boot up on all 4 cores.

Nautilus is a NUMA-aware AeroKernel. To see this in action, try (with a sufficiently new
version of QEMU):

```
$> qemu-system-x86_64 -cdrom nautilus.iso \
                      -m 8G \
                      -numa node,nodeid=0,cpus=0-1 \
                      -numa node,nodeid=1,cpus=2-3 \
                      -smp 4,sockets=2,cores=2,threads=1
```

Nautilus supports debugging over the serial port. This is useful if you want to
debug a physical machine remotely. All prints after the serial port has been
initialized will be redirected to COM1. To use this, find the SERIAL_REDIRECT
entry and enable it in `make menuconfig`. You can now run like this:

`$> qemu-system-x86_64 -cdrom nautilus.iso -m 2G -serial stdio`

Sometimes it is useful to interact with the Nautilus root shell via serial port,
e.g. when you're running under QEMU on a system that does not have a windowing
system. You'll want to first put a character device on the serial port by
rebuilding Nautilus after selecting the *Place a virtual console interface on a character device* option.
Then, after Nautilus boots (making sure you enabled the `-serial stdio` option
in QEMU) you'll see a virtual console at your terminal. You can get to the root
shell by getting to the terminal list with `\``3`. You can then select the root
shell, and you will be able to run shell commands and see output. If you want to
see more kernel output, you can use serial redirection and serial mirroring in
your config.

If you'd like to use Nautilus networking with QEMU, you should use a TUN/TAP
interface. First, you can run the following on your host machine:

```
$> sudo tunctl -d tap0
$> sudo tunctl -t tap0
$> sudo ifconfig tap0 up 10.10.10.2 netmask 255.255.255.0
```

Then you can use the tap interface with QEMU as follows. This particular
invocation attaches both a virtual e1000 fast ethernet card and a virtio
network interface:

```
sudo qemu-system-x86_64 -smp 2 \ 
                        -m 2048 \
                        -vga std \
                        -serial stdio \
                        -cdrom nautilus.iso \
                        -netdev tap,ifname=tap0,script=no,id=net0 \
                            -device virtio-net,netdev=net0 \
                        -netdev user,id=net1 \
                            -device e1000,netdev=net1 \
                        -drive if=none,id=hd0,format=raw,file=nautilus.iso \
                            -device virtio-blk,drive=hd0
```


# Running and Debugging under BOCHS

While we recommend using QEMU, sometimes it is nice to use the native debugging 
support in BOCHS. We've used BOCHS successfully with version 2.6.8. You must have
a version of BOCHS that is built with x86_64 support, which does not seem to be the
default in a lot of package repos. We had to build it manually. You probably also 
want to enable the native debugger.

Here is a BOCHS config file (`~/.bochsrc`) that we used successfully:

```
ata0-master: type=cdrom, path=nautilus.iso, status=inserted
boot: cdrom
com1: enabled=1, mode=file, dev=serial.out
cpu: count=2
cpuid: level=6, mmx=1, level=6, x86_64=1, 1g_pages=1
megs: 2048
```

# Running and Debugging under Gem5

You can configure and build Nautilus for execution in the Gem5
architectural simulator (http://gem5.org).  Note that Gem5 is very
slow.  Simulated time is 2-3 orders of magnitude slower than
real-time.  If you care about interaction, and not simulation
accuracy, configure Nautilus to override the APIC timing calibration
results, a suboption under the Gem5 target architecture.  Once you
have built the kernel for the Gem5 target architecture, you can copy
nautilus.bin to ~gem5/binaries, and run it using Gem5's example full
system configuration (~gem5/configs/example/fs.py), like this (for two
cpus):

cd ~gem5
build/X86/gem5.opt -d run.out configs/example/fs.py -n 2

Nautilus on Gem5 follows Gem5's boot model for Linux.  If you don't
want to change anything, just symlink binaries/nautilus.bin as the
linux kernel executable the example config expects.  Alternatively,
you can modify the config like this, or do something similar in your
own config:

     test_sys = makeLinuxX86System(...)
+++  test_sys.kernel = binary('nautilus.bin')

Once Gem5 is running, you can debug Nautilus in the following
Gem5-standard ways:

telnet localhost 3456  # access serial0 / com1

gdb binaries/nautilus.bin
(gdb) target remote localhost:7000 # attach debugger to cpu 0
(gdb) set architecture i386:x86-64
(gdb) ...

Note that if you want to interact with Nautilus running on Gem5, you
will need to use the virtual console on a char device ("serial0") to
do so.   If you don't want to interact, please see the "autoexec.bat"
startup script feature in src/arch/gem5/init.c.

# Rapid Development

If you'd like to get started quickly with development, a good way is to use 
[Vagrant](https://www.vagrantup.com/). We've provided a `Vagrantfile` in
the top-level Nautilus directory for provisioning a Vagrant VM which has
pretty much everything you need to develop and run Nautilus. This setup
currently only works for VMWare Fusion/Desktop (which requires the paid Vagrant
VMWare provider). We hope to get this working for VirtualBox, and perhaps AWS
soon. If you already have Vagrant installed, to get started you can do the
following from the top-level Nautilus directory:

```
$> vagrant up
```

This will run for several minutes and provision a VM with all the required
packages. It will automatically clone the latest version of Nautilus and build
it. To connect to the VM, you can ssh into it, and immediately start running
Nautilus. There is a demo put in the VM's nautilus directory which will boot
Nautilus in QEMU with a virtual console on a serial port and the QEMU monitor in
another tmux pane:

```
$> vagrant ssh
[vagrant@localhost] cd nautilus
[vagrant@localhost] . ./demo
```

# Resources

You can find publications related to Nautilus and HRTs/HVMs at 
http://halek.co, http://pdinda.org, http://interweaving.org,
and the labs below.

Our labs:

[HExSA Lab](http://hexsa.halek.co) at IIT 
[Prescience Lab](http://www.presciencelab.org) at Northwestern


# Acknowledgements

Nautilus was made possible by support from the United States National Science
Foundation (NSF) via grant CNS-0709168, the Department of Energy (DOE) via
grant DE-SC0005343, and Sandia National Laboratories through the Hobbes
Project, which is funded by the 2013 Exascale Operating and Runtime Systems
Program under the Office of Advanced Scientific Computing Research in the DOE
Office of Science. Sandia National Laboratories is a multi-program laboratory
managed and operated by Sandia Corporation, a wholly owned subsidiary of
Lockheed Martin Corporation, for the U.S. Department of Energy�s National
Nuclear Security Administration under contract DE-AC04-94AL85000.

Kyle C. Hale (c) 2018 

Illinois Institute of Technology 

Northwestern University 

Numerous people develop Nautilus.  Please see copyrights in the
headers and the commit logs for details.   