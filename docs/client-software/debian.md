---
sidebar_position: 3
---

# Debian Client

This guide will help you install and configure the Mini Gateway client on Debian-based systems.

## System Requirements

- Debian 11 (Bullseye) or later
- 4GB RAM minimum
- 100MB free disk space

## Installation

1. Add the repository:
```bash
curl -fsSL https://packages.example.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/mini-gateway-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/mini-gateway-archive-keyring.gpg] https://packages.example.com/debian stable main" | sudo tee /etc/apt/sources.list.d/mini-gateway.list
```

2. Update package list:
```bash
sudo apt update
```

3. Install the client:
```bash
sudo apt install mini-gateway-client
```

## Configuration

1. Edit the configuration file:
```bash
sudo nano /etc/mini-gateway/client.conf
```

2. Add your gateway server details:
```ini
server = your-server-address
port = your-port
username = your-username
password = your-password
```

3. Start the service:
```bash
sudo systemctl start mini-gateway-client
```

4. Enable auto-start:
```bash
sudo systemctl enable mini-gateway-client
```

## Troubleshooting

Common issues and solutions:

- **Service Not Starting**: Check logs with `journalctl -u mini-gateway-client`
- **Connection Failed**: Verify server address and port
- **Authentication Error**: Check credentials in config file

## Support

For additional support:
- Check the [FAQ](../faq.md)
- Visit our [GitHub repository](https://github.com/your-repo/mini-gateway)
- Contact support at support@example.com 