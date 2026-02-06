# 4.1 Embedded Linux Basics

## Topics Covered

### Kernel Space vs User Space

#### Architecture
```
+------------------+
|   Applications   |  User Space (unprivileged)
+------------------+
|   Libraries      |  (libc, pthread, etc.)
+------------------+
| System Call API  |  Interface
+------------------+
|   Kernel         |  Kernel Space (privileged)
| Drivers, MM, FS  |
+------------------+
|   Hardware       |
+------------------+
```

#### Kernel Space
- **Privileged Mode**: CPU runs in privileged mode
- **Direct Hardware Access**: Can access all memory and I/O
- **No Memory Protection**: Bugs can crash the system
- **Shared Address Space**: All kernel code shares memory

- **Components**:
  - Process scheduler
  - Memory management
  - Device drivers
  - Filesystem layer
  - Network stack

#### User Space
- **Unprivileged Mode**: CPU runs in user mode
- **Protected Memory**: Each process has isolated memory
- **System Calls**: Request kernel services via syscalls
- **Preemptible**: Can be interrupted and context-switched

- **Communication with Kernel**:
  - System calls (read, write, open, ioctl, etc.)
  - Device files (/dev/*)
  - sysfs (/sys/*)
  - procfs (/proc/*)

#### Context Switching
- User space → Kernel space: System call, interrupt, exception
- Kernel space → User space: System call return, interrupt return
- Overhead: Microseconds (context save/restore, cache flush)

### Boot Process

#### Boot Sequence
```
Power-On → ROM Code → Bootloader → Kernel → Init → User Space
```

#### 1. ROM Code (First Stage)
- **Location**: On-chip ROM (burned by manufacturer)
- **Tasks**:
  - Basic CPU initialization
  - Clock setup
  - Load bootloader from boot device (SD, eMMC, NOR/NAND Flash)
  - Verify bootloader (if secure boot enabled)
- **Examples**: AM335x ROM, i.MX ROM

#### 2. U-Boot (Second Stage Bootloader)
- **Universal Bootloader**: Most common for embedded Linux
- **Tasks**:
  - Initialize DRAM
  - Setup peripherals
  - Load kernel and device tree
  - Pass boot arguments
  - Jump to kernel

- **U-Boot Commands**:
  ```bash
  printenv            # Show environment variables
  setenv bootargs ... # Set kernel command line
  saveenv             # Save environment
  fatload mmc 0:1 0x80000000 zImage  # Load kernel
  bootz 0x80000000 - 0x82000000      # Boot kernel
  ```

- **Boot Script** (boot.scr):
  ```bash
  setenv bootargs console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait
  fatload mmc 0:1 ${kernel_addr} zImage
  fatload mmc 0:1 ${fdt_addr} am335x-boneblack.dtb
  bootz ${kernel_addr} - ${fdt_addr}
  ```

#### 3. Linux Kernel
- **Decompression**: Kernel image decompressed (zImage, bzImage)
- **Architecture Setup**: CPU-specific initialization
- **Memory Management**: Setup page tables, MMU
- **Driver Initialization**: Built-in drivers probe and initialize
- **Mount Root Filesystem**: From boot arguments (root=)
- **Execute Init**: First user-space process (PID 1)

#### 4. Init System
- **Traditional init**: SysV init scripts (/etc/init.d/)
- **Systemd**: Modern init system with dependencies
- **BusyBox init**: Lightweight for embedded
- **Tasks**:
  - Mount filesystems
  - Start services
  - Setup networking
  - Launch applications

### Device Tree (DTS/DTB)

#### Purpose
- **Hardware Description**: Describe hardware to kernel
- **Board Configuration**: Without recompiling kernel
- **Platform Data**: Pass configuration to drivers

#### Device Tree Source (.dts)
```dts
/dts-v1/;

/ {
    compatible = "ti,am335x-bone-black", "ti,am33xx";
    
    cpus {
        cpu@0 {
            compatible = "arm,cortex-a8";
            device_type = "cpu";
            clock-frequency = <1000000000>; // 1 GHz
        };
    };
    
    memory@80000000 {
        device_type = "memory";
        reg = <0x80000000 0x20000000>; // 512MB at 0x80000000
    };
    
    leds {
        compatible = "gpio-leds";
        
        led0 {
            label = "beaglebone:green:usr0";
            gpios = <&gpio1 21 0>;  // GPIO1_21
            default-state = "off";
        };
    };
    
    i2c0: i2c@44e0b000 {
        compatible = "ti,omap4-i2c";
        reg = <0x44e0b000 0x1000>;
        interrupts = <70>;
        clock-frequency = <400000>; // 400 kHz
        
        eeprom@50 {
            compatible = "atmel,24c256";
            reg = <0x50>;
            pagesize = <64>;
        };
    };
};
```

#### Key Concepts
- **Nodes**: Represent devices or buses
- **Properties**: Key-value pairs (compatible, reg, interrupts)
- **phandle**: Reference to other nodes
- **Overlay**: Modify existing device tree at runtime

#### Device Tree Blob (.dtb)
- **Compiled Form**: Binary representation
- **Compilation**: `dtc -I dts -O dtb -o output.dtb input.dts`
- **Loaded by Bootloader**: Passed to kernel

#### Runtime Device Tree
- **Location**: /sys/firmware/devicetree/
- **Overlays**: /sys/kernel/config/device-tree/overlays/

### Kernel Modules & Drivers

#### Kernel Modules
- **Dynamic Loading**: Load/unload without reboot
- **Extension**: Add functionality to running kernel

- **Module Operations**:
  ```bash
  insmod my_module.ko     # Insert module
  rmmod my_module         # Remove module
  lsmod                   # List loaded modules
  modprobe my_module      # Load with dependencies
  modinfo my_module.ko    # Show module info
  ```

- **Simple Module Example**:
  ```c
  #include <linux/module.h>
  #include <linux/kernel.h>
  #include <linux/init.h>
  
  static int __init hello_init(void) {
      printk(KERN_INFO "Hello, kernel!\n");
      return 0;
  }
  
  static void __exit hello_exit(void) {
      printk(KERN_INFO "Goodbye, kernel!\n");
  }
  
  module_init(hello_init);
  module_exit(hello_exit);
  
  MODULE_LICENSE("GPL");
  MODULE_AUTHOR("Your Name");
  MODULE_DESCRIPTION("A simple hello world module");
  ```

- **Makefile**:
  ```makefile
  obj-m += hello.o
  
  all:
      make -C /lib/modules/$(shell uname -r)/build M=$(PWD) modules
  
  clean:
      make -C /lib/modules/$(shell uname -r)/build M=$(PWD) clean
  ```

#### Device Drivers

##### Character Devices
- **Stream-Oriented**: Byte streams (serial ports, keyboards)
- **Major/Minor Numbers**: Identify device

```c
#include <linux/fs.h>
#include <linux/cdev.h>

static dev_t dev_num;
static struct cdev my_cdev;

static int my_open(struct inode *inode, struct file *file) {
    printk(KERN_INFO "Device opened\n");
    return 0;
}

static ssize_t my_read(struct file *file, char __user *buf, 
                       size_t count, loff_t *pos) {
    // Copy data to user space
    return 0;
}

static struct file_operations fops = {
    .owner = THIS_MODULE,
    .open = my_open,
    .read = my_read,
    // .write, .release, .ioctl, etc.
};

static int __init my_driver_init(void) {
    // Allocate device number
    alloc_chrdev_region(&dev_num, 0, 1, "my_device");
    
    // Initialize and add cdev
    cdev_init(&my_cdev, &fops);
    cdev_add(&my_cdev, dev_num, 1);
    
    return 0;
}
```

##### Block Devices
- **Block-Oriented**: Fixed-size blocks (hard drives, flash)
- **Buffer Cache**: Kernel caches blocks

##### Network Devices
- **Packet-Oriented**: Network interfaces (Ethernet, Wi-Fi)
- **No Device Files**: Accessed via sockets

#### Platform Drivers
- **Platform Devices**: Non-discoverable devices (defined in device tree)
- **Probing**: Match by compatible string

```c
#include <linux/platform_device.h>
#include <linux/of.h>

static int my_probe(struct platform_device *pdev) {
    struct device_node *node = pdev->dev.of_node;
    struct resource *res;
    void __iomem *base;
    
    // Get memory resource from device tree
    res = platform_get_resource(pdev, IORESOURCE_MEM, 0);
    base = devm_ioremap_resource(&pdev->dev, res);
    
    // Initialize hardware
    
    return 0;
}

static int my_remove(struct platform_device *pdev) {
    // Cleanup
    return 0;
}

static const struct of_device_id my_of_match[] = {
    { .compatible = "vendor,my-device", },
    { },
};
MODULE_DEVICE_TABLE(of, my_of_match);

static struct platform_driver my_driver = {
    .probe = my_probe,
    .remove = my_remove,
    .driver = {
        .name = "my-driver",
        .of_match_table = my_of_match,
    },
};

module_platform_driver(my_driver);
```

## Key Concepts

- **printk**: Kernel logging (not printf)
- **__user**: Pointer to user-space memory
- **copy_to_user/copy_from_user**: Safe user-space access
- **ioremap**: Map physical to virtual address
- **Concurrency**: Spinlocks, mutexes, RCU

## Practical Exercises

1. Write a "Hello World" kernel module
2. Create a character device driver
3. Parse device tree in a platform driver
4. Customize U-Boot boot script
5. Analyze kernel boot messages (dmesg)
6. Create a device tree overlay
7. Debug a kernel driver with printk

## Kernel Configuration

```bash
# Configure kernel
make menuconfig

# Build kernel
make -j$(nproc) zImage

# Build device tree
make dtbs

# Build modules
make modules

# Install modules (to target)
make INSTALL_MOD_PATH=/mnt/rootfs modules_install
```

## Resources

- Linux Device Drivers (LDD3) book
- Kernel documentation (Documentation/)
- Device tree specification
- U-Boot documentation
