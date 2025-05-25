---
sidebar_position: 1
---

# Installation on Alpine Linux

This guide will help you install Mini Gateway on Alpine Linux.

## Prerequisites

- Alpine Linux 3.16 or later
- Root or sudo access
- Internet connection

## Installation Steps

1. Update the package index:
```bash
apk update
```

2. Install required dependencies:
```bash
apk add curl wget bash
```

3. Download the Mini Gateway package:
```bash
wget https://github.com/your-repo/mini-gateway/releases/latest/download/mini-gateway-alpine.tar.gz
```

4. Extract the package:
```bash
tar xzf mini-gateway-alpine.tar.gz
```

5. Run the installation script:
```bash
cd mini-gateway
./install.sh
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