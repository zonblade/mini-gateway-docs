---
sidebar_position: 2
---

# Proxy Management

This guide covers how to manage proxies in Mini Gateway.

## Proxy Types

Mini Gateway supports various proxy types:
- HTTP/HTTPS
- SOCKS4/SOCKS5
- Shadowsocks
- WireGuard

## Adding a New Proxy

### Using the Web Interface

1. Navigate to Proxies > Add Proxy
2. Select proxy type
3. Configure proxy settings:
   - Name
   - Protocol
   - Host
   - Port
   - Authentication (if required)
4. Click "Create Proxy"

### Using the CLI

```bash
mini-gateway proxy create --name my-proxy --type http --host 192.168.1.100 --port 8080
```

## Proxy Configuration

### Basic Settings

- **Name**: Unique identifier
- **Protocol**: Proxy protocol
- **Host**: Server address
- **Port**: Listening port
- **Authentication**: Username/password if required

### Advanced Settings

- **Load Balancing**: Configure multiple upstream servers
- **Health Checks**: Set up monitoring
- **Rate Limiting**: Control traffic flow
- **SSL/TLS**: Configure encryption

## Proxy Groups

1. Create a proxy group:
   - Set group name
   - Add proxies to group
   - Configure load balancing strategy

2. Common strategies:
   - Round-robin
   - Least connections
   - IP hash
   - Weighted distribution

## Monitoring

### Health Checks

1. Configure health check parameters:
   - Interval
   - Timeout
   - Success threshold
   - Failure threshold

2. View health status:
   - Web interface dashboard
   - CLI status command
   - API endpoints

### Performance Metrics

Monitor key metrics:
- Response time
- Connection count
- Error rate
- Bandwidth usage

## Best Practices

- Regular health monitoring
- Implement failover strategies
- Use appropriate proxy types
- Secure authentication
- Document configurations
- Regular maintenance 