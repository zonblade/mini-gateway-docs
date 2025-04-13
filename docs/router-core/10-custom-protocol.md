---
sidebar_position: 10
---

# Custom Protocol

The Router Core includes a custom Message Queue-Less (MQLESS) protocol that enables direct communication between clients and the gateway without requiring an intermediate message broker. This document explains the protocol design, implementation, and usage for contributors.

## Overview

The MQLESS protocol provides:

- **Direct TCP communication**: Bypasses message brokers for lower latency
- **Service-oriented architecture**: Services register to handle specific requests
- **URI-like addressing**: Simple, familiar request format
- **Asynchronous handling**: Non-blocking request processing
- **Dynamic service registration**: Services can be added/removed at runtime

## Protocol Specification

### Request Format

The protocol uses a URI-like format for handshakes:

```
gate://<service_name>/<action>?<param1>=<value1>&<param2>=<value2>...
```

Components:
- **gate://** - Protocol identifier
- **service_name** - Target service to handle the request
- **action** - Specific operation to perform
- **parameters** - Key-value pairs providing request details

### Connection Flow

1. Client establishes TCP connection to protocol server
2. Client sends handshake line in MQLESS format
3. Server parses request and routes to appropriate service
4. Service processes the request and sends response
5. Connection remains open for bi-directional communication
6. Either party can close the connection when done

## Architecture

The protocol implementation follows a modular design:

```
┌─────────────────────────────────────────────────────────┐
│                     Protocol Server                      │
│                                                         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │  Connection   │   │  Request      │   │ Service   │  │
│  │  Handling     │   │  Parsing      │   │ Registry  │  │
│  └───────┬───────┘   └───────┬───────┘   └─────┬─────┘  │
│          │                   │                 │         │
│          ▼                   ▼                 ▼         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │  Socket       │   │ Parameter     │   │Service    │  │
│  │  I/O          │   │ Extraction    │   │Handlers   │  │
│  └───────────────┘   └───────────────┘   └───────────┘  │
└─────────────────────────────────────────────────────────┘
```

## Implementation

### Service Protocol Trait

Services implement the `ServiceProtocol` trait to handle requests:

```rust
#[async_trait]
pub trait ServiceProtocol: Send + Sync {
    /// Create a new instance of this service
    fn new() -> Self where Self: Sized;
    
    /// Handle an upstream request from a client
    async fn upstream_peer(
        &self,
        socket: &mut TcpStream,
        buffer: &[u8],
        buffer_size: usize,
        params: &ConnectionParams
    ) -> io::Result<()>;
    
    /// Log service activity
    async fn logging(
        &self,
        params: &ConnectionParams,
        status: Option<&str>,
        metrics: Option<HashMap<String, String>>
    );
}
```

### Service Registration

Services are registered with the protocol server during initialization:

```rust
pub fn register_service<T: ServiceProtocol + 'static>(
    name: &str, 
    service: T
) -> (String, Box<dyn ServiceProtocol>) {
    (name.to_string(), Box::new(service))
}

// Example usage:
let service_handler = init();
let mut handler = service_handler.write().await;
handler.add_service("registry".to_string(), Box::new(Registry::new()));
```

### Service Handler

The `SharedServiceHandler` manages registered services:

```rust
pub type SharedServiceHandler = Arc<RwLock<ServiceHandler>>;

pub struct ServiceHandler {
    services: HashMap<String, Box<dyn ServiceProtocol>>,
}

impl ServiceHandler {
    pub fn new() -> Self {
        Self {
            services: HashMap::new(),
        }
    }
    
    pub fn add_service(&mut self, name: String, service: Box<dyn ServiceProtocol>) {
        self.services.insert(name, service);
    }
    
    pub fn add_services(&mut self, services: Vec<(String, Box<dyn ServiceProtocol>)>) {
        for (name, service) in services {
            self.add_service(name, service);
        }
    }
    
    pub fn get_service(&self, name: &str) -> Option<&Box<dyn ServiceProtocol>> {
        self.services.get(name)
    }
    
    pub fn get_services(&self) -> &HashMap<String, Box<dyn ServiceProtocol>> {
        &self.services
    }
}
```

### Connection Handling

The `handle_connection` function manages protocol communication:

```rust
pub async fn handle_connection(
    mut socket: TcpStream,
    service_handler: Option<SharedServiceHandler>,
) -> io::Result<()> {
    // Get buffer size from configuration
    let buffer_size = ProtocolConfig::BufferSize
        .get()
        .parse::<usize>()
        .unwrap_or(DEFAULT_BUFFER_SIZE);
    
    // Allocate buffer for incoming messages
    let mut buffer = vec![0u8; buffer_size];
    
    // Read the handshake message
    let n = socket.read(&mut buffer).await?;
    
    if n == 0 {
        // Connection closed before handshake
        return Ok(());
    }
    
    // Parse handshake to extract connection parameters
    let params = match parse_connection_params(&buffer[0..n]) {
        Some(p) => p,
        None => {
            // Invalid handshake, return error
            let response = b"ERROR: Invalid protocol format\n";
            socket.write_all(response).await?;
            return Ok(());
        }
    };
    
    // Check if we have a service handler
    if let Some(handler) = service_handler {
        // Find the requested service
        let service_name = &params.service;
        let read_handler = handler.read().await;
        
        if let Some(service) = read_handler.get_service(service_name) {
            // Let the service handle the connection
            service.upstream_peer(&mut socket, &buffer, n, &params).await?;
            
            // Log activity
            service.logging(&params, None, None).await;
            return Ok(());
        }
    }
    
    // No matching service, send error response
    let response = format!("ERROR: Unknown service '{}'\n", params.service);
    socket.write_all(response.as_bytes()).await?;
    
    Ok(())
}
```

### Protocol Server Initialization

The server is initialized with a specific address and buffer size:

```rust
pub async fn init(service_handler: Option<SharedServiceHandler>) -> io::Result<()> {
    // Check if protocol server is enabled
    if !ProtocolConfig::Enabled.xget::<bool>().unwrap_or(DEFAULT_ENABLED) {
        log::info!("Protocol server is disabled.");
        return Ok(());
    }
    
    // Get configured listen address
    let addr = ProtocolConfig::ListenAddr.get();
    log::info!("Starting protocol server on {}", addr);
    
    // Create TCP listener
    let listener = TcpListener::bind(&addr).await?;
    
    // Global server active state flag
    let is_active = Arc::new(AtomicBool::new(true));
    
    // Process incoming connections
    while is_active.load(Ordering::SeqCst) {
        // Accept a new connection
        let (socket, _) = match listener.accept().await {
            Ok(conn) => conn,
            Err(e) => {
                log::error!("Failed to accept connection: {}", e);
                continue;
            }
        };
        
        // Clone service handler for this connection
        let handler_clone = match &service_handler {
            Some(h) => Some(Arc::clone(h)),
            None => None,
        };
        
        // Spawn a task to handle this connection
        tokio::spawn(async move {
            if let Err(e) = handle_connection(socket, handler_clone).await {
                log::error!("Error handling connection: {}", e);
            }
        });
    }
    
    Ok(())
}
```

## Configuration

The protocol server is configured through the `ProtocolConfig` enum:

```rust
pub enum ProtocolConfig {
    ListenAddr,
    BufferSize,
    ProtocolPrefix,
    Enabled,
}

// Default values
pub const DEFAULT_PROTOCOL_PREFIX: &str = "gate://";
pub const DEFAULT_LISTEN_ADDR: &str = "127.0.0.1:30099";
pub const DEFAULT_BUFFER_SIZE: usize = 8192;
pub const DEFAULT_ENABLED: bool = true;

pub fn init_config() {
    ProtocolConfig::ListenAddr.set(DEFAULT_LISTEN_ADDR);
    ProtocolConfig::BufferSize.set(&DEFAULT_BUFFER_SIZE.to_string());
    ProtocolConfig::ProtocolPrefix.set(DEFAULT_PROTOCOL_PREFIX);
    ProtocolConfig::Enabled.xset::<bool>(DEFAULT_ENABLED);
}
```

## Connection Parameters

Request parameters are parsed and stored in a structured format:

```rust
pub struct ConnectionParams {
    pub service: String,
    pub action: String,
    pub parameters: HashMap<String, String>,
    pub raw: String,
}
```

## Example Service Implementation

Here's a simplified example of a service implementation:

```rust
struct DataRegistry {
    data: RwLock<HashMap<String, String>>,
}

#[async_trait]
impl ServiceProtocol for DataRegistry {
    fn new() -> Self {
        Self {
            data: RwLock::new(HashMap::new()),
        }
    }
    
    async fn upstream_peer(
        &self,
        socket: &mut TcpStream,
        _buffer: &[u8],
        _buffer_size: usize,
        params: &ConnectionParams,
    ) -> io::Result<()> {
        match params.action.as_str() {
            "get" => {
                let key = params.parameters.get("key").unwrap_or(&"".to_string());
                let data = self.data.read().await;
                let value = data.get(key).unwrap_or(&"".to_string());
                socket.write_all(value.as_bytes()).await?;
            },
            "set" => {
                let key = params.parameters.get("key").unwrap_or(&"".to_string());
                let value = params.parameters.get("value").unwrap_or(&"".to_string());
                let mut data = self.data.write().await;
                data.insert(key.clone(), value.clone());
                socket.write_all(b"OK").await?;
            },
            _ => {
                socket.write_all(b"ERROR: Unknown action").await?;
            }
        }
        
        Ok(())
    }
    
    async fn logging(
        &self,
        params: &ConnectionParams,
        _status: Option<&str>,
        _metrics: Option<HashMap<String, String>>,
    ) {
        log::info!(
            "[PROTOCOL] Service: {}, Action: {}", 
            params.service, 
            params.action
        );
    }
}
```

## Protocol URI Parser

The `parse_connection_params` function extracts parameters from protocol URIs:

```rust
pub fn parse_connection_params(buffer: &[u8]) -> Option<ConnectionParams> {
    // Convert buffer to string
    let raw = match std::str::from_utf8(buffer) {
        Ok(s) => s.trim_end(),
        Err(_) => return None,
    };
    
    // Get protocol prefix from configuration
    let prefix = ProtocolConfig::ProtocolPrefix.get();
    
    // Check if message starts with protocol prefix
    if !raw.starts_with(&prefix) {
        return None;
    }
    
    // Remove prefix and split by '/'
    let uri = &raw[prefix.len()..];
    let parts: Vec<&str> = uri.split('/').collect();
    
    if parts.len() < 2 {
        return None;
    }
    
    // Extract service and action
    let service = parts[0].to_string();
    
    // Handle action and parameters
    let action_and_params = parts[1];
    let action_parts: Vec<&str> = action_and_params.split('?').collect();
    let action = action_parts[0].to_string();
    
    // Parse parameters if present
    let mut parameters = HashMap::new();
    if action_parts.len() > 1 {
        let params_str = action_parts[1];
        for param in params_str.split('&') {
            let kv: Vec<&str> = param.split('=').collect();
            if kv.len() == 2 {
                parameters.insert(kv[0].to_string(), kv[1].to_string());
            }
        }
    }
    
    Some(ConnectionParams {
        service,
        action,
        parameters,
        raw: raw.to_string(),
    })
}
```

## Client Usage Example

Here's how a client might use the protocol:

```rust
async fn connect_to_protocol_server() -> io::Result<()> {
    // Connect to protocol server
    let mut stream = TcpStream::connect("127.0.0.1:30099").await?;
    
    // Send handshake
    let handshake = "gate://registry/set?key=status&value=active";
    stream.write_all(handshake.as_bytes()).await?;
    
    // Read response
    let mut buffer = [0u8; 1024];
    let n = stream.read(&mut buffer).await?;
    println!("Response: {}", std::str::from_utf8(&buffer[0..n]).unwrap());
    
    // Close connection
    Ok(())
}
```

## Thread Safety Considerations

The protocol implementation ensures thread safety through:

1. **Arc/RwLock for Service Handler**: Shared access to the service registry
2. **Trait bounds for Service Protocol**: All services must be Send + Sync
3. **Per-connection task isolation**: Each connection runs in its own task
4. **Immutable service references**: Service implementations accessed through read-only references

## Error Handling

The protocol implements robust error handling:

```rust
// Connection error handling
if let Err(e) = handle_connection(socket, handler_clone).await {
    log::error!("Error handling connection: {}", e);
}

// Service lookup error handling
if let Some(service) = read_handler.get_service(service_name) {
    // Service found, handle request
} else {
    // Service not found, return error
    let response = format!("ERROR: Unknown service '{}'\n", params.service);
    socket.write_all(response.as_bytes()).await?;
}

// Protocol parsing error handling
let params = match parse_connection_params(&buffer[0..n]) {
    Some(p) => p,
    None => {
        // Invalid handshake, return error
        let response = b"ERROR: Invalid protocol format\n";
        socket.write_all(response).await?;
        return Ok(());
    }
};
```

## Integration with Main Application

The protocol server is started as part of the application initialization:

```rust
pub fn start_interface() {
    init_config();
    thread::spawn(|| {
        let runtime = tokio::runtime::Runtime::new().expect("Failed to create Tokio runtime");
        runtime.block_on(async {
            log::debug!("Starting protocol server...");
            // Initialize the service handler
            let service_handler = services::init();

            // Register example services
            let mut handler = service_handler.write().await;
            let registry = app::registry::DataRegistry::new();
            let services = vec![services::register_service("registry", registry)];
            handler.add_services(services);
            
            // Start the protocol server
            if let Err(e) = init(Some(Arc::clone(&service_handler))).await {
                log::error!("Protocol server failed to start: {}", e);
            }
        });
    });
}
```

## Security Considerations

When implementing custom protocol services, consider:

1. **Authentication**: Validate client identity before processing requests
2. **Authorization**: Verify client permissions for requested actions
3. **Input validation**: Sanitize all input parameters
4. **Rate limiting**: Prevent abuse through request limiting
5. **Timeout handling**: Set appropriate timeouts for connections
6. **Encryption**: Consider using TLS for sensitive communications
