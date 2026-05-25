---
layout: post
title: "How Linux Finds Its Hardware: Device Tree and ACPI"
date: 2026-05-23 12:00:00 +0000
categories: kernel
---

## The problem: hardware isn't self-describing

When a Linux kernel boots, it faces an immediate question: what hardware is actually on this board? On a desktop x86 machine, the firmware answers that question through ACPI tables baked into the BIOS. On an ARM board, there is no BIOS -- the kernel receives a compiled binary called a Device Tree Blob that describes every peripheral, bus, and interrupt line on the system. Both mechanisms solve the same problem, but they do it in very different ways. Recently I have been working with an I2C temperature sensor driver ([MAX31732](https://github.com/torvalds/linux/blob/master/drivers/hwmon/max31732.c)) and it gave me a good excuse to trace through both paths end to end.

## Part 1: Device Tree

### From text to binary

Device tree descriptions start life as human-readable `.dts` (device tree source) files. SoC vendors provide `.dtsi` (device tree source include) files that describe the chip's peripherals, and board manufacturers write `.dts` files that include the SoC dtsi and configure what is actually wired up on a particular board.

Here is a minimal example:

```dts
/dts-v1/;
#include "myvendor-soc.dtsi"

/ {
    model = "My Board v1.0";
    compatible = "myvendor,myboard";

    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x40000000>;  /* 1GB at 0x80000000 */
    };
};

/* Enable and configure the I2C bus defined in the SoC dtsi */
&i2c0 {
    status = "okay";
    clock-frequency = <400000>;

    temp_sensor: temperature@4c {
        compatible = "adi,amax31732";
        reg = <0x4c>;
        interrupt-parent = <&gpio>;
        interrupts = <5 IRQ_TYPE_LEVEL_LOW>, <6 IRQ_TYPE_LEVEL_LOW>;
        interrupt-names = "ALARM1", "ALARM2";
        adi,alarm1-interrupt-mode;
        adi,alarm1-fault-queue = <2>;
    };
};
```

A few things to notice. The `compatible` string follows a `"vendor,device"` convention. The `reg` property means different things depending on the bus -- on a memory-mapped bus it is a base address and size, on I2C it is the slave address (`0x4c`). The `status = "okay"` pattern exists because SoC dtsi files typically define all peripherals as `"disabled"` by default, and board files selectively enable them.

These source files get compiled into a binary blob using `dtc`:

```bash
# Compile DTS to binary DTB
dtc -I dts -O dtb -o board.dtb board.dts

# Decompile a DTB back to readable DTS
dtc -I dtb -O dts -o board.dts board.dtb

# Dump the live device tree from a running kernel
dtc -I dtb -O dts /proc/device-tree > running.dts
```

### The boot handoff

The bootloader (typically U-Boot on ARM) loads two things into RAM: the kernel image and the DTB. It may also patch the DTB in-place before handing off -- injecting the kernel command line into the `/chosen` node, filling in MAC addresses, or adjusting memory regions based on what it detected at runtime.

Then it jumps to the kernel entry point. The critical detail is how the DTB address is passed:

| Architecture | How DTB address is passed |
|---|---|
| ARM (32-bit) | Register `r2` |
| ARM64 | Register `x0` |
| RISC-V | Register `a1` |

That is it. The kernel's very first job is to validate the magic number at the address it received, and then unflatten the blob into an in-memory tree of `struct device_node` objects.

### Kernel boot sequence

Here is what happens step by step once the kernel has the DTB pointer:

1. **`setup_machine_fdt(dtb)`** -- validates the DTB magic, does an early scan for `/chosen` (kernel command line, initrd location) and `/memory` (RAM layout).

2. **`unflatten_device_tree()`** -- walks the flat binary and builds an in-memory tree. Each node becomes a `struct device_node` with pointers to its parent, children, siblings, and a linked list of properties.

3. **`of_platform_populate()`** -- walks the unflattened tree and creates a `struct platform_device` for each node that looks like a hardware device. For I2C and SPI buses, the bus controller driver does this enumeration instead -- the I2C core calls `of_i2c_register_devices()` to create `struct i2c_client` objects for each child node.

4. **Driver matching** -- for each device created, the kernel looks through all registered drivers for a compatible string match.

### The compatible string match

This is the central mechanism that connects hardware descriptions to driver code. A driver declares what it can handle using an `of_device_id` table:

```c
static const struct of_device_id max31732_of_match[] = {
    { .compatible = "adi,amax31732", },
    { },  /* sentinel -- marks end of table */
};
MODULE_DEVICE_TABLE(of, max31732_of_match);
```

The `MODULE_DEVICE_TABLE` macro is important -- it generates a special ELF section that `depmod` reads to build `/lib/modules/$(uname -r)/modules.alias`. This is how the kernel knows to automatically load the right module when a matching device appears. You can see these aliases yourself:

```bash
grep amax31732 /lib/modules/$(uname -r)/modules.alias
# alias of:N*T*Cadi,amax31732* amax31732
```

The match table gets wired into the driver struct:

```c
static struct i2c_driver max31732_driver = {
    .driver = {
        .name           = "amax31732",
        .of_match_table = of_match_ptr(max31732_of_match),
    },
    .probe    = max31732_probe,
    .id_table = max31732_ids,
};
module_i2c_driver(max31732_driver);
```

The `of_match_ptr()` macro compiles out the table reference when `CONFIG_OF` is disabled, so the driver can still build on platforms without device tree support.

### Reading properties in probe()

Once the kernel matches a driver to a device, it calls the driver's `probe()` function. This is where the driver reads hardware-specific configuration from the device tree node. There are two API families for this:

**The `of_` API** -- device-tree specific:

```c
struct device_node *np = dev->of_node;

/* Read a u32 */
u32 speed;
of_property_read_u32(np, "clock-frequency", &speed);

/* Read a string */
const char *name;
of_property_read_string(np, "clock-names", &name);

/* Check a boolean (present/absent) */
if (of_property_read_bool(np, "fifo-enable"))
    port->has_fifo = true;
```

**The `fwnode` API** -- firmware-agnostic (works with both DT and ACPI):

```c
struct fwnode_handle *fwnode = dev_fwnode(dev);

/* Read a u32 */
u32 queue;
fwnode_property_read_u32(fwnode, "adi,alarm1-fault-queue", &queue);

/* Check a boolean */
if (fwnode_property_read_bool(fwnode, "adi,alarm1-interrupt-mode"))
    /* configure interrupt mode */

/* Get an IRQ by name */
int irq = fwnode_irq_get_byname(fwnode, "ALARM1");
```

The MAX31732 driver uses the `fwnode` API throughout, which means it can work on both DT and ACPI systems without any code changes. More on why this matters in Part 3.

### Debugging device tree issues

When things go wrong, the live device tree is exposed as a filesystem:

```bash
# Browse the live device tree
ls /proc/device-tree/
cat /proc/device-tree/model

# Find all nodes with a specific compatible string
grep -rl "amax31732" /proc/device-tree/

# Check if a device was created from DT
ls /sys/bus/i2c/devices/
# 0-004c  1-0050  ...

# Check what driver is bound to a device
ls -la /sys/bus/i2c/devices/0-004c/driver
# -> /sys/bus/i2c/drivers/amax31732

# Check probe failures in dmesg
dmesg | grep -i "amax31732"
```

### Device tree overlays

Overlays allow modifying the device tree at runtime without recompiling the whole DTB. This is commonly used for add-on boards (like Raspberry Pi HATs) or hot-pluggable hardware:

```dts
/dts-v1/;
/plugin/;

/ {
    fragment@0 {
        target = <&i2c0>;
        __overlay__ {
            #address-cells = <1>;
            #size-cells = <0>;

            pressure@76 {
                compatible = "bosch,bmp280";
                reg = <0x76>;
            };
        };
    };
};
```

```bash
# Compile and apply an overlay at runtime
dtc -I dts -O dtb -o sensor.dtbo sensor-overlay.dts
mkdir /sys/kernel/config/device-tree/overlays/sensor
cat sensor.dtbo > /sys/kernel/config/device-tree/overlays/sensor/dtbo

# Remove it
rmdir /sys/kernel/config/device-tree/overlays/sensor
```

---

## Part 2: ACPI

ACPI (Advanced Configuration and Power Interface) is the dominant hardware description mechanism on x86 and is increasingly used on ARM64 server platforms. Where device tree is a static data structure, ACPI includes an entire bytecode interpreter -- it can describe hardware *and* define executable methods for power management, device configuration, and runtime queries.

### From ASL to AML

ACPI tables are authored in ASL (ACPI Source Language) and compiled into AML (ACPI Machine Language) bytecode using Intel's `iasl` compiler. Unlike device tree blobs which are loaded separately by the bootloader, AML tables are embedded directly in the system firmware (BIOS/UEFI ROM).

Here is what an I2C temperature sensor looks like in ASL:

```asl
DefinitionBlock ("", "SSDT", 2, "VENDOR", "TEMPSENS", 0x00000001)
{
    External (\_SB.PCI0.I2C1, DeviceObj)

    Scope (\_SB.PCI0.I2C1)
    {
        Device (TMP0)
        {
            /* Hardware ID -- used for driver matching (like compatible) */
            Name (_HID, "AMAX3173")

            /* Compatible ID -- fallback match (optional) */
            Name (_CID, "PNP0C50")

            /* Device status: present and enabled */
            Name (_STA, 0x0F)

            /* Current Resource Settings -- I2C address, IRQs */
            Name (_CRS, ResourceTemplate ()
            {
                I2cSerialBusV2 (
                    0x004C,             // I2C slave address
                    ControllerInitiated,
                    400000,             // 400kHz
                    AddressingMode7Bit,
                    "\\_SB.PCI0.I2C1",  // I2C controller
                )
                GpioInt (Level, ActiveLow, Shared, PullUp, 0,
                         "\\_SB.PCI0.GPIO")
                {
                    5  // pin 5 -- ALARM1
                }
                GpioInt (Level, ActiveLow, Shared, PullUp, 0,
                         "\\_SB.PCI0.GPIO")
                {
                    6  // pin 6 -- ALARM2
                }
            })

            /* Device-Specific Data -- custom properties (like DT properties) */
            Name (_DSD, Package ()
            {
                ToUUID("daffd814-6eba-4d8c-8a91-bc9bbf4aa301"),
                Package ()
                {
                    Package () { "adi,alarm1-interrupt-mode", 1 },
                    Package () { "adi,alarm1-fault-queue", 2 },
                }
            })
        }
    }
}
```

A few things jump out when comparing this to device tree. The `_HID` (Hardware ID) serves the same role as the `compatible` string, but it follows ACPI's own naming convention -- typically a 4-character vendor prefix plus a 4-character device ID, or a PNP ID. The `_CRS` (Current Resource Settings) bundles what device tree spreads across `reg`, `interrupts`, and other properties into a single encoded resource block. And the `_DSD` (Device Specific Data) is the ACPI equivalent of custom device tree properties -- it was specifically designed to let ACPI carry the same key-value properties that drivers already read via `fwnode_property_read_*()`.

### Compiling and inspecting ACPI tables

```bash
# Compile ASL to AML
iasl -tc sensor.asl
# produces: sensor.aml

# Dump all ACPI tables from a running system
sudo acpidump > acpi_tables.dat

# Extract individual tables
acpixtract -a acpi_tables.dat
# produces: dsdt.dat, ssdt1.dat, ssdt2.dat, ...

# Decompile AML back to readable ASL
iasl -d dsdt.dat
# produces: dsdt.dsl (human-readable ASL)

# List all ACPI tables in sysfs
ls /sys/firmware/acpi/tables/
# DSDT  SSDT1  SSDT2  FACP  MADT  MCFG  ...

# Dump a specific table directly
sudo cat /sys/firmware/acpi/tables/DSDT > dsdt.aml
iasl -d dsdt.aml
```

The `acpidump` + `iasl -d` pipeline is extremely useful for debugging. If a driver is not probing on an ACPI system, the first thing to check is whether the firmware actually defines the device with the right `_HID`.

### The ACPI boot handoff

Unlike device tree where the bootloader explicitly passes a pointer, ACPI tables are discovered through a chain of pointers that starts at a well-known location:

1. **UEFI firmware** places ACPI tables in RAM and publishes the **RSDP** (Root System Description Pointer) via the EFI System Table (`EFI_ACPI_TABLE_GUID`). On legacy BIOS systems, the RSDP sits at a well-known address in the BIOS ROM area (`0x000E0000` to `0x000FFFFF`).

2. **RSDP** points to the **XSDT** (Extended System Description Table), which contains an array of pointers to all other tables.

3. The kernel follows these pointers to find the **DSDT** (Differentiated System Description Table) -- the main table containing device definitions -- plus any **SSDT** (Secondary System Description Tables) that supplement it.

4. The **ACPICA** subsystem (an Intel-maintained AML interpreter embedded in the kernel) parses the bytecode and builds a hierarchical namespace: `\_SB.PCI0.I2C1.TMP0`.

Here is the pointer chain:

```
RSDP  ──>  XSDT  ──>  ┌── DSDT   (device definitions, AML bytecode)
                       ├── SSDT   (supplementary devices)
                       ├── MADT   (interrupt controllers)
                       ├── MCFG   (PCIe configuration)
                       ├── SRAT   (NUMA topology)
                       ├── FACP   (power management)
                       └── ...
```

### ACPI driver matching

Drivers declare what ACPI devices they handle using an `acpi_device_id` table:

```c
static const struct acpi_device_id my_acpi_match[] = {
    { "AMAX3173", 0 },   /* matches _HID = "AMAX3173" */
    { },
};
MODULE_DEVICE_TABLE(acpi, my_acpi_match);

static struct i2c_driver max31732_driver = {
    .driver = {
        .name            = "amax31732",
        .of_match_table  = of_match_ptr(max31732_of_match),  /* DT */
        .acpi_match_table = ACPI_PTR(my_acpi_match),         /* ACPI */
    },
    .probe    = max31732_probe,
    .id_table = max31732_ids,
};
```

The matching precedence when a device appears is: ACPI `_HID`/`_CID` match first (on ACPI systems), then OF `compatible` match (on DT systems), then I2C `id_table` match (fallback for board files or manual instantiation).

### ACPI vs DT: what the driver sees

The beauty of the `_DSD` mechanism is that the driver sees the same property names regardless of whether the system uses ACPI or DT. When the MAX31732 driver calls:

```c
fwnode_property_read_bool(dev_fwnode(dev), "adi,alarm1-interrupt-mode")
```

On a DT system, this reads the boolean property from the device tree node. On an ACPI system, it reads from the `_DSD` package. The driver code is identical -- the `fwnode` layer handles the translation.

### ACPI's runtime capabilities

One major difference from device tree is that ACPI tables contain executable code. The AML bytecode can define methods that the kernel calls at runtime:

| Method | Purpose |
|---|---|
| `_STA` | Query device status (present? enabled?) |
| `_ON` / `_OFF` | Turn device power on/off |
| `_PS0` to `_PS3` | Set device power state (D0=full power, D3=off) |
| `_DSM` | Device-specific method (vendor-defined RPC) |
| `_CRS` | Current resource settings |
| `_SRS` | Set resource settings |
| `_PRT` | PCI routing table (IRQ routing) |
| `_OSC` | OS capabilities negotiation |

This is something device tree fundamentally cannot do. A DTS file is pure data -- it describes what exists but cannot execute logic. ACPI can, for example, define a `_DSM` method that toggles a GPIO to reset a device, or a `_PS3` method that sequences multiple regulators in the right order to power down a complex subsystem.

### Debugging ACPI

```bash
# Check if ACPI is present
ls /sys/firmware/acpi/tables/

# Browse the ACPI namespace
ls /sys/bus/acpi/devices/
# AMAX3173:00  PNP0C50:00  ...

# Check device status
cat /sys/bus/acpi/devices/AMAX3173:00/status
# 15  (0x0F = present + enabled + functioning)

# Check driver binding
ls -la /sys/bus/acpi/devices/AMAX3173:00/driver

# ACPI errors in dmesg
dmesg | grep -i acpi | grep -i error

# Verbose ACPI debugging (boot parameter)
# Add to kernel cmdline: acpi.debug_level=0x2 acpi.debug_layer=0xFFFFFFFF
```

---

## Part 3: The fwnode abstraction

So we have two completely different firmware description mechanisms -- one passes a flat binary blob through a CPU register, the other embeds bytecode in a BIOS ROM. Yet the MAX31732 driver handles both with a single code path. This is the `fwnode` (firmware node) abstraction at work.

```
           Driver code
               |
     fwnode_property_read_u32()
     fwnode_property_read_bool()
     fwnode_irq_get_byname()
               |
       +-------+-------+
       |               |
  OF fwnode        ACPI fwnode
  (Device Tree)    (_DSD properties)
```

The `dev_fwnode(dev)` call returns a `struct fwnode_handle *` that points to either an OF node or an ACPI node, depending on what firmware described the device. The `fwnode_property_*` functions dispatch through a vtable to the right backend.

This is why newer drivers prefer the `fwnode` API over the `of_` API. The `of_` functions like `of_property_read_u32()` only work with device tree nodes. A driver that uses them exclusively will not work on an ACPI system, even if the firmware provides the exact same properties via `_DSD`. The `fwnode` API costs nothing extra on DT-only platforms but gives you ACPI portability for free.

Here is the MAX31732 driver's alarm parsing function to see this in practice:

```c
static int max31732_parse_alarms(struct device *dev, struct max31732_data *data)
{
    u32 alarm_que;

    /* Boolean property: present in DT node or _DSD package */
    if (fwnode_property_read_bool(dev_fwnode(dev), "adi,alarm1-interrupt-mode"))
        regmap_clear_bits(data->regmap, MAX31732_REG_CONF1, MAX3173X_ALARM_MODE);
    else
        regmap_set_bits(data->regmap, MAX31732_REG_CONF1, MAX3173X_ALARM_MODE);

    /* Integer property with default value */
    alarm_que = MAX31732_ALARM_FAULT_QUE;
    fwnode_property_read_u32(dev_fwnode(dev), "adi,alarm1-fault-queue", &alarm_que);
    /* ... configure hardware ... */
}
```

Not a single `#ifdef` or platform check. The same binary runs on an ARM board with a device tree and an x86 server with ACPI firmware.

---

## Putting it all together

Here is the full picture, both paths side by side, from source file to a running driver:

| Stage | Device Tree | ACPI |
|---|---|---|
| Source language | DTS / DTSI | ASL |
| Compiler | `dtc` | `iasl` |
| Binary format | `.dtb` (flat blob, magic `0xd00dfeed`) | `.aml` (AML bytecode) |
| Where binary lives | Separate file loaded by bootloader | Embedded in firmware ROM |
| Passed to kernel via | CPU register (`x0` on ARM64) | RSDP pointer in EFI system table |
| Kernel parses into | `struct device_node` tree | ACPICA namespace (`\_SB.PCI0...`) |
| Device matching key | `compatible = "adi,amax31732"` | `_HID = "AMAX3173"` |
| Resources (addr, IRQ) | `reg`, `interrupts` properties | `_CRS` resource template |
| Custom properties | Named properties in node | `_DSD` UUID-keyed packages |
| Runtime methods | None (static data only) | `_PSx`, `_DSM`, `_STA` (executable AML) |
| Runtime modification | DT overlays via configfs | SSDT loading, `_DSM` calls |
| Driver API | `of_property_read_*()` (DT only) | `acpi_evaluate_*()` (ACPI only) |
| Unified driver API | `fwnode_property_*()` (both) | `fwnode_property_*()` (both) |
| Decompile command | `dtc -I dtb -O dts board.dtb` | `iasl -d dsdt.aml` |
| Live inspection | `/proc/device-tree/` | `/sys/firmware/acpi/tables/` |
| Typical platforms | ARM, RISC-V, embedded | x86, ARM64 servers |

Both paths converge at `probe()`. By the time a driver's probe function is called, the kernel has already matched the device, allocated a `struct device`, and wired up the `fwnode` handle. The driver reads its configuration through `fwnode_property_*()` and it does not need to know or care which firmware described the hardware.

---

> This blog post is inspired by working on the MAX31732 temperature sensor driver and reading the [kernel-internals.org device tree documentation](https://kernel-internals.org/drivers/device-tree/). Further reading: `Documentation/devicetree/bindings/` and `Documentation/firmware-guide/acpi/` in the kernel source tree.
