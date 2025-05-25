---
sidebar_position: 4
---

# CLI Commands

This guide covers the command-line interface (CLI) commands for Mini Gateway.

## Basic Commands

### System Commands

```bash
# Check version
mini-gateway version

# Start service
mini-gateway start

# Stop service
mini-gateway stop

# Restart service
mini-gateway restart

# Check status
mini-gateway status
```

### Configuration

```bash
# Show current configuration
mini-gateway config show

# Edit configuration
mini-gateway config edit

# Validate configuration
mini-gateway config validate
```

## User Management

```bash
# Create user
mini-gateway user create --username john --password secret --role operator

# List users
mini-gateway user list

# Delete user
mini-gateway user delete --username john

# Update user
mini-gateway user update --username john --role admin
```

## Proxy Management

```bash
# Create proxy
mini-gateway proxy create --name my-proxy --type http --host 192.168.1.100 --port 8080

# List proxies
mini-gateway proxy list

# Delete proxy
mini-gateway proxy delete --name my-proxy

# Update proxy
mini-gateway proxy update --name my-proxy --port 8081
```

## Gateway Management

```bash
# Show system info
mini-gateway system info

# Show performance metrics
mini-gateway system metrics

# Backup configuration
mini-gateway backup create --name daily-backup

# Restore configuration
mini-gateway backup restore --name daily-backup
```

## Monitoring

```bash
# Show logs
mini-gateway logs show

# Follow logs
mini-gateway logs follow

# Show specific log level
mini-gateway logs show --level error

# Clear logs
mini-gateway logs clear
```

## Security

```bash
# Generate SSL certificate
mini-gateway ssl generate --domain example.com

# Update SSL certificate
mini-gateway ssl update --domain example.com

# Show security status
mini-gateway security status
```

## Help and Documentation

```bash
# Show help
mini-gateway --help

# Show command help
mini-gateway <command> --help

# Show version
mini-gateway --version
```

## Best Practices

- Use `--help` for command details
- Check command output for errors
- Use appropriate permissions
- Document custom commands
- Regular command review
- Test commands in staging
``` 