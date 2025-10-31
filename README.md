# STF Device Database

This repository contains device metadata for [DeviceFarmer/STF](https://github.com/DeviceFarmer/stf) (Smartphone Test Farm), providing custom device information, images, and specifications that appear in the STF web interface.

## What's Inside

- **devices-latest.json** - Device specifications database containing:
  - CPU information (cores, frequency, chipset name)
  - Display specifications (resolution, screen size)
  - Memory information (RAM, storage)
  - OS version and release dates
  - Manufacturer and carrier information
  - Custom display names

- **icon/** - Device icon images in multiple sizes:
  - `x24/` - 24px height thumbnails
  - `x120/` - 120px height icons

- **photo/** - Full-size device images:
  - `x800/` - High-resolution device photos

## Using This Database

This device database is designed to be used as a Git submodule mounted into STF containers to override the default device information.

### Integration with STF

Mount this directory into your STF containers at the device database location:

```yaml
volumes:
  - /path/to/stf-devices:/app/node_modules/@devicefarmer/stf-device-db/dist
```

### Adding New Devices

When you connect a new device to your device farm, follow these steps:

#### 1. Gather Device Information via ADB

Connect to the device and retrieve its properties:
```bash
adb shell getprop ro.product.model          # Critical: This becomes the JSON key
adb shell getprop ro.product.manufacturer
adb shell getprop ro.build.version.release
adb shell getprop ro.boot.hardware.sku      # Optional, for "long" field
```

Check memory:
```bash
adb shell cat /proc/meminfo | grep MemTotal
adb shell df -h /data | tail -n 1 | awk '{print $2}'
```

#### 2. Fetch Device Specifications

Look up the device on [GSMArena](https://www.gsmarena.com) or the manufacturer's website to get:
- CPU name, cores, and max frequency
- Display resolution (height x width) and screen size in inches
- Official release date
- Verify RAM/storage matches what ADB reported

#### 3. Create JSON Entry

Add an entry to `devices-latest.json`:

**Critical**: The JSON key must match `ro.product.model` EXACTLY:
- Convert spaces to underscores
- Preserve special characters (parentheses, hyphens, etc.)
- Watch for leading/trailing spaces (convert to underscores)

Example: `" edge plus 5G UW (2022)"` becomes `_edge_plus_5G_UW_(2022)`

Insert alphabetically within the manufacturer's group.

#### 4. Add Device Images

Create three image sizes using ImageMagick:

```bash
# Copy original to x800 (no resize)
cp source.jpg photo/x800/Device_Model_Key.jpg

# Create 120px height version
magick source.jpg -resize x120 icon/x120/Device_Model_Key.jpg

# Create 24px height version
magick source.jpg -resize x24 icon/x24/Device_Model_Key.jpg
```

All filenames must match the JSON key exactly.

#### 5. Commit and Deploy

```bash
git add devices-latest.json icon/ photo/
git commit -m "Add [Device Name]"
git push
```

Then update your STF deployment to pull the new submodule commit.

## Device Entry Format

```json
"Device_Model_Key": {
  "cpu": {
    "cores": 8,
    "freq": 2.84,
    "name": "Chipset Name"
  },
  "date": "2019-04-11T15:00:00.000Z",
  "display": {
    "h": 3120,
    "s": 6.1,
    "w": 1440
  },
  "maker": {
    "code": "xx",
    "name": "Manufacturer"
  },
  "memory": {
    "ram": 6144,
    "rom": 131072
  },
  "name": {
    "id": "Display Name",
    "long": "Hardware Model"
  },
  "os": {
    "type": "android",
    "ver": "12.0"
  }
}
```

**Important Notes**:
- RAM and ROM are in MB (e.g., 6GB RAM = 6144 MB, 128GB ROM = 131072 MB)
- The "id" field is the user-friendly display name (can be customized)
- The "long" field typically contains the hardware SKU or model number
- The JSON key must match the device's exact model string

## Image Requirements

### Icon Sizes
- **x24**: Exactly 24px height, proportional width
- **x120**: Exactly 120px height, proportional width

### Photo Size
- **x800**: Original resolution (no specific size requirement)

All images should be in JPEG format and named to match the device's JSON key.

## Production Deployment Guide

Looking to set up your own distributed STF device farm with Docker Compose?

**📖 See the [Distributed Setup Guide](DISTRIBUTED_SETUP_GUIDE.md)** for a complete, production-ready deployment including:
- Central server configuration
- Multiple remote machine setup
- SSL/TLS configuration
- Device metadata management
- Troubleshooting and monitoring

This guide bridges the gap between the official STF documentation (which focuses on systemd) and practical Docker Compose deployments for distributed environments.

## Current Devices

This database currently contains metadata for 60+ devices including:
- Samsung Galaxy series (S6-S25, Note, Tab)
- Google Pixel series (1-10)
- OnePlus devices (3, 6T, 7T, 9, 12R)
- LG devices (G4-G8 ThinQ, V series)
- Motorola devices (Edge+, Moto series)
- Legacy devices (BlackBerry, iPhone 3GS-6S, Palm)
- And more...

## Contributing

If you'd like to contribute device metadata:

1. Fork this repository
2. Add your device information following the format above
3. Include properly sized images
4. Submit a pull request with clear device information

Please ensure your submissions:
- Use accurate specifications from official sources
- Include all three image sizes
- Follow the existing naming conventions
- Don't include any proprietary or copyrighted images

## Related Projects

- [DeviceFarmer/stf](https://github.com/DeviceFarmer/stf) - The main STF project
- [STF Device Database](https://github.com/devicefarmer/stf-device-db) - Official device database

## License

This device database is provided as-is for use with DeviceFarmer/STF deployments. Device specifications are compiled from publicly available information. Device images should be considered fair use for the purpose of device identification in testing infrastructure.

---

*This database is actively maintained and updated as new devices are added to our device farm.*
