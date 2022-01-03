
Device Tree: is a data structure describing the hardware components of a
particular computer so that the operating system's kernel can use and manage 
those components, including the CPU or CPUs, the memory, the buses and the 
peripherals.

The primary purpose of Device Tree in Linux is to provide a way to 
describe non-discoverable hardware. This information was previously 
hard coded in source code. 


The "Open Firmware Device Tree", or simply Device Tree (DT), is a data
structure and language for describing hardware.  More specifically, it
is a description of hardware that is readable by an operating system
so that the operating system doesn't need to hard code details of the
machine.


Structurally, the DT is a tree, or acyclic graph with named nodes, and
nodes may have an arbitrary number of named properties encapsulating
arbitrary data.

Linux uses DT data for three major purposes:
1) platform identification,
2) runtime configuration, and
3) device population.

Platform Identification
======================

First and foremost, the kernel will use data in the DT to identify the
specific machine.  In a perfect world, the specific platform shouldn't
matter to the kernel because all platform details would be described
perfectly by the device tree in a consistent and reliable manner.
Hardware is not perfect though, and so the kernel must identify the
machine during early boot so that it has the opportunity to run
machine-specific fixups.

In the majority of cases, the machine identity is irrelevant, and the
kernel will instead select setup code based on the machine's core
CPU or SoC.  On ARM for example, setup_arch() in
arch/arm/kernel/setup.c will call setup_machine_fdt() in
arch/arm/kernel/devtree.c which searches through the machine_desc
table and selects the machine_desc which best matches the device tree
data.  It determines the best match by looking at the 'compatible'
property in the root device tree node, and comparing it with the
dt_compat list in struct machine_desc (which is defined in
arch/arm/include/asm/mach/arch.h if you're curious).

The 'compatible' property contains a sorted list of strings starting
with the exact name of the machine, followed by an optional list of
boards it is compatible with sorted from most compatible to least.  For
example, the root compatible properties for the TI BeagleBoard and its
successor, the BeagleBoard xM board might look like, respectively:

	compatible = "ti,omap3-beagleboard", "ti,omap3450", "ti,omap3";
	compatible = "ti,omap3-beagleboard-xm", "ti,omap3450", "ti,omap3";

Where "ti,omap3-beagleboard-xm" specifies the exact model, it also
claims that it compatible with the OMAP 3450 SoC, and the omap3 family
of SoCs in general.  You'll notice that the list is sorted from most
specific (exact board) to least specific (SoC family).

Runtime configuration
-------------------------
In most cases, a DT will be the sole method of communicating data from
firmware to the kernel, so also gets used to pass in runtime and
configuration data like the kernel parameters string and the location
of an initrd image.

Most of this data is contained in the /chosen node, and when booting
Linux it will look something like this:

	chosen {
		bootargs = "console=ttyS0,115200 loglevel=8";
		initrd-start = <0xc8000000>;
		initrd-end = <0xc8200000>;
	};

The bootargs property contains the kernel arguments, and the initrd-*
properties define the address and size of an initrd blob.  Note that
initrd-end is the first address after the initrd image, so this doesn't
match the usual semantic of struct resource.  The chosen node may also
optionally contain an arbitrary number of additional properties for
platform-specific configuration data.

During early boot, the architecture setup code calls of_scan_flat_dt()
several times with different helper callbacks to parse device tree
data before paging is setup.  The of_scan_flat_dt() code scans through
the device tree and uses the helpers to extract information required
during early boot.  Typically the early_init_dt_scan_chosen() helper
is used to parse the chosen node including kernel parameters,
early_init_dt_scan_root() to initialize the DT address space model,
and early_init_dt_scan_memory() to determine the size and
location of usable RAM.

On ARM, the function setup_machine_fdt() is responsible for early
scanning of the device tree after selecting the correct machine_desc
that supports the board.


Device population
---------------------
After the board has been identified, and after the early configuration data
has been parsed, then kernel initialization can proceed in the normal
way.  At some point in this process, unflatten_device_tree() is called
to convert the data into a more efficient runtime representation.
This is also when machine-specific setup hooks will get called, like
the machine_desc .init_early(), .init_irq() and .init_machine() hooks
on ARM.  The remainder of this section uses examples from the ARM
implementation, but all architectures will do pretty much the same
thing when using a DT.

As can be guessed by the names, .init_early() is used for any machine-
specific setup that needs to be executed early in the boot process,
and .init_irq() is used to set up interrupt handling.  Using a DT
doesn't materially change the behaviour of either of these functions.
If a DT is provided, then both .init_early() and .init_irq() are able
to call any of the DT query functions (of_* in include/linux/of*.h) to
get additional data about the platform.

The most interesting hook in the DT context is .init_machine() which
is primarily responsible for populating the Linux device model with
data about the platform.  Historically this has been implemented on
embedded platforms by defining a set of static clock structures,
platform_devices, and other data in the board support .c file, and
registering it en-masse in .init_machine().  When DT is used, then
instead of hard coding static devices for each platform, the list of
devices can be obtained by parsing the DT, and allocating device
structures dynamically.

The simplest case is when .init_machine() is only responsible for
registering a block of platform_devices.  A platform_device is a concept
used by Linux for memory or I/O mapped devices which cannot be detected
by hardware, and for 'composite' or 'virtual' devices (more on those
later).  While there is no 'platform device' terminology for the DT,
platform devices roughly correspond to device nodes at the root of the
tree and children of simple memory mapped bus nodes.


example:
About now is a good time to lay out an example.  Here is part of the
device tree for the NVIDIA Tegra board.

/{
	compatible = "nvidia,harmony", "nvidia,tegra20";
	#address-cells = <1>;
	#size-cells = <1>;
	interrupt-parent = <&intc>;

	chosen { };
	aliases { };

	memory {
		device_type = "memory";
		reg = <0x00000000 0x40000000>;
	};

	soc {
		compatible = "nvidia,tegra20-soc", "simple-bus";
		#address-cells = <1>;
		#size-cells = <1>;
		ranges;

		intc: interrupt-controller@50041000 {
			compatible = "nvidia,tegra20-gic";
			interrupt-controller;
			#interrupt-cells = <1>;
			reg = <0x50041000 0x1000>, < 0x50040100 0x0100 >;
		};

		serial@70006300 {
			compatible = "nvidia,tegra20-uart";
			reg = <0x70006300 0x100>;
			interrupts = <122>;
		};

		i2s1: i2s@70002800 {
			compatible = "nvidia,tegra20-i2s";
			reg = <0x70002800 0x100>;
			interrupts = <77>;
			codec = <&wm8903>;
		};

		i2c@7000c000 {
			compatible = "nvidia,tegra20-i2c";
			#address-cells = <1>;
			#size-cells = <0>;
			reg = <0x7000c000 0x100>;
			interrupts = <70>;

			wm8903: codec@1a {
				compatible = "wlf,wm8903";
				reg = <0x1a>;
				interrupts = <347>;
			};
		};
	};

	sound {
		compatible = "nvidia,harmony-sound";
		i2s-controller = <&i2s1>;
		i2s-codec = <&wm8903>;
	};
};
