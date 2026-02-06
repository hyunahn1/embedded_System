# 4.2 Yocto Project (Build System)

## Topics Covered

### Yocto Project Concepts

#### Overview
- **Purpose**: Build custom Linux distributions for embedded devices
- **Layer-Based**: Modular, maintainable architecture
- **Cross-Platform**: Supports multiple architectures
- **Reproducible**: Bit-identical builds

#### Core Components
- **Poky**: Reference distribution (Yocto + OE-Core)
- **BitBake**: Build engine (similar to make)
- **OpenEmbedded-Core (OE-Core)**: Core metadata

#### Architecture
```
┌─────────────────────────────────┐
│   User Configuration            │
│   (local.conf, bblayers.conf)   │
└─────────────────────────────────┘
            ↓
┌─────────────────────────────────┐
│   BitBake Build Engine          │
│   (Parse, Fetch, Build, Package)│
└─────────────────────────────────┘
            ↓
┌─────────────────────────────────┐
│   Layers (Metadata)             │
│   - Recipes (.bb)               │
│   - Classes (.bbclass)          │
│   - Configuration (.conf)       │
└─────────────────────────────────┘
            ↓
┌─────────────────────────────────┐
│   Output: Root Filesystem       │
│   Kernel, Bootloader, Packages  │
└─────────────────────────────────┘
```

#### Workflow
```bash
# 1. Setup build environment
source oe-init-build-env

# 2. Configure build (conf/local.conf)
MACHINE = "beaglebone-yocto"
DISTRO = "poky"

# 3. Build image
bitbake core-image-minimal

# 4. Flash to target
dd if=tmp/deploy/images/.../image.wic of=/dev/sdX
```

### Metadata: Recipes, Classes, Configuration

#### Recipes (.bb files)
- **Purpose**: Describe how to build a package
- **Components**:
  - Source location (SRC_URI)
  - Dependencies (DEPENDS, RDEPENDS)
  - Build instructions (do_compile, do_install)
  - Licensing (LICENSE)

- **Example Recipe** (hello_1.0.bb):
  ```python
  DESCRIPTION = "Simple hello world application"
  LICENSE = "MIT"
  LIC_FILES_CHKSUM = "file://LICENSE;md5=..."
  
  SRC_URI = "file://hello.c \
             file://LICENSE"
  
  S = "${WORKDIR}"
  
  do_compile() {
      ${CC} ${CFLAGS} ${LDFLAGS} hello.c -o hello
  }
  
  do_install() {
      install -d ${D}${bindir}
      install -m 0755 hello ${D}${bindir}
  }
  ```

- **Recipe Variables**:
  - **${PN}**: Package name (e.g., "hello")
  - **${PV}**: Package version (e.g., "1.0")
  - **${S}**: Source directory
  - **${D}**: Destination directory (staging)
  - **${WORKDIR}**: Working directory

#### Recipe Tasks
```python
do_fetch       # Download sources
do_unpack      # Extract archives
do_patch       # Apply patches
do_configure   # Run ./configure
do_compile     # Build (make)
do_install     # Install to ${D}
do_package     # Create packages (.deb, .rpm, .ipk)
```

#### Append Files (.bbappend)
- **Purpose**: Modify existing recipes without editing original
- **Location**: Any layer, matched by recipe name

```python
# In meta-custom/recipes-core/base-files/base-files_%.bbappend
FILESEXTRAPATHS:prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://custom-issue"

do_install:append() {
    install -m 0644 ${WORKDIR}/custom-issue ${D}${sysconfdir}/issue
}
```

#### Classes (.bbclass)
- **Purpose**: Reusable build logic
- **Inheritance**: `inherit <classname>`

- **Common Classes**:
  - **autotools**: GNU autotools (./configure && make)
  - **cmake**: CMake projects
  - **kernel**: Kernel building
  - **systemd**: Systemd service integration

```python
# In recipe
inherit autotools

# Equivalent to:
do_configure() {
    ./configure --prefix=${prefix}
}

do_compile() {
    make
}

do_install() {
    make install DESTDIR=${D}
}
```

#### Configuration Files (.conf)

##### local.conf (Build Configuration)
```python
# Machine selection
MACHINE = "raspberrypi4"

# Parallel build jobs
BB_NUMBER_THREADS = "8"
PARALLEL_MAKE = "-j 8"

# Add features
EXTRA_IMAGE_FEATURES = "debug-tweaks ssh-server-openssh"

# Disk space monitoring
BB_DISKMON_DIRS = "..."
```

##### bblayers.conf (Layer Configuration)
```python
BBLAYERS ?= " \
  /path/to/poky/meta \
  /path/to/poky/meta-poky \
  /path/to/poky/meta-yocto-bsp \
  /path/to/meta-openembedded/meta-oe \
  /path/to/meta-custom \
  "
```

##### Machine Configuration (meta-custom/conf/machine/mymachine.conf)
```python
require conf/machine/include/arm/armv7a/tune-cortexa8.inc

PREFERRED_PROVIDER_virtual/kernel = "linux-custom"
KERNEL_DEVICETREE = "am335x-boneblack.dtb"

MACHINE_FEATURES = "usbhost usbgadget wifi bluetooth"

IMAGE_FSTYPES = "tar.bz2 ext4 wic"

SERIAL_CONSOLES = "115200;ttyS0"
```

### Layers (Creating Custom Layers)

#### Layer Structure
```
meta-custom/
├── conf/
│   ├── layer.conf
│   └── machine/
│       └── mymachine.conf
├── recipes-kernel/
│   └── linux/
│       ├── linux-custom_5.10.bb
│       └── linux-custom/
│           ├── defconfig
│           └── 0001-custom-patch.patch
├── recipes-core/
│   └── images/
│       └── custom-image.bb
├── recipes-app/
│   └── myapp/
│       ├── myapp_1.0.bb
│       └── myapp/
│           ├── myapp.c
│           └── LICENSE
└── README

```

#### layer.conf
```python
# Layer identification
BBPATH .= ":${LAYERDIR}"

BBFILES += "${LAYERDIR}/recipes-*/*/*.bb \
            ${LAYERDIR}/recipes-*/*/*.bbappend"

BBFILE_COLLECTIONS += "meta-custom"
BBFILE_PATTERN_meta-custom = "^${LAYERDIR}/"
BBFILE_PRIORITY_meta-custom = "6"

LAYERDEPENDS_meta-custom = "core"
LAYERSERIES_COMPAT_meta-custom = "kirkstone"
```

#### Creating a Layer
```bash
# Use bitbake-layers tool
bitbake-layers create-layer meta-custom

# Add layer to build
bitbake-layers add-layer /path/to/meta-custom

# Show layers
bitbake-layers show-layers
```

### Images (Building Custom Images)

#### Image Recipes
```python
# custom-image.bb
DESCRIPTION = "Custom embedded Linux image"

IMAGE_INSTALL = "packagegroup-core-boot \
                 ${CORE_IMAGE_EXTRA_INSTALL} \
                 openssh \
                 nano \
                 myapp \
                 "

IMAGE_FEATURES += "ssh-server-openssh"

# Inherit core image class
inherit core-image

# Additional space (MB)
IMAGE_ROOTFS_EXTRA_SPACE = "500"
```

#### Package Groups
```python
# packagegroup-custom.bb
DESCRIPTION = "Custom package group"

inherit packagegroup

RDEPENDS:${PN} = "packagegroup-core-boot \
                  python3 \
                  i2c-tools \
                  can-utils \
                  "
```

#### Building Images
```bash
# List available images
ls meta*/recipes*/images/*.bb

# Build image
bitbake custom-image

# Build SDK (for application development)
bitbake custom-image -c populate_sdk

# Build specific package
bitbake myapp

# Clean package
bitbake -c clean myapp

# Rebuild from scratch
bitbake -c cleansstate myapp
```

### DevTool (Workspace Management)

#### Purpose
- **Modify Recipes**: Edit, build, test in workspace
- **Upstream Workflow**: Prepare patches for submission

#### Commands
```bash
# Add recipe to workspace (download source)
devtool add myapp https://github.com/user/myapp.git

# Modify source
cd workspace/sources/myapp
# ... make changes ...

# Build and test
devtool build myapp

# Deploy to target device
devtool deploy-target myapp root@192.168.1.100

# Generate patch
devtool finish myapp meta-custom

# Update existing recipe
devtool modify linux-custom
```

### BSP (Board Support Package)

#### Purpose
- **Hardware Support**: Enable specific board/SoC
- **Components**:
  - Bootloader configuration
  - Kernel configuration and device tree
  - Machine configuration
  - Drivers

#### BSP Layer Structure
```
meta-mybsp/
├── conf/
│   ├── layer.conf
│   └── machine/
│       └── myboard.conf
├── recipes-bsp/
│   └── u-boot/
│       ├── u-boot_%.bbappend
│       └── u-boot/
│           └── myboard/
│               └── 0001-add-board-support.patch
├── recipes-kernel/
│   └── linux/
│       ├── linux-custom_%.bbappend
│       └── linux-custom/
│           └── myboard/
│               ├── defconfig
│               └── myboard.dts
└── recipes-core/
    └── images/
        └── myboard-image.bb
```

#### Machine Configuration
```python
# conf/machine/myboard.conf
require conf/machine/include/soc-family.inc

PREFERRED_PROVIDER_virtual/kernel = "linux-custom"
PREFERRED_PROVIDER_virtual/bootloader = "u-boot"

KERNEL_IMAGETYPE = "zImage"
KERNEL_DEVICETREE = "myboard.dtb"

MACHINE_FEATURES = "serial usbhost usbgadget ethernet wifi"

UBOOT_MACHINE = "myboard_defconfig"

IMAGE_FSTYPES = "wic wic.bmap ext4"

WKS_FILE = "myboard-sdimage.wks"
```

## Key Concepts

- **Sysroot**: Isolated build environment per recipe
- **Shared State Cache (sstate)**: Reuse build artifacts
- **tmp/deploy/images**: Output images and packages
- **DL_DIR**: Downloaded source archives
- **TMPDIR**: Temporary build files

## Practical Exercises

1. Build core-image-minimal for a target
2. Create a custom layer with an application
3. Write a recipe for a custom application
4. Append an existing recipe
5. Build a custom image with specific packages
6. Use devtool to modify and test a recipe
7. Create a BSP for a new board

## Build System Tips

```bash
# Environment variables
echo $BUILDDIR
echo $MACHINE

# BitBake commands
bitbake -e myapp | grep ^SRC_URI=    # Show effective variable
bitbake -g custom-image              # Generate dependency graph
bitbake -c listtasks myapp           # List available tasks

# Clean builds
bitbake -c clean myapp               # Clean workdir
bitbake -c cleansstate myapp         # Clean sstate
rm -rf tmp                           # Clean everything
```

## Common Issues

- **Parse errors**: Check recipe syntax
- **Fetch failures**: Network issues, wrong SRC_URI
- **License checksum mismatch**: Update LIC_FILES_CHKSUM
- **Do_compile failed**: Check build dependencies (DEPENDS)
- **Do_install failed**: Ensure files exist before install

## References

- [Yocto Project Documentation](https://docs.yoctoproject.org/)
- [BitBake User Manual](https://docs.yoctoproject.org/bitbake/)
- [OpenEmbedded Layer Index](https://layers.openembedded.org/)
- Yocto Project Mega Manual
