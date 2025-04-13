---
sidebar_position: 1
---

# Introduction

Router Core is the central component of the mini-gateway system, providing high-performance network routing, proxying, and gateway functionality. It's built on the [Pingora](https://github.com/cloudflare/pingora) framework (using a custom fork) to provide efficient and reliable networking capabilities.

## Overview

Router Core serves as:

- **TCP/TLS Proxy**: Routes traffic based on connection parameters (address, SNI, etc.)
- **HTTP Gateway**: Routes HTTP requests based on path patterns with rewriting capabilities
- **Protocol Server**: Provides a custom protocol for direct client-to-gateway communication
- **Configuration Management**: Supports dynamic configuration changes without restarts

The system is designed with high performance, low latency, and reliability as primary goals, making it suitable for production environments where network routing is critical.

## Core Features

- **High-performance proxy routing**: Efficiently routes TCP and TLS traffic
- **HTTP path-based routing**: Routes requests based on URL path patterns with regex support
- **TLS termination**: Handles SSL/TLS connections with SNI-based routing
- **Dynamic configuration**: Updates routing rules without service interruption
- **Custom protocol support**: Provides direct, message queue-less communication
- **Multi-threaded architecture**: Maximizes performance on multi-core systems
- **Comprehensive logging**: Structured logging with component-specific log files

## Technology Stack

- **Rust**: For memory safety, performance, and concurrency
- **Pingora**: High-performance HTTP/TCP proxy framework (custom fork)
- **Tokio**: Asynchronous runtime for efficient I/O operations
- **mini-config**: Internal configuration management system

## Target Audience

This documentation is intended for **contributors** to the Router Core component, who need to understand the internal architecture, code structure, and implementation details. It assumes familiarity with:

- Rust programming language and its ecosystem
- Network programming concepts (TCP, TLS, HTTP)
- Proxy and gateway architectural patterns

If you're an end-user looking for usage instructions, please refer to the general user documentation instead.

## Repository Structure

Router Core is organized into several key modules:

```
router-core/
├── src/
│   ├── app/                  # Application-level functionality
│   │   ├── gateway.rs        # HTTP gateway implementation
│   │   └── proxy.rs          # TCP/TLS proxy implementation
│   ├── config.rs             # Configuration system
│   ├── service/              # Service implementations
│   │   ├── proxy.rs          # Proxy service definitions
│   │   └── gateway.rs        # Gateway service definitions
│   ├── system/               # System-level components
│   │   ├── default_page/     # Default error page handlers
│   │   ├── protocol/         # Custom protocol implementation
│   │   ├── server.rs         # Server initialization and management
│   │   └── writer/           # Logging system
│   └── main.rs               # Application entry point
```

## Getting Started for Contributors

To start contributing to Router Core:

1. Familiarize yourself with the [architecture](./01-architecture.md)
2. Understand the [gateway](./03-gateway.md) and [proxy](./04-proxy.md) components
3. Review the [logging system](./05-logging.md) to understand diagnostic capabilities
4. Explore the [custom protocol](./10-custom-protocol.md) for specialized communications

The following sections provide detailed information about each component of the Router Core system.
