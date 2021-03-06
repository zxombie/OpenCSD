AutoFDO and ARM Trace   {#AutoFDO}
=====================

@brief Using CoreSight trace and perf with OpenCSD for AutoFDO.

## Introduction

Feedback directed optimization (FDO, also know as profile guided
optimization - PGO) uses a profile of a program's execution to guide the
optmizations performed by the compiler.  Traditionally, this involves
building an instrumented version of the program, which records a profile of
execution as it runs.  The instrumentation adds significant runtime
overhead, possibly changing the behaviour of the program and it may not be
possible to run the instrumented program in a production environment
(e.g. where performance criteria must be met).

AutoFDO uses facilities in the hardware to sample the behaviour of the
program in the production environment and generate the execution profile.
An improved profile can be obtained by including the branch history
(i.e. a record of the last branches taken) when generating an instruction
samples.  On Arm systems, the ETM can be used to generate such records.

The process can be broken down into the following steps:

* Record execution trace of the program
* Convert the execution trace to instruction samples with branch histories
* Convert the instruction samples to source level profiles
* Use the source level profile with the compiler

This article describes how to enable ETM trace on Arm targets running Linux
and use the ETM trace to generate AutoFDO profiles and compile an optimized
program.


## Execution trace on Arm targets

Debug and trace of Arm targets is provided by CoreSight.  This consists of
a set of components that allow access to debug logic, record (trace) the
execution of a processor and route this data through the system, collecting
it into a store.

To record the execution of a processor, we require the following
components:

* A trace source.  The core contains a trace unit, called an ETM that emits
  data describing the instructions executed by the core.
* Trace links.  The trace data generated by the ETM must be moved through
  the system to the component that collects the data (sink).  Links
  include:
    * Funnels: merge multiple streams of data
    * FIFOs: buffer data to smooth out bursts
    * Replicators: send a stream of data to multiple components
* Sinks.  These receive the trace data and store it or send it to an
  external device:
    * ETB: A small circular buffer (64-128 kilobytes) that stores the most
      recent data
    * ETR: A larger (several megabytes) buffer that uses system RAM to
      store data
    * TPIU: Sends data to an off-chip capture device (e.g. Arm DSTREAM) 

Each Arm SoC design may have a different layout (topology) of components.
This topology is described to the OS drivers by the platform's devicetree
or (in future) ACPI firmware.

For application profiling, we need to store several megabytes of data
within the system, so will use ETR with the capture tool (perf)
periodically draining the buffer to a file.

Even though we have a large capture buffer, the ETM can still generate a
lot of data very quickly - typically an ETM will generate ~1 bit of data
per instruction (depending on the workload), which results in 256Mbytes per
second for a core running at 2GHz.  This leads to problems storing and
decoding such large volumes of data.  AutoFDO uses samples of program
execution, so we can avoid this problem by using the ETM's features to
only record small slices of execution - e.g. collect ~5000 cycles of data
every 50M cycles.  This reduces the data rate to a manageable level - a few
megabytes per minute.  This technique is known as 'strobing'.
 

## Enabling trace

### Driver support

To collect ETM trace, the CoreSight drivers must be included in the
kernel.  Some of the driver support is not yet included in the mainline
kernel and many targets are using older kernels.  To enable CoreSight trace
on these targets, Arm have provided backports of the latest CoreSight
drivers and ETM strobing patch at:

  <http://linux-arm.org/git?p=linux-coresight-backports.git>

This repository can be cloned with:

```
git clone git://linux-arm.org/linux-coresight-backports.git
```

You can include these backports in your kernel by either merging the
appropriate branch using git or generating patches (using `git
format-patch`).

For 5.x based kernel onwards, the only patch which needs to be applied is the one enabling strobing - etm4x: `Enable strobing of ETM`.

For 4.9 based kernels, use the `coresight-4.9-etr-etm_strobe` branch:

```
git merge coresight-4.9-etr-etm_strobe
```

or

```
git format-patch --output-directory /output/dir v4.9..coresight-4.9-etr-etm_strobe
cd my_kernel
git am /output/dir/*.patch # or patch -p1 /output/dir/*.patch if not using git
```

For 4.14 based kernels, use the `coresight-4.14-etm_strobe` branch:

```
git merge coresight-4.14-etm_strobe
```

or

```
git format-patch --output-directory /output/dir v4.14..coresight-4.14-etm_strobe
cd my_kernel
git am /output/dir/*.patch # or patch -p1 /output/dir/*.patch if not using git
```

The CoreSight trace drivers must also be enabled in the kernel
configuration.  This can be done using the configuration menu (`make
menuconfig`), selecting `Kernel hacking` / `arm64 Debugging`  /`CoreSight Tracing Support` and
enabling all options, or by setting the following in the configuration
file:

```
CONFIG_CORESIGHT=y
CONFIG_CORESIGHT_LINK_AND_SINK_TMC=y
CONFIG_CORESIGHT_SINK_TPIU=y
CONFIG_CORESIGHT_SOURCE_ETM4X=y
CONFIG_CORESIGHT_DYNAMIC_REPLICATOR=y
CONFIG_CORESIGHT_STM=y
CONFIG_CORESIGHT_CATU=y
```

Compile the kernel for your target in the usual way, e.g.

```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- 
```

Each target may have a different layout of CoreSight components.  To
collect trace into a sink, the kernel drivers need to know which other
devices need to be configured to route data from the source to the sink.
This is described in the devicetree (and in future, the ACPI tables).  The
device tree will define which CoreSight devices are present in the system,
where they are located and how they are connected together.  The devicetree
for some platforms includes a description of the platform's CoreSight
components, but in other cases you may have to ask the platform/SoC vendor
to supply it or create it yourself (see Appendix: Describing CoreSight in
Devicetree).
  
Once the target has been booted with the devicetree describing the
CoreSight devices, you should find the devices in sysfs:

```
# ls /sys/bus/coresight/devices/
etm0  etm2  etm4  etm6  funnel0  funnel2  funnel4      stm0      tmc_etr0
etm1  etm3  etm5  etm7  funnel1  funnel3  replicator0  tmc_etf0
```

The naming convention for etm devices can be different according to the kernel version you're using.
For more information about the naming scheme, please check out the [Linux Kernel Documentation](https://www.kernel.org/doc/html/latest/trace/coresight/coresight.html#device-naming-scheme)

If `/sys/bus/coresight/devices/` is empty, you may want to check out your Kernel configuration to make sure your .config file is including CoreSight dependencies, such as the clock.

### Perf tools

The perf tool is used to capture execution trace, configuring the trace
sources to generate trace, routing the data to the sink and collecting the
data from the sink.

Arm recommends to use the perf version corresponding to the kernel running
on the target.  This can be built from the same kernel sources with

```
make -C tools/perf CORESIGHT=1 VF=1 ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu-
```

When specifying CORESIGHT=1, perf will be built using the installed OpenCSD library.
If you are cross compiling, then additional setup is required to ensure the build process links against the correct version of the library.

If the post-processing (`perf inject`) of the captured data is not being
done on the target, then the OpenCSD library is not required for this build
of perf.

Trace is captured by collecting the `cs_etm` event from perf.  The sink
to collect data into is specified as a parameter of this event.  Trace can
also be restricted to user space or kernel space with 'u' or 'k'
parameters.  For example:

```
perf record -e cs_etm/@tmc_etr0/u --per-thread -- /bin/ls
```

Will record the userspace execution of '/bin/ls' using tmc_etr0 as sink.

## Capturing modes

You can trace a single-threaded program in two different ways:

1. By specifying `--per-thread`, and in this case the CoreSight subsystem will
record only a trace relative to the given program.

2. By NOT specifying `--per-thread`, and in this case CPU-wide tracing will
be enabled. In this scenario the trace will contain both the target program trace
and other workloads that were executing on the same CPU



## Processing trace and profiles

perf is also used to convert the execution trace an instruction profile.
This requires a different build of perf, using the version of perf from
Linux v4.17 or later, as the trace processing code isn't included in the
driver backports.  Trace decode is provided by the OpenCSD library
(<https://github.com/Linaro/OpenCSD>), v0.9.1 or later.  This is packaged
for debian testing (install the libopencsd0, libopencsd-dev packages) or
can be compiled from source and installed. 

The autoFDO tool <https://github.com/google/autofdo> is used to convert the
instruction profiles to source profiles for the GCC and clang/llvm
compilers.


## Recording and profiling

Once trace collection using perf is working, we can now use it to profile
an application.

The application must be compiled to include sufficient debug information to
map instructions back to source lines.  For GCC, use the `-g1` or `-gmlt`
options.  For clang/llvm, also add the `-fdebug-info-for-profiling` option.

perf identifies the active program or library using the build identifier
stored in the elf file.  This should be added at link time with the compiler
flag `-Wl,--build-id=sha1`.

The next step is to record the execution trace of the application using the
perf tool.  The ETM strobing should be configured before running the perf
tool.  There are two parameters:

  * window size: A number of CPU cycles (W)
  * period: Trace is enabled for W cycle every _period_ * W cycles.
  
For example, a typical configuration is to use a window size of 5000 cycles
and a period of 10000 - this will collect 5000 cycles of trace every 50M
cycles.  With these proof-of-concept patches, the strobe parameters are
configured via sysfs - each ETM will have `strobe_window` and
`strobe_period` parameters in `/sys/bus/coresight/devices/<sink>` and
these values will have to be written to each (In a future version, this
will be integrated into the drivers and perf tool).
The `set_strobing.sh` script in this directory [`<opencsd>/decoder/tests/auto-fdo`] automates this process.

To collect trace from an application using ETM strobing, run:

```
sudo ./set_strobing.sh 5000 10000
perf record -e cs_etm/@tmc_etr0/u --per-thread -- <your app>"
```

The raw trace can be examined using the `perf report` command:

```
perf report -D -i perf.data --stdio
```

Perf needs to be built from your linux kernel version souce code repository against the OpenCSD library in order to be able to properly read ETM-gathered samples and post-process them.
If running `perf report` produces an error like:

```
0x1f8 [0x268]: failed to process type: 70 [Operation not permitted]
Error:
failed to process sample
```
or

```
"file uses a more recent and unsupported ABI (8 bytes extra). incompatible file format".
```

You are probably using a perf version which is not using this library: please make sure to install this project in your system by either compiling it from [Source Code]( <https://github.com/Linaro/OpenCSD>) from v0.9.1 or later and compile perf using this library.
Otherwise, this project is packaged for debian (install the libopencsd0, libopencsd-dev packages).


For example:

```
0x1d370 [0x30]: PERF_RECORD_AUXTRACE size: 0x2003c0  offset: 0  ref: 0x39ba881d145f8639  idx: 0  tid: 4551  cpu: -1

. ... CoreSight ETM Trace data: size 2098112 bytes
        Idx:0; ID:12;   I_ASYNC : Alignment Synchronisation.
        Idx:12; ID:12;  I_TRACE_INFO : Trace Info.; INFO=0x0
        Idx:17; ID:12;  I_ADDR_L_64IS0 : Address, Long, 64 bit, IS0.; Addr=0xFFFF000008A4991C; 
        Idx:48; ID:14;  I_ASYNC : Alignment Synchronisation.
        Idx:60; ID:14;  I_TRACE_INFO : Trace Info.; INFO=0x0
        Idx:65; ID:14;  I_ADDR_L_64IS0 : Address, Long, 64 bit, IS0.; Addr=0xFFFF000008A4991C; 
        Idx:96; ID:14;  I_ASYNC : Alignment Synchronisation.
        Idx:108; ID:14; I_TRACE_INFO : Trace Info.; INFO=0x0
        Idx:113; ID:14; I_ADDR_L_64IS0 : Address, Long, 64 bit, IS0.; Addr=0xFFFF000008A4991C; 
        Idx:122; ID:14; I_TRACE_ON : Trace On.
        Idx:123; ID:14; I_ADDR_CTXT_L_64IS0 : Address & Context, Long, 64 bit, IS0.; Addr=0x0000000000407B00; Ctxt: AArch64,EL0, NS; 
        Idx:134; ID:14; I_ATOM_F3 : Atom format 3.; ENN
        Idx:135; ID:14; I_ATOM_F5 : Atom format 5.; NENEN
        Idx:136; ID:14; I_ATOM_F5 : Atom format 5.; ENENE
        Idx:137; ID:14; I_ATOM_F5 : Atom format 5.; NENEN
        Idx:138; ID:14; I_ATOM_F3 : Atom format 3.; ENN
        Idx:139; ID:14; I_ATOM_F3 : Atom format 3.; NNE
        Idx:140; ID:14; I_ATOM_F1 : Atom format 1.; E
.....
```

The execution trace is then converted to an instruction profile using
the perf build with trace decode support.  This may be done on a different
machine than that which collected the trace (e.g. when cross compiling for
an embedded target).  The `perf inject` command
decodes the execution trace and generates periodic instruction samples,
with branch histories:

!! Careful: if you are using a device different than the one used to collect the profiling data,
you'll need to run `perf buildid-cache` as described below.
```
perf inject -i perf.data -o inj.data --itrace=i100000il
```

The `--itrace` option configures the instruction sample behaviour:

* `i100000i` generates an instruction sample every 100000 instructions
  (only instruction count periods are currently supported, future versions
  may support time or cycle count periods)
* `l` includes the branch histories on each sample
* `b` generates a sample on each branch (not used here)

Perf requires the original program binaries to decode the execution trace.
If running the `inject` command on a different system than the trace was
captured on, then the binary and any shared libraries must be added to
perf's cache with:

```
perf buildid-cache -a /path/to/binary_or_library
```

`perf report` can also be used to show the instruction samples:

```
perf report -D -i inj.data --stdio
.......
0x1528 [0x630]: PERF_RECORD_SAMPLE(IP, 0x2): 4551/4551: 0x434b98 period: 3093 addr: 0
... branch stack: nr:64
.....  0: 0000000000434b58 -> 0000000000434b68 0 cycles  P   0
.....  1: 0000000000436a88 -> 0000000000434b4c 0 cycles  P   0
.....  2: 0000000000436a64 -> 0000000000436a78 0 cycles  P   0
.....  3: 00000000004369d0 -> 0000000000436a60 0 cycles  P   0
.....  4: 000000000043693c -> 00000000004369cc 0 cycles  P   0
.....  5: 00000000004368a8 -> 0000000000436928 0 cycles  P   0
.....  6: 000000000042d070 -> 00000000004368a8 0 cycles  P   0
.....  7: 000000000042d108 -> 000000000042d070 0 cycles  P   0
.......
..... 57: 0000000000448ee0 -> 0000000000448f24 0 cycles  P   0
..... 58: 0000000000448ea4 -> 0000000000448ebc 0 cycles  P   0
..... 59: 0000000000448e20 -> 0000000000448e94 0 cycles  P   0
..... 60: 0000000000448da8 -> 0000000000448ddc 0 cycles  P   0
..... 61: 00000000004486f4 -> 0000000000448da8 0 cycles  P   0
..... 62: 00000000004480fc -> 00000000004486d4 0 cycles  P   0
..... 63: 0000000000448658 -> 00000000004480ec 0 cycles  P   0
 ... thread: program1:4551
 ...... dso: /home/root/program1
.......
```

The instruction samples produced by `perf inject` is then passed to the
autofdo tool to generate source level profiles for the compiler.  For
clang/LLVM:

```
create_llvm_prof -binary=/path/to/binary -profile=inj.data -out=program.llvmprof
```

And for GCC:

```
create_gcov -binary=/path/to/binary -profile=inj.data -gcov_version=1 -gcov=program.gcov
```

The profiles can be viewed with:

```
llvm-profdata show -sample program.llvmprof
```

Or, for GCC:

```
dump_gcov -gcov_version=1 program.gcov
```

## Using profile in the compiler

The profile produced by the above steps can then be passed to the compiler
to optimize the next build of the program.

For GCC, use the `-fauto-profile` option:

```
gcc -O2 -fauto-profile=program.gcov -o program program.c
```

For Clang, use the `-fprofile-sample-use` option:

```
clang -O2 -fprofile-sample-use=program.llvmprof -o program program.c
```


## Summary

The basic commands to run an application and create a compiler profile are:

```
sudo ./set_strobing.sh 5000 10000
perf record -e cs_etm/@tmc_etr0/u --per-thread -- <your app>"
perf inject -i perf.data -o inj.data --itrace=i100000il
create_llvm_prof -binary=/path/to/binary -profile=inj.data -out=program.llvmprof
```

Use `create_gcov` for gcc.


## References

* AutoFDO tool: <https://github.com/google/autofdo>
* GCC's wiki on autofdo: <https://gcc.gnu.org/wiki/AutoFDO>, <https://gcc.gnu.org/wiki/AutoFDO/Tutorial>
* Google paper: <https://ai.google/research/pubs/pub45290>
* CoreSight kernel docs: Documentation/trace/coresight.txt


## Appendix: Describing CoreSight in Devicetree


Each component has an entry in the device tree that describes its:

* type: The `compatible` field defines which driver to use
* location: A `reg` defines the component's address and size on the bus
* clocks: The `clocks` and `clock-names` fields state which clock provides
  the `apb_pclk` clock.
* connections to other components: `port` and `ports` field link the
  component to ports of other components

To create the device tree, some information about the platform is required:

* The memory address of the CoreSight components.  This is the address in
  the CPU's address space where the CPU can access each CoreSight
  component.
* The connections between the components.
  
This information can be found in the SoC's reference manual or you may need
to ask the platform/SoC vendor to supply it.

An ETMv4 source is declared with a section like this:

```
	etm0: etm@22040000 {
		compatible = "arm,coresight-etm4x", "arm,primecell";
		reg = <0 0x22040000 0 0x1000>;

		cpu = <&A72_0>;
		clocks = <&soc_smc50mhz>;
		clock-names = "apb_pclk";
		port {
			cluster0_etm0_out_port: endpoint {
				remote-endpoint = <&cluster0_funnel_in_port0>;
			};
		};
	};
```

This describes an ETMv4 attached to core A72_0, located at 0x22040000, with
its output linked to port 0 of a funnel.  The funnel is described with:

```
	funnel@220c0000 { /* cluster0 funnel */
		compatible = "arm,coresight-funnel", "arm,primecell";
		reg = <0 0x220c0000 0 0x1000>;

		clocks = <&soc_smc50mhz>;
		clock-names = "apb_pclk";
		power-domains = <&scpi_devpd 0>;
		ports {
			#address-cells = <1>;
			#size-cells = <0>;

			port@0 {
				reg = <0>;
				cluster0_funnel_out_port: endpoint {
					remote-endpoint = <&main_funnel_in_port0>;
				};
			};

			port@1 {
				reg = <0>;
				cluster0_funnel_in_port0: endpoint {
					slave-mode;
					remote-endpoint = <&cluster0_etm0_out_port>;
				};
			};

			port@2 {
				reg = <1>;
				cluster0_funnel_in_port1: endpoint {
					slave-mode;
					remote-endpoint = <&cluster0_etm1_out_port>;
				};
			};
		};
	};
```

This describes a funnel located at 0x220c0000, receiving data from 2 ETMs
and sending the merged data to another funnel.  We continue describing
components with similar blocks until we reach the sink (an ETR):

```
	etr@20070000 {
		compatible = "arm,coresight-tmc", "arm,primecell";
		reg = <0 0x20070000 0 0x1000>;
		iommus = <&smmu_etr 0>;

		clocks = <&soc_smc50mhz>;
		clock-names = "apb_pclk";
		power-domains = <&scpi_devpd 0>;
		port {
			etr_in_port: endpoint {
				slave-mode;
				remote-endpoint = <&replicator_out_port1>;
			};
		};
	};
```

Full descriptions of the properties of each component can be found in the
Linux source at Documentation/devicetree/bindings/arm/coresight.txt.
The Arm Juno platform's devicetree (arch/arm64/boot/dts/arm) provides an example
description of CoreSight description.

Many systems include a TPIU for off-chip trace.  While this isn't required
for self-hosted trace, it should still be included in the devicetree.  This
allows the drivers to access it to ensure it is put into a disabled state,
otherwise it may limit the trace bandwidth causing data loss.
