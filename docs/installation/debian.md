---
sidebar_position: 2
---

# Installation on Debian

This guide will help you install Mini Gateway on Debian-based systems.

## Prerequisites

- Debian 11 (Bullseye) or later
- Root or sudo access
- Internet connection

## Installation Steps

1. Update the package index:
```bash
sudo apt update
```

2. Install required dependencies:
```bash
sudo apt install curl wget
```

3. Download the Mini Gateway package:
```bash
wget https://github.com/your-repo/mini-gateway/releases/latest/download/mini-gateway-debian.deb
```

4. Install the package:
```bash
sudo dpkg -i mini-gateway-debian.deb
```

5. Install any missing dependencies:
```bash
sudo apt install -f
```

## Verification

After installation, verify the setup:
```bash
mini-gateway --version
```

## Next Steps

- Configure your gateway settings
- Set up user management
- Install client software 