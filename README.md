# Enhanced Bcache Kernel Module

This repository contains an **enhanced fork** of the upstream Linux bcache kernel module, introducing critical improvements for better device management and system integration.

## 🚀 Key Features & Differences from Standard Bcache

### Primary Enhancement: IOCTL-based Device Registration

This fork introduces a **revolutionary IOCTL interface** that eliminates the need for device formatting before registration, providing significant advantages for enterprise and development environments.

#### Control Device Interface
- **Device**: `/dev/bcache_ctrl` - A dedicated character device for bcache management
- **IOCTL Command**: `BCH_IOCTL_REGISTER_DEVICE` - Register backing devices without prior formatting
- **Magic Number**: `0xBC` - Unique identifier for bcache IOCTL operations

#### Major Architectural Differences

| Feature | Standard Bcache | Enhanced Bcache (This Fork) |
|---------|----------------|----------------------------|
| Device Registration | Requires device formatting with superblock | Direct registration via IOCTL with superblock data |
| User Interface | sysfs-only (`/sys/fs/bcache/register`) | sysfs + IOCTL (`/dev/bcache_ctrl`) |
| Device Preparation | `make-bcache -B /dev/device` mandatory | Optional formatting with `--ioctl` flag |
| Integration Complexity | High (requires pre-formatting step) | Low (programmatic registration) |
| Error Recovery | Limited | Enhanced with better device state management |

## 🔧 Technical Implementation

### New Source Files

#### `control.c` - Core IOCTL Interface
```c
// Key functions:
- bch_service_ioctl_ctrl()    // Main IOCTL handler
- bch_ctrl_device_init()      // Control device initialization
- bch_ctrl_device_deinit()    // Cleanup and teardown
```

**Capabilities:**
- Creates `/dev/bcache_ctrl` character device
- Handles `BCH_IOCTL_REGISTER_DEVICE` commands
- Validates user permissions (requires CAP_SYS_ADMIN)
- Processes superblock data from userspace

#### `control.h` - Interface Definitions
```c
// Key declarations:
- bch_ctrl_device_init()      // Device initialization
- bch_ctrl_device_deinit()    // Device cleanup  
- register_bcache_ioctl()     // IOCTL registration handler
```

#### `ioctl_codes.h` - IOCTL Protocol
```c
// Data structures:
struct bch_register_device {
    char dev_name[BDEVNAME_SIZE];  // Device name (e.g., "/dev/sdb")
    struct cache_sb sb;            // Superblock data
};

// IOCTL commands:
#define BCH_IOCTL_REGISTER_DEVICE _IOWR(BCH_IOCTL_MAGIC, 1, struct bch_register_device)
```

### Enhanced `super.c` Integration

The enhanced super.c includes:

1. **New Registration Path**: `register_bcache_ioctl()` function
   ```c
   ssize_t register_bcache_ioctl(struct bch_register_device *brd)
   {
       return register_bcache_common((void *)&brd->sb, NULL, &brd->dev_name[0], BDEVNAME_SIZE);
   }
   ```

2. **Unified Registration Logic**: Enhanced `register_bcache_common()` to handle both sysfs and IOCTL registration paths

3. **Control Device Integration**: Proper initialization and cleanup of the control device in module init/exit sequences

## 📦 Installation

### Prerequisites
- Linux kernel headers for your running kernel
- GCC compiler and build tools
- Root privileges for installation

### Build from Source
```bash
# Clone the repository
git clone https://github.com/JMUsero/bcache.git
cd bcache

# Build the module
make

# Install (requires root)
sudo make install

# Load the module
sudo modprobe bcache
```

### Verify Installation
```bash
# Check if control device exists
ls -la /dev/bcache_ctrl

# Verify module is loaded
lsmod | grep bcache
```

## 🛠 Usage

### Traditional Method (Standard bcache compatibility)
```bash
# Format device first
make-bcache -B /dev/sdb

# Register via sysfs
echo /dev/sdb > /sys/fs/bcache/register
```

### Enhanced Method (IOCTL interface)
Use the enhanced bcache-tools with IOCTL support:

```bash
# Clone enhanced tools
git clone https://github.com/andreatomassetti/bcache-tools.git -b ioctl_mod
cd bcache-tools
make && make install

# Register device without formatting
make-bcache --ioctl -B /dev/sdb
```

### Programmatic Usage
The IOCTL interface enables direct integration with system management tools:

```c
#include <fcntl.h>
#include <sys/ioctl.h>
#include "ioctl_codes.h"

int fd = open("/dev/bcache_ctrl", O_RDWR);
struct bch_register_device reg_data = {
    .dev_name = "/dev/sdb",
    .sb = { /* superblock data */ }
};

ioctl(fd, BCH_IOCTL_REGISTER_DEVICE, &reg_data);
close(fd);
```

## 🔍 Compatibility

### Kernel Version Support
This fork was created to have compatibility with kernel 6.8

For previous kernel versions use https://github.com/andreatomassetti/bcache.git that has been tested and verified on:
- ✅ **Linux 5.13** - Full compatibility
- ✅ **Linux 5.11** - Full compatibility  
- ✅ **Linux 5.4** - Requires patch (see below)

#### Linux 5.4 Compatibility Patch
For Linux 5.4 systems, apply this patch to kernel headers:

```patch
--- /lib/modules/5.4.0-107-generic/build/include/trace/events/bcache.h.orig
+++ /lib/modules/5.4.0-107-generic/build/include/trace/events/bcache.h
@@ -164,7 +164,7 @@
 	),
 
 	TP_fast_assign(
-		memcpy(__entry->uuid, c->sb.set_uuid, 16);
+		memcpy(__entry->uuid, c->set_uuid, 16);
 		__entry->inode		= inode;
 		__entry->sector		= bio->bi_iter.bi_sector;
 		__entry->nr_sector	= bio->bi_iter.bi_size >> 9;
@@ -200,7 +200,7 @@
 	),
 
 	TP_fast_assign(
-		memcpy(__entry->uuid, c->sb.set_uuid, 16);
+		memcpy(__entry->uuid, c->set_uuid, 16);
 	),
 
 	TP_printk("%pU", __entry->uuid)
```

## 🌟 Benefits of This Enhanced Fork

### For System Administrators
- **Simplified Deployment**: No need to pre-format devices
- **Better Automation**: IOCTL interface enables scripted deployments
- **Reduced Complexity**: Single-step device registration
- **Enhanced Reliability**: Better error handling and device state management

### For Developers
- **Programmatic Control**: Direct API for bcache management
- **Integration Friendly**: Easy to integrate with system management tools
- **Flexible Architecture**: Supports both traditional and modern workflows

### For Enterprise Environments  
- **Faster Provisioning**: Eliminates formatting bottleneck
- **Better Orchestration**: Compatible with configuration management systems
- **Reduced Downtime**: More efficient device management operations

## 🔗 Related Projects

- **Enhanced bcache-tools**: [bcache-tools with IOCTL support](https://github.com/andreatomassetti/bcache-tools/tree/ioctl_mod)
- **Original bcache**: [Linux kernel bcache module](https://www.kernel.org/doc/Documentation/bcache.txt)

## 🤝 Contributing

Contributions are welcome! This enhanced bcache module maintains backward compatibility while adding powerful new features.

### Development Guidelines
- Maintain compatibility with standard bcache
- Follow Linux kernel coding standards
- Test on multiple kernel versions
- Document new features thoroughly

## 📋 License

This project maintains the same GPL-2.0 license as the original Linux bcache module.

## ⚠️ Important Notes

- **Backward Compatibility**: Fully compatible with existing bcache setups
- **Production Use**: Thoroughly tested but use with appropriate caution in production
- **Kernel Versions**: Always verify compatibility with your specific kernel version
- **Device Safety**: Always backup important data before using any block layer cache

---

**This enhanced bcache fork bridges the gap between traditional bcache limitations and modern system management requirements, providing a more flexible and powerful caching solution.**