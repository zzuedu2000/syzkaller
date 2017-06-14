# syzkaller - linux syscall fuzzer

`syzkaller` is an unsupervised, coverage-guided Linux syscall fuzzer.
It is meant to be used with
[KASAN](https://kernel.org/doc/html/latest/dev-tools/kasan.html) (available upstream with `CONFIG_KASAN=y`),
[KTSAN](https://github.com/google/ktsan) (prototype available),
[KMSAN](https://github.com/google/kmsan) (prototype available),
or [KUBSAN](https://kernel.org/doc/html/latest/dev-tools/ubsan.html) (available upstream with `CONFIG_UBSAN=y`).

Project mailing list: [syzkaller@googlegroups.com](https://groups.google.com/forum/#!forum/syzkaller).
You can subscribe to it with a google account or by sending an email to syzkaller+subscribe@googlegroups.com.

List of [found bugs](docs/found_bugs.md).

How to [report Linux kernel bugs](docs/reporting_linux_kernel_bugs.md).

How to [contribute](docs/contributing.md).

## Usage

The following components are needed to use syzkaller:

 - C compiler with coverage support
 - Linux kernel with coverage additions
 - Virtual machine or a physical device
 - syzkaller itself

Generic steps to set up syzkaller are described below.
More specific information (like the exact steps for a particular host system, VM type and a kernel architecture) can be found on [the wiki](https://github.com/google/syzkaller/wiki).

### C Compiler

Syzkaller is a coverage-guided fuzzer and therefore it needs the kernel to be built with coverage support, which requires a recent GCC version.
Coverage support was submitted to GCC in revision `231296`, released in GCC v6.0.

### Linux Kernel

Besides coverage support in GCC, you also need support for it on the kernel side.
KCOV was committed upstream in Linux kernel version 4.6 and can be enabled by configuring the kernel with `CONFIG_KCOV=y`.
For older kernels you need to backport commit [kernel: add kcov code coverage](https://github.com/torvalds/linux/commit/5c9a8750a6409c63a0f01d51a9024861022f6593).

To enable more syzkaller features and improve bug detection abilities, it's recommended to use additional config options.
See [this page](docs/linux_kernel_configs.md) for details.

### VM Setup

Syzkaller performs kernel fuzzing on slave virtual machines or physical devices.
These slave enviroments are referred to as VMs.
Out-of-the-box syzkaller supports QEMU, kvmtool and GCE virtual machines, Android devices and Odroid C2 boards.

These are the generic requirements for a syzkaller VM:

 - The fuzzing processes communicate with the outside world, so the VM image needs to include
   networking support.
 - The program files for the fuzzer processes are transmitted into the VM using SSH, so the VM image
   needs a running SSH server.
 - The VM's SSH configuration should be set up to allow root access for the identity that is
   included in the `syz-manager`'s configuration.  In other words, you should be able to do `ssh -i
   $SSHID -p $PORT root@localhost` without being prompted for a password (where `SSHID` is the SSH
   identification file and `PORT` is the port that are specified in the `syz-manager` configuration
   file).
 - The kernel exports coverage information via a debugfs entry, so the VM image needs to mount
   the debugfs filesystem at `/sys/kernel/debug`.

To use QEMU syzkaller VMs you have to install QEMU on your host system, see [QEMU docs](http://wiki.qemu.org/Manual) for details.
The [create-image.sh](tools/create-image.sh) script can be used to create a suitable Linux image.
Detailed steps for setting up syzkaller with QEMU on a Linux host can be found on wiki for [x86-64](docs/setup_ubuntu-host_qemu-vm_x86-64-kernel.md) and [arm64](docs/setup_linux-host_qemu-vm_arm64-kernel.md) kernels.

For some details on fuzzing the kernel on an Android device check out [this wiki page](docs/setup_linux-host_android-device_arm64-kernel.md) and the explicit instructions for an Odroid C2 board are available [here](docs/setup_ubuntu-host_odroid-c2-board_arm64-kernel.md).

### Syzkaller

The syzkaller tools are written in [Go](https://golang.org), so a Go compiler (>= 1.8) is needed
to build them.

Go distribution can be downloaded from https://golang.org/dl/.
Unpack Go into a directory, say, `$HOME/go`.
Then, set `GOROOT=$HOME/go` env var.
Then, add Go binaries to `PATH`, `PATH=$HOME/go/bin:$PATH`.
Then, set `GOPATH` env var to some empty dir, say `GOPATH=$HOME/gopath`.
Then, run `go get -u -d github.com/google/syzkaller/...` to checkout syzkaller sources with all dependencies.
Then, `cd $GOPATH/src/github.com/google/syzkaller` and
build with `make`, which generates compiled binaries in the `bin/` folder.

To build additional syzkaller tools run `make all-tools`.

## Configuration

The operation of the syzkaller `syz-manager` process is governed by a configuration file, passed at
invocation time with the `-config` option.  This configuration can be based on the
[example](syz-manager/config/testdata/qemu.cfg); the file is in JSON format with the
following keys in its top-level object:

 - `http`: URL that will display information about the running `syz-manager` process.
 - `workdir`: Location of a working directory for the `syz-manager` process. Outputs here include:
     - `<workdir>/crashes/*`: crash output files (see [Crash Reports](#crash-reports))
     - `<workdir>/corpus.db`: corpus with interesting programs
     - `<workdir>/instance-x`: per VM instance temporary files
 - `syzkaller`: Location of the `syzkaller` checkout.
 - `vmlinux`: Location of the `vmlinux` file that corresponds to the kernel being tested.
 - `procs`: Number of parallel test processes in each VM (4 or 8 would be a reasonable number).
 - `leak`: Detect memory leaks with kmemleak.
 - `image`: Location of the disk image file for the QEMU instance; a copy of this file is passed as the
   `-hda` option to `qemu-system-x86_64`.
 - `sandbox` : Sandboxing mode, the following modes are supported:
     - "none": don't do anything special (has false positives, e.g. due to killing init)
     - "setuid": impersonate into user nobody (65534), default
     - "namespace": use namespaces to drop privileges
       (requires a kernel built with `CONFIG_NAMESPACES`, `CONFIG_UTS_NS`,
       `CONFIG_USER_NS`, `CONFIG_PID_NS` and `CONFIG_NET_NS`)
 - `enable_syscalls`: List of syscalls to test (optional).
 - `disable_syscalls`: List of system calls that should be treated as disabled (optional).
 - `suppressions`: List of regexps for known bugs.
 - `type`: Type of virtual machine to use, e.g. `qemu` or `adb`.
 - `vm`: object with VM-type-specific parameters; for example, for `qemu` type paramters include:
     - `count`: Number of VMs to run in parallel.
     - `kernel`: Location of the `bzImage` file for the kernel to be tested;
       this is passed as the `-kernel` option to `qemu-system-x86_64`.
     - `cmdline`: Additional command line options for the booting kernel, for example `root=/dev/sda1`.
     - `sshkey`: Location (on the host machine) of an SSH identity to use for communicating with
       the virtual machine.
     - `cpu`: Number of CPUs to simulate in the VM (*not currently used*).
     - `mem`: Amount of memory (in MiB) for the VM; this is passed as the `-m` option to `qemu-system-x86_64`.

See also [config.go](syz-manager/config/config.go) for all config parameters.


## Running syzkaller

Start the `syz-manager` process as:
```
./bin/syz-manager -config my.cfg
```

The `-config` command line option gives the location of the configuration file [described above](#configuration).

The `syz-manager` process will wind up QEMU virtual machines and start fuzzing in them.
Found crashes, statistics and other information is exposed on the HTTP address provided in manager config.


## Process Structure

The process structure for the syzkaller system is shown in the following diagram;
red labels indicate corresponding configuration options.

![Process structure for syzkaller](structure.png?raw=true)

The `syz-manager` process starts, monitors and restarts several VM instances (support for
physical machines is not implemented yet), and starts a `syz-fuzzer` process inside of the VMs.
It is responsible for persistent corpus and crash storage. As opposed to `syz-fuzzer` processes,
it runs on a host with stable kernel which does not experience white-noise fuzzer load.

The `syz-fuzzer` process runs inside of presumably unstable VMs (or physical machines under test).
The `syz-fuzzer` guides fuzzing process itself (input generation, mutation, minimization, etc)
and sends inputs that trigger new coverage back to the `syz-manager` process via RPC.
It also starts transient `syz-executor` processes.

Each `syz-executor` process executes a single input (a sequence of syscalls).
It accepts the program to execute from the `syz-fuzzer` process and sends results back.
It is designed to be as simple as possible (to not interfere with fuzzing process),
written in C++, compiled as static binary and uses shared memory for communication.

## Crash Reports

When `syzkaller` finds a crasher, it saves information about it into `workdir/crashes` directory. The directory contains one subdirectory per unique crash type. Each subdirectory contains a `description` file with a unique string identifying the crash (intended for bug identification and deduplication); and up to 100 `logN` and `reportN` files, one pair per test machine crash:
```
 - crashes/
   - 6e512290efa36515a7a27e53623304d20d1c3e
     - description
     - log0
     - report0
     - log1
     - report1
     ...
   - 77c578906abe311d06227b9dc3bffa4c52676f
     - description
     - log0
     - report0
     ...
```

Descriptions are extracted using a set of [regular expressions](report/report.go#L33). This set may need to be extended if you are using a different kernel architecture, or are just seeing a previously unseen kernel error messages.

`logN` files contain raw `syzkaller` logs and include kernel console output as well as programs executed before the crash. These logs can be fed to `syz-repro` tool for [crash location and minimization](docs/tools_syz-execprog_syz-prog2c_syz-repro.md), or to `syz-execprog` tool for [manual localization](docs/executing_syzkaller_programs.md). `reportN` files contain post-processed and symbolized kernel crash reports (e.g. a KASAN report). Normally you need just 1 pair of these files (i.e. `log0` and `report0`), because they all presumably describe the same kernel bug. However, `syzkaller` saves up to 100 of them for the case when the crash is poorly reproducible, or if you just want to look at a set of crash reports to infer some similarities or differences.

There are 3 special types of crashes:
 - `no output from test machine`: the test machine produces no output whatsoever
 - `lost connection to test machine`: the ssh connection to the machine was unexpectedly closed
 - `test machine is not executing programs`: the machine looks alive, but no test programs were executed for long period of time
Most likely you won't see `reportN` files for these crashes (e.g. if there is no output from the test machine, there is nothing to put into report). Sometimes these crashes indicate a bug in `syzkaller` itself (especially if you see a Go panic message in the logs). However, frequently they mean a kernel lockup or something similarly bad (here are just a few examples of bugs found this way: [1](https://groups.google.com/d/msg/syzkaller/zfuHHRXL7Zg/Tc5rK8bdCAAJ), [2](https://groups.google.com/d/msg/syzkaller/kY_ml6TCm9A/wDd5fYFXBQAJ), [3](https://groups.google.com/d/msg/syzkaller/OM7CXieBCoY/etzvFPX3AQAJ)).

## Syscall description

`syzkaller` uses declarative description of syscalls to generate, mutate, minimize,
serialize and deserialize programs (sequences of syscalls). See details about the
format and extending the descriptions [here](docs/syscall_descriptions.md).

## Troubleshooting

Here are some things to check if there are problems running syzkaller.

 - Check that QEMU can successfully boot the virtual machine.  For example,
   if `IMAGE` is set to the VM's disk image (as per the `image` config value)
   and `KERNEL` is set to the test kernel (as per the `kernel` config value)
   then something like the following command should start the VM successfully:

       ```qemu-system-x86_64 -hda $IMAGE -m 256 -net nic -net user,host=10.0.2.10,hostfwd=tcp::23505-:22 -enable-kvm -kernel $KERNEL -append root=/dev/sda```

 - Check that inbound SSH to the running virtual machine works.  For example, with
   a VM running and with `SSHKEY` set to the SSH identity (as per the `sshkey` config value) the
   following command should connect:

       ```ssh -i $SSHKEY -p 23505 root@localhost```

 - Check that the `CONFIG_KCOV` option is available inside the VM:
    - `ls /sys/kernel/debug       # Check debugfs mounted`
    - `ls /sys/kernel/debug/kcov  # Check kcov enabled`
    - Build the test program from `Documentation/kcov.txt` and run it inside the VM.

 - Check that debug information (from the `CONFIG_DEBUG_INFO` option) is available
    - Pass the hex output from the kcov test program to `addr2line -a -i -f -e $VMLINUX` (where
      `VMLINUX` is the vmlinux file, as per the `vmlinux` config value), to confirm
      that symbols for the kernel are available.

 - Use the `-v N` command line option to increase the amount of logging output, from both
   the `syz-manager` top-level program and the `syz-fuzzer` instances (which go to the
   output files in the `crashes` subdirectory of the working directory). Higher values of
   N give more output.

 - If logging indicates problems with the executor program (e.g. `executor failure`),
   try manually running a short sequence of system calls:
     - Build additional tools with `make all-tools`
     - Copy `syz-executor` and `syz-execprog` into a running VM.
     - In the VM run `./syz-execprog -executor ./syz-executor -debug sampleprog` where
       sampleprog is a simple system call script (e.g. just containing `getpid()`).
     - For example, if this reports that `clone` has failed, this probably indicates
       that the test kernel does not include support for all of the required namespaces.
       In this case, running the `syz-execprog` test with the `-nobody=0` option fixes the problem,
       so the main configuration needs to be updated to set `dropprivs` to `false`.

## External Articles

 - [Kernel QA with syzkaller and qemu](https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syzkaller_general.md) (tutorial on how to setup syzkaller with qemu)
 - [Syzkaller crash DEMO](https://github.com/hardenedlinux/Debian-GNU-Linux-Profiles/blob/master/docs/harbian_qa/fuzz_testing/syzkaller_crash_demo.md) (tutorial on how to extend syzkaller with new syscalls)
 - [Coverage-guided kernel fuzzing with syzkaller](https://lwn.net/Articles/677764/) (by David Drysdale)
 - [ubsan, kasan, syzkaller und co](http://www.strlen.de/talks/debug-w-syzkaller.pdf) ([video](https://www.youtube.com/watch?v=Acp0A9X1254)) (by Florian Westphal)
 - [Debugging a kernel crash found by syzkaller](http://vegardno.blogspot.de/2016/08/sync-debug.html) (by Quentin Casasnovas)
 - [Linux Plumbers 2016 talk slides](https://docs.google.com/presentation/d/1iAuTvzt_xvDzS2misXwlYko_VDvpvCmDevMOq2rXIcA/edit?usp=sharing)
 - [syzkaller: the next gen kernel fuzzer](https://www.slideshare.net/DmitryVyukov/syzkaller-the-next-gen-kernel-fuzzer) (basics of operations, tutorial on how to run syzkaller and how to extend it to fuzz new drivers)

## Disclaimer

This is not an official Google product.
