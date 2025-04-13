---
sidebar_position: 4
---

# Proxy

The Proxy component of Router Core provides TCP/TLS traffic forwarding capabilities with advanced routing based on connection characteristics. This document details the implementation, architecture, and behavior of the proxy system for contributors.

## Overview

The Proxy component handles low-level network connections:

- **TCP Proxying**: Forwards plain TCP connections
- **TLS Termination**: Handles TLS connections with SNI support
- **WebSocket Support**: Correctly handles WebSocket protocol connections
- **Connection Routing**: Routes connections based on host headers or SNI values
- **Bidirectional Communication**: Maintains duplex data flow between client and target

## Architecture

The Proxy is built on the Pingora framework's `ServerApp` trait:

```
┌─────────────────────────────────────────────────────────┐
│                       ProxyApp                           │
│                                                         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │  Connection   │   │  Protocol     │   │   Host    │  │
│  │  Handling     │   │  Detection    │   │  Routing  │  │
│  └───────┬───────┘   └───────┬───────┘   └─────┬─────┘  │
│          │                   │                 │         │
│          ▼                   ▼                 ▼         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │  Duplex Data  │   │ TLS SNI       │   │Redirect   │  │
│  │  Forwarding   │   │ Extraction    │   │Rules      │  │
│  └───────────────┘   └───────────────┘   └───────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Implementation

### Proxy Application

The main implementation is the `ProxyApp` struct:

```rust
pub struct ProxyApp {
    client_connectors: std::collections::HashMap<String, TransportConnector>,
    redirects: Vec<RedirectRule>,
}
```

- **client_connectors**: Cache of transport connectors for backend services
- **redirects**: List of redirection rules for this proxy instance

### Redirect Rules

Each redirection rule is defined by:

```rust
struct RedirectRule {
    host: Option<String>,
    alt_target: Option<BasicPeer>,
    alt_tls: bool,
}
```

- **host**: Optional hostname to match (from SNI or HTTP Host header)
- **alt_target**: Target backend to forward traffic to
- **alt_tls**: Whether this rule applies to TLS connections

### Initialization

The Proxy is initialized with source address filtering:

```rust
pub fn new(alt_source: &str) -> Self {
    // Load proxy configuration
    let node = config::RoutingData::ProxyRouting.xget::<Vec<ProxyNode>>();
    
    // Filter rules for this source address
    // Create transport connectors for all targets
    
    // Return initialized proxy
    ProxyApp {
        client_connectors,
        redirects,
    }
}
```

### Connection Processing

The `process_new` method handles incoming connections:

```rust
async fn process_new(
    self: &Arc<Self>,
    mut io: Stream,
    _shutdown: &ShutdownWatch,
) -> Option<Stream> {
    // Read initial data from client
    // Determine if it's TLS or HTTP
    // Extract hostname from SNI or Host header
    
    // Find matching redirect rule
    // Connect to target backend
    // Forward initial data
    // Set up bidirectional communication
    
    // Return None to indicate connection is fully handled
    None
}
```

### Protocol Detection

The proxy detects the protocol type by examining the first byte:

```rust
// Determine if this is a TLS connection
let is_tls = n > 0 && buf[0] == 0x16;
log::debug!(
    "Connection type : {}",
    if is_tls { "TLS" } else { "Plain HTTP" }
);

// Also detect WebSocket upgrade requests
let is_websocket = !is_tls
    && preview
        .lines()
        .any(|line| line.to_lowercase().contains("upgrade: websocket"));
```

### Host & SNI Extraction

For routing purposes, the proxy extracts host information:

```rust
// Extract host information
let host_info = if is_tls {
    // For TLS, try to extract SNI from the Client Hello
    extract_sni(&buf[0..n])
} else {
    // For plain HTTP, extract Host header
    preview.lines().find_map(|line| {
        if line.to_lowercase().starts_with("host:") {
            Some(line[5..].trim().to_string())
        } else {
            None
        }
    })
};
```

### Rule Matching

Connection routing is based on the extracted host information:

```rust
// Find matching redirect rule
let proxy_to = if let Some(host) = host_info {
    // First try exact host match
    let host_match = self.redirects
        .iter()
        .find(|rule| {
            rule.host.as_ref().map_or(false, |h| h == &host) && 
            rule.alt_tls == is_tls
        });
    
    // Fall back to catch-all rule if needed
    if host_match.is_some() {
        host_match
    } else {
        self.redirects
            .iter()
            .find(|rule| rule.host.is_none() && rule.alt_tls == is_tls)
    }
} else {
    // No host info, match on TLS status only
    self.redirects
        .iter()
        .find(|rule| rule.host.is_none() && rule.alt_tls == is_tls)
}
```

### Duplex Communication

The core of the proxy functionality is bidirectional data forwarding:

```rust
async fn duplex(&self, mut server_session: Stream, mut client_session: Stream) {
    let mut upstream_buf = [0; 8192];
    let mut downstream_buf = [0; 8192];
    let id = client_session.id();
    
    loop {
        let downstream_read = tokio::time::timeout(
            timeout_duration, 
            server_session.read(&mut upstream_buf)
        );
        let upstream_read = tokio::time::timeout(
            timeout_duration,
            client_session.read(&mut downstream_buf)
        );
        
        // Use select! to handle whichever read completes first
        select! {
            result = downstream_read => {
                // Process client to server data
            },
            result = upstream_read => {
                // Process server to client data
            },
        }
        
        // Handle read results and forward data
        // Break loop on connection close or error
    }
}
```

### TLS SNI Extraction

A key feature is extracting Server Name Indication from TLS handshakes:

```rust
fn extract_sni(buf: &[u8]) -> Option<String> {
    // Verify this is a TLS handshake
    if buf.len() < 5 || buf[0] != 0x16 {
        return None;
    }

    // Try to find SNI extension
    if let Some(pos) = find_sni_extension(buf) {
        // Extract hostname from SNI extension
        if pos + 5 < buf.len() {
            let hostname_len = ((buf[pos] as usize) << 8) | (buf[pos + 1] as usize);
            if pos + 2 + hostname_len <= buf.len() {
                if let Ok(hostname) = std::str::from_utf8(&buf[pos + 2..pos + 2 + hostname_len]) {
                    return Some(hostname.to_string());
                }
            }
        }
    }
    None
}
```

### Error Handling

The proxy implements detailed error handling with appropriate logging:

```rust
fn handle_read_error(e: std::io::Error, id: i32, is_upstream: bool) {
    let prefix = if is_upstream { "upstream" } else { "downstream" };
    let status = if is_upstream { "10" } else { "00" };
    
    if let Some(os_err) = e.raw_os_error() {
        match os_err {
            54 => log::info!("[PXY] |ID:{}, STATUS:{}, SIZE:0, COMMENT:CONNECTION_RESET |", id, status),
            60 => log::info!("[PXY] |ID:{}, STATUS:{}, SIZE:0, COMMENT:OPERATION_TIMEOUT |", id, status),
            _ => log::error!("Error reading from {}: {} (code: {:?})", prefix, e, os_err),
        }
    } else {
        log::error!("Error reading from {}: {}", prefix, e);
    }
}
```

### Timeout Management

Connections have configurable timeout protection:

```rust
// Set timeout for read operations
let timeout_duration = std::time::Duration::from_secs(120);

// Timeout handling for reads
let downstream_read = tokio::time::timeout(
    timeout_duration, 
    server_session.read(&mut upstream_buf)
);

// Timeout handler
fn handle_timeout(id: i32, is_upstream: bool) {
    let status = if is_upstream { "10" } else { "00" };
    log::info!("[PXY] |ID:{}, STATUS:{}, SIZE:0, COMMENT:READ_TIMEOUT |", id, status);
}
```

## Configuration Model

Proxy routes are configured using the `ProxyNode` struct:

```rust
pub struct ProxyNode {
    pub tls: bool,
    pub sni: Option<String>,
    pub tls_pem: Option<String>,
    pub tls_key: Option<String>,
    pub addr_listen: String,
    pub addr_target: String,
}
```

These are stored in the configuration system under the `ProxyRouting` key.

## TLS Implementation

The TLS proxy requires certificate and key files:

```rust
let tls_pem = match proxy.tls_pem {
    Some(ref pem) => pem,
    None => {
        log::error!("TLS PEM file not found for proxy: {}", proxy.addr_listen);
        continue;
    }
};
let tls_key = match proxy.tls_key {
    Some(ref key) => key,
    None => {
        log::error!("TLS Key file not found for proxy: {}", proxy.addr_listen);
        continue;
    }
};
let proxy_set = service::proxy::proxy_service_tls(
    &proxy.addr_listen,
    &tls_pem,
    &tls_key,
);
```

## Detailed Event Tracking

The proxy implements detailed connection event tracking:

```rust
// Events that can occur during communication
enum DuplexEvent {
    DownstreamRead(usize),
    UpstreamRead(usize),
}

// Event-based processing
match event {
    DuplexEvent::DownstreamRead(0) => {
        log::info!("[PXY] |ID:{}, STATUS:00, SIZE:0, COMMENT:- |", id);
        debug!("downstream session closing");
        return;
    }
    DuplexEvent::UpstreamRead(0) => {
        log::info!("[PXY] |ID:{}, STATUS:10, SIZE:0, COMMENT:- |", id);
        debug!("upstream session closing");
        return;
    }
    DuplexEvent::DownstreamRead(n) => {
        log::info!("[PXY] |ID:{}, STATUS:01, SIZE:{}, COMMENT:- |", id, n);
        // Forward data from client to server
    }
    DuplexEvent::UpstreamRead(n) => {
        log::info!("[PXY] |ID:{}, STATUS:11, SIZE:{}, COMMENT:- |", id, n);
        // Forward data from server to client
    }
}
```

## Integration with Server

The Proxy is instantiated in the server initialization for both TLS and non-TLS:

```rust
// Non-TLS Proxy server thread
{
    let handle = thread::spawn(|| {
        // Initialize server
        let mut my_server = Server::new(opt).expect("Failed to create server");
        my_server.bootstrap();
        
        // Get configuration and filter for non-TLS
        let mut node = config::RoutingData::ProxyRouting
            .xget::<Vec<ProxyNode>>()
            .unwrap_or(vec![]);
        node.retain(|x| x.tls == false);
        
        // Create proxy services
        let mut proxies: Vec<Box<dyn Service>> = vec![];
        for proxy in node {
            let proxy_set = service::proxy::proxy_service(&proxy.addr_listen);
            proxies.push(Box::new(proxy_set));
        }
        
        // Add services and run
        my_server.add_services(proxies);
        my_server.run(RunArgs::default());
    });
}
```

## Performance Optimizations

The Proxy implementation includes several optimizations:

1. **Connection reuse**: Transport connectors are cached
2. **Buffer reuse**: Fixed buffers are reused for data transfer
3. **Asynchronous I/O**: Uses non-blocking operations via Tokio
4. **Parallel processing**: Each connection is handled independently
5. **Efficient TLS parsing**: Minimal TLS header parsing for SNI extraction

## Default Fallbacks

If a connection cannot be matched to a rule:

```rust
// Default fallback options
.map(|rule| {
    rule.alt_target
        .as_ref()
        .unwrap_or(&BasicPeer::new(DEFAULT_PORT.p404))
        .clone()
})
.unwrap_or_else(|| BasicPeer::new(DEFAULT_PORT.p500));
```

Both HTTP 404 and 500 fallbacks are available, depending on the error type.

## Debug Logging

For troubleshooting, the proxy logs:

- Connection initialization
- Protocol detection results
- Host/SNI extraction
- Target selection
- Data transfer details
- Connection termination
- Error conditions
