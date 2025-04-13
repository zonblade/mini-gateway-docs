---
sidebar_position: 1
---

# Architecture

The Router Core architecture follows a modular, multi-threaded design that maximizes performance, reliability, and extensibility. This document provides an in-depth explanation of its architecture, component interactions, and threading model.

## Architectural Overview

Router Core uses a layered architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────┐
│                      Main Process                        │
│                                                         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │  Configuration│   │  Protocol     │   │ Ctrl+X    │  │
│  │  Management   │   │  Interface    │   │ Handler   │  │
│  └───────┬───────┘   └───────┬───────┘   └─────┬─────┘  │
│          │                   │                 │         │
│          ▼                   ▼                 ▼         │
│  ┌─────────────────────────────────────────────────────┐│
│  │                  Server Thread Pool                  ││
│  │                                                      ││
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐ ││
│  │  │  Gateway   │  │  Proxy     │  │  TLS Proxy     │ ││
│  │  │  Services  │  │  Services  │  │  Services      │ ││
│  │  └────────────┘  └────────────┘  └────────────────┘ ││
│  │                                                      ││
│  │  ┌────────────┐  ┌────────────┐  ┌────────────────┐ ││
│  │  │  404 Error │  │  500 Error │  │  TLS Honeypot  │ ││
│  │  │  Handler   │  │  Handler   │  │  Handler       │ ││
│  │  └────────────┘  └────────────┘  └────────────────┘ ││
│  └─────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────┘
```

## Threading Model

Router Core uses a multi-threaded architecture to maximize performance:

1. **Main Thread**: Manages the application lifecycle and coordinates other threads
   - Initializes configuration
   - Starts the protocol interface
   - Sets up signal handlers
   - Monitors system state

2. **Protocol Interface Thread**: Runs the custom protocol server
   - Manages service registration
   - Handles client connections
   - Routes commands to appropriate services

3. **Server Thread Pool**: Multiple threads that handle different server components
   - Gateway Service Thread: Handles HTTP routing
   - Non-TLS Proxy Thread: Handles plain TCP connections
   - TLS Proxy Thread: Handles secure TLS connections
   - Error Handler Threads: Handle default error pages (404, 500)
   - TLS Honeypot Thread: Security monitoring for suspicious TLS connections

Each thread operates independently but shares configuration state through thread-safe mechanisms.

## Component Relationships

### Main Process

The main process (in `main.rs`) serves as the orchestrator for the entire system:

```rust
fn main() {
    // Configure file-based logging
    config::init();
    system::writer::writer_start();
    
    // Start custom protocol server
    system::protocol::start_interface();
    
    // Set up signal handling
    // ...
    
    // Main control loop
    loop {
        // Check for termination signal
        // Start server if needed
        // ...
    }
}
```

The main loop continuously monitors system state and responds to control signals.

### Server Initialization

The `system::server::init()` function initializes and runs all server components:

```rust
pub fn init() {
    // Vector to store thread handles
    let mut server_threads = Vec::new();
    
    // Create threads for each server component:
    // - Gateway services
    // - Non-TLS proxy
    // - TLS proxy
    // - Error handlers (404, 500)
    // - TLS honeypot
    
    // Wait for all threads to complete
    for handle in server_threads {
        // Join threads...
    }
}
```

Each server component runs in its own thread, with coordination handled through shared configuration.

## Data Flow

### Gateway Request Flow

1. Client sends HTTP request to gateway port
2. Gateway thread accepts connection
3. Request is matched against path patterns
4. Path is transformed according to matching rule
5. Request is forwarded to target backend
6. Response is returned to client

### Proxy Connection Flow

1. Client connects to proxy port
2. Proxy thread accepts connection
3. Initial data is analyzed (TLS, WebSocket, HTTP)
4. Connection is matched to target based on host/SNI
5. Bidirectional tunnel is established to target
6. Data flows between client and target

### Protocol Communication Flow

1. Client connects to protocol server
2. Client sends handshake with service name and parameters
3. Protocol server routes to appropriate service handler
4. Service processes the request
5. Response is sent back to client

## Configuration System

Router Core uses a dynamic configuration system based on the `mini-config` crate:

```rust
// Configuration initialization
pub fn init(){
    // Set default values
    RoutingData::ProxyID.set("-");
    RoutingData::GatewayID.set("-");
    // Initialize routing tables
    RoutingData::GatewayRouting.xset::<Vec<GatewayNode>>(vec![]);
    RoutingData::ProxyRouting.xset::<Vec<ProxyNode>>(vec![]);
}
```

Configuration is accessed through enum-based keys that provide type safety and centralized management.

## Logging Architecture

The logging system uses a tag-based approach to direct different types of logs to separate files:

```
┌───────────────┐     ┌───────────────┐    ┌───────────────┐
│ Gateway       │     │ Proxy         │    │ Protocol      │
│ Component     │─┬──▶│ Component     │───▶│ Component     │
└───────────────┘ │   └───────────────┘    └───────────────┘
                  │          │                     │
                  │          │                     │
                  │          ▼                     ▼
                  │   ┌───────────────┐    ┌───────────────┐
                  └──▶│ Logging       │◀───│ Network       │
                      │ Dispatcher    │    │ Listener      │
                      └───────┬───────┘    └───────────────┘
                              │
                              ▼
               ┌─────────────────────────────┐
               │                             │
  ┌────────────▼─────┐  ┌───────────────┐  ┌─▼──────────────┐
  │ core.log         │  │ core.proxy.log │  │ core.gateway.log│
  │ (Default logs)   │  │ (Proxy logs)   │  │ (Gateway logs)  │
  └──────────────────┘  └───────────────┘  └────────────────┘
```

Log messages are tagged and routed to the appropriate file based on their component.

## Memory Management

Router Core leverages Rust's ownership model for memory safety while employing several patterns for shared state:

1. **Thread-local state**: Each server thread maintains its own state when possible
2. **Arc/Mutex patterns**: For shared data that needs thread-safe access
3. **Read-write locks**: For configuration that is frequently read but rarely written
4. **Static references**: For global services and configuration

## Error Handling

The error handling strategy employs multiple techniques:

1. **Result propagation**: Functions return `Result` types that are propagated upward
2. **Dedicated error handlers**: Special services for handling HTTP errors
3. **Graceful degradation**: Fallback to default behavior when optimal path fails
4. **Detailed logging**: Structured logging for error diagnosis

## Startup Sequence

1. Initialize configuration system
2. Start logging system
3. Launch protocol interface thread
4. Set up signal handlers
5. Enter main control loop
6. On-demand start of server threads:
   - Gateway services
   - Proxy services
   - Default page handlers

## Shutdown Sequence

1. Receive termination signal (SIGINT or custom Ctrl+X)
2. Set active state flag to false
3. Allow server threads to complete current operations
4. Join threads for clean shutdown

## Performance Considerations

Router Core's architecture is optimized for performance:

- **Minimal copying**: Data buffers are reused when possible
- **Zero-copy networking**: Uses efficient I/O patterns
- **Connection pooling**: Reuses connections to backend services
- **Fine-grained locking**: Minimizes contention for shared resources
- **Non-blocking I/O**: Leverages Tokio for asynchronous network operations
