---
sidebar_position: 5
---

# Logging System

The Router Core logging system provides comprehensive diagnostic capabilities through component-specific log files and structured log formats. This document explains the architecture, implementation, and usage of the logging system for contributors.

## Overview

The logging system implements:

- **Tag-based routing**: Different components log to separate files
- **Structured format**: Consistent log format across components
- **Rotatable files**: Log files designed for rotation
- **Configurable levels**: Different verbosity levels for various components
- **Custom formatting**: Timestamp and component prefixes

## Architecture

The logging system uses a custom writer that routes logs to appropriate files:

```
┌─────────────────────────────────────────────────────────┐
│                   Logging System                         │
│                                                         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │ Log           │   │ Component     │   │ File      │  │
│  │ Collection    │   │ Filtering     │   │ Writing   │  │
│  └───────┬───────┘   └───────┬───────┘   └─────┬─────┘  │
│          │                   │                 │         │
│          ▼                   ▼                 ▼         │
│  ┌───────────────────┐  ┌───────────────┐  ┌───────────┐│
│  │ Log Level         │  │Tag Extraction │  │Directory  ││
│  │ Management        │  │& Routing      │  │Management ││
│  └───────────────────┘  └───────────────┘  └───────────┘│
└─────────────────────────────────────────────────────────┘
```

## Implementation

### Initialization

The logging system is initialized at application startup:

```rust
pub fn setup_tag_based_logging() -> Result<(), Box<dyn std::error::Error>> {
    // Set global log level
    std::env::set_var("RUST_LOG", "info");

    // Determine the log file path
    let log_dir = get_default_log_dir();

    // Create the log directory if it doesn't exist
    let log_dir_path = Path::new(&log_dir);
    if !log_dir_path.exists() {
        fs::create_dir_all(log_dir_path)?;
    }

    // Define log file paths
    let log_path_default    = format!("{}/core.log"         , log_dir);
    let log_path_proxy      = format!("{}/core.proxy.log"   , log_dir);
    let log_path_gateway    = format!("{}/core.gateway.log" , log_dir);
    let log_path_netlisten  = format!("{}/core.net.log"     , log_dir);
}
```

### Log File Locations

The system determines the appropriate log directory based on the platform:

```rust
fn get_default_log_dir() -> String {
    #[cfg(target_os = "linux")]
    {
        // On Linux, use /var/log for system services, otherwise use the home directory
        if is_running_as_service() {
            return "/var/log/runegram".to_string();
        } else {
            if let Ok(home) = std::env::var("HOME") {
                return format!("{}/.local/share/runegram/log", home);
            }
        }
    }

    #[cfg(target_os = "macos")]
    {
        if let Some(dirs) = dirs::home_dir() {
            return format!("{}/Library/Logs/runegram", dirs.display());
        }
    }

    // Fallback to current directory
    "./log".to_string()
}
```

### Log File Setup

The system configures separate log writers for each component:

```rust
// Configure log writers with appropriate prefixes
let writer_default = WriterBuilder::new()
    .with_file(log_path_default)
    .with_prefix("[CORE] ")
    .build();

let writer_proxy = WriterBuilder::new()
    .with_file(log_path_proxy)
    .with_prefix("[PROXY] ")
    .build();

let writer_gateway = WriterBuilder::new()
    .with_file(log_path_gateway)
    .with_prefix("[GATEWAY] ")
    .build();

let writer_netlisten = WriterBuilder::new()
    .with_file(log_path_netlisten)
    .with_prefix("[NET] ")
    .build();
```

### Component Tagging

Log messages are tagged by their source component:

```rust
// Gateway component log
log::info!("[GWX] Response code: {}", response_code);

// Proxy component log
log::info!("[PXY] |ID:{}, STATUS:{}, SIZE:{}, COMMENT:- |", id, status, n);

// Protocol component log
log::debug!("Starting protocol server...");
```

### Log Message Routing

The custom log router inspects message content to route to the appropriate file:

```rust
fn route_log(record: &log::Record) -> Option<LogDestination> {
    let msg = record.args().to_string();
    
    // Route based on tag prefix
    if msg.starts_with("[GWX]") || msg.starts_with("[GATEWAY]") {
        Some(LogDestination::Gateway)
    } else if msg.starts_with("[PXY]") || msg.starts_with("[PROXY]") {
        Some(LogDestination::Proxy)
    } else if msg.starts_with("[NET]") {
        Some(LogDestination::NetListen)
    } else {
        Some(LogDestination::Default)
    }
}
```

### Log Formatting

Log messages include timestamps and severity levels:

```rust
// Format with timestamp, level and message
let formatted = format!(
    "{} {} {}",
    chrono::Local::now().format("%Y-%m-%d %H:%M:%S"),
    record.level(),
    msg
);
```

## Writer Interface

The `writer_start()` function initializes the entire logging system:

```rust
pub fn writer_start() {
    if let Err(e) = setup_tag_based_logging() {
        eprintln!("Failed to set up logging: {}", e);
        // Fall back to basic console logging
        std::env::set_var("RUST_LOG", "info");
        env_logger::init();
    }
}
```

## Log Levels

Different components can have different log levels:

```rust
// Control detail level for different components
std::env::set_var("RUST_LOG", "info,router_core::app=debug,router_core::system::protocol=debug");
```

## Structured Log Formats

The proxy component uses a parseable log format:

```
[PXY] |ID:12345, STATUS:01, SIZE:1024, COMMENT:- |
```

Where:
- **ID**: Connection identifier
- **STATUS**: Operation type code (01 = downstream→upstream data)
- **SIZE**: Bytes transferred
- **COMMENT**: Optional human-readable note

## Error Reporting

Errors are logged with context and error codes:

```rust
if let Some(os_err) = e.raw_os_error() {
    match os_err {
        54 => log::info!("[PXY] |ID:{}, STATUS:{}, SIZE:0, COMMENT:CONNECTION_RESET |", id, status),
        60 => log::info!("[PXY] |ID:{}, STATUS:{}, SIZE:0, COMMENT:OPERATION_TIMEOUT |", id, status),
        _ => log::error!("Error reading from {}: {} (code: {:?})", prefix, e, os_err),
    }
} else {
    log::error!("Error reading from {}: {}", prefix, e);
}
```

## Developer Usage

### Choosing the Right Log Level

- **ERROR**: For problems that block operation
- **WARN**: For non-fatal issues that may require attention
- **INFO**: For normal operation information
- **DEBUG**: For detailed diagnostic information
- **TRACE**: For very detailed flow information

### Component Tagging

To ensure logs go to the right files, tag your messages:

```rust
// Gateway component
log::info!("[GWX] Request received: {}", path);

// Proxy component
log::info!("[PXY] Connection from: {}", addr);

// Network listener
log::info!("[NET] Binding to: {}", addr);
```

### Structured Logging

For machine-parseable logs, use the pipe format:

```rust
log::info!("|COMPONENT:{}, EVENT:{}, DURATION:{}, RESULT:{} |", 
           component_name, event_type, duration_ms, result_code);
```

## Integration with External Loggers

The logging system can be integrated with external systems:

```rust
pub fn init_external_logger(config: &LogConfig) -> Result<(), LogError> {
    // Initialize our logging system
    setup_tag_based_logging()?;
    
    // Configure additional external logger if needed
    if config.enable_syslog {
        setup_syslog_integration()?;
    }
    
    Ok(())
}
```

## Performance Considerations

The logging system is designed for minimal performance impact:

- **Asynchronous writing**: Log writing doesn't block request processing
- **Level filtering**: Low-priority messages can be efficiently filtered
- **Buffer pooling**: Reduces memory allocations
- **Conditional compilation**: Debug logs can be compiled out in release builds
