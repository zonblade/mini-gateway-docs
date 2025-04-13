---
sidebar_position: 3
---

# Gateway

The Gateway component of Router Core is a high-performance HTTP router that directs incoming requests to appropriate backend services based on path patterns. This document covers its architecture, implementation details, and usage for contributors.

## Overview

The Gateway handles HTTP/HTTPS traffic routing with advanced path-based routing capabilities:

- **Path pattern matching**: Uses regular expressions to match request paths
- **URL rewriting**: Transforms URLs before forwarding to backends
- **Query preservation**: Maintains query parameters during rewriting
- **Dynamic configuration**: Updates routing rules without restarts
- **Prioritized rules**: Higher priority rules evaluated first

## Architecture

The Gateway implements the Pingora `ProxyHttp` trait to provide HTTP routing functionality:

```
┌─────────────────────────────────────────────────────────┐
│                    GatewayApp                           │
│                                                         │
│  ┌───────────────┐   ┌───────────────┐   ┌───────────┐  │
│  │ Configuration │   │  Rule         │   │   HTTP    │  │
│  │ Management    │   │  Evaluation   │   │ Proxying  │  │
│  └───────┬───────┘   └───────┬───────┘   └─────┬─────┘  │
│          │                   │                 │         │
│          ▼                   ▼                 ▼         │
│  ┌───────────────────┐  ┌───────────────┐  ┌───────────┐│
│  │RedirectRule Storage│ │Path Transform │  │Request    ││
│  │(thread-safe)      │ │Logic          │  │Forwarding ││
│  └───────────────────┘  └───────────────┘  └───────────┘│
└─────────────────────────────────────────────────────────┘
```

## Implementation

### Gateway Application

The `GatewayApp` struct is the main implementation that provides HTTP routing:

```rust
pub struct GatewayApp {
    source: String,
    last_check_time: RwLock<Instant>,
    check_interval: Duration,
}
```

- **source**: The listener address this gateway instance is bound to
- **last_check_time**: Timestamp of the last configuration check (with thread-safe access)
- **check_interval**: Time between configuration refresh checks

### Redirect Rule

Each routing rule is represented by a `RedirectRule` struct:

```rust
struct RedirectRule {
    pattern: Regex,
    target: String,
    alt_listen: String,
    alt_target: Option<BasicPeer>,
    priority: usize,
}
```

- **pattern**: Regular expression to match against request paths
- **target**: Template for path transformation (with capture group syntax)
- **alt_listen**: The source address:port this rule applies to
- **alt_target**: Target backend service to forward requests to
- **priority**: Rule evaluation priority (higher values checked first)

### Rule Storage

Rules are stored in a thread-safe global collection:

```rust
// Thread-safe shared collection of rules
static REDIRECT_RULES: LazyLock<RwLock<HashMap<String, Vec<RedirectRule>>>> = 
    LazyLock::new(|| RwLock::new(HashMap::new()));

// For tracking configuration changes
static SAVED_CONFIG_ID: LazyLock<RwLock<String>> = 
    LazyLock::new(|| RwLock::new(String::new()));
```

This design allows:
- Multiple gateway instances to share rule configurations
- Thread-safe updates to rules without stopping request processing
- Efficiency through avoiding unnecessary rule parsing

### Initialization

The Gateway is initialized with:

```rust
pub fn new(alt_source: &str) -> Self {
    log::debug!("Creating GatewayApp with source: {}", alt_source);
    let app = GatewayApp {
        source: alt_source.to_string(),
        last_check_time: RwLock::new(Instant::now()),
        check_interval: Duration::from_secs(5),
    };
    app.populate();
    app
}
```

This creates a Gateway instance listening on the specified address and loads its initial rules.

### Configuration Loading

The `populate()` method loads and refreshes configuration:

```rust
fn populate(&self) {
    // Check if configuration changed
    let config_id = config::RoutingData::GatewayID.get();
    
    // Skip if no changes
    if saved_id == config_id { return; }
    
    // Load gateway node configurations
    let node = config::RoutingData::GatewayRouting.xget::<Vec<GatewayNode>>();
    
    // Convert configuration to redirect rules
    // Filter rules for this gateway instance
    // Sort by priority
    
    // Update the shared rule storage
}
```

### Request Processing

The `upstream_peer` method handles the core routing logic:

```rust
async fn upstream_peer(
    &self,
    session: &mut Session,
    _ctx: &mut Self::CTX,
) -> pingora::Result<Box<HttpPeer>> {
    // Extract the request path
    let path = session.req_header().uri.path();
    
    // Check if configuration should be refreshed
    if self.should_check_config() {
        self.populate();
    }
    
    // Get the rules for this source
    let rules = self.get_rules();
    
    // Try to match path against redirect rules
    for rule in &rules {
        if let Some(captures) = rule.pattern.captures(path) {
            if let Some(alt_target) = &rule.alt_target {
                // Process path transformation
                // Preserve query parameters
                // Update request URI
                // Create peer for target backend
                return Ok(Box::new(new_peer));
            }
        }
    }
    
    // Default fallback if no rules match
    // ...
}
```

### Path Transformation

The path transformation logic handles regex capture groups:

```rust
// Check if we need replacements
let needs_replacement = target_ref.contains('$');

// Only process replacements if needed
let final_path = if needs_replacement {
    let mut new_path = target_ref.to_string();
    
    // Process capture group references
    for i in 1..captures.len() {
        if let Some(capture) = captures.get(i) {
            replacements.push((format!("${}", i), capture.as_str()));
        }
    }
    
    // Apply replacements
    for (pattern, replacement) in replacements {
        new_path = new_path.replace(&pattern, replacement);
    }
    
    new_path
} else {
    target_ref.to_string()
};
```

### Query Parameter Handling

Query parameters are preserved during URL transformation:

```rust
// Get original query parameters
let query_opt = uri_ref.query();

// Preserve them in the transformed URL
let new_path_and_query = if let Some(query) = query_opt {
    format!("{}?{}", final_path, query)
} else {
    final_path
};
```

### Request Logging

The `logging` method tracks request metrics:

```rust
async fn logging(
    &self,
    session: &mut Session,
    _e: Option<&pingora::Error>,
    _ctx: &mut Self::CTX,
) {
    let response_code = session
        .response_written()
        .map_or(0, |resp| resp.status.as_u16());
   log::info!("[GWX] Response code: {}", response_code);
}
```

## Configuration Model

Gateway routes are configured using the `GatewayNode` struct:

```rust
pub struct GatewayNode {
    pub priority: u8,
    pub addr_listen: String,
    pub addr_target: String,
    pub path_listen: String,
    pub path_target: String,
}
```

These are stored in the configuration system under the `GatewayRouting` key.

## Performance Optimizations

The Gateway implementation includes several optimizations:

1. **Rule caching**: Rules are parsed once and cached
2. **Periodic configuration checks**: Avoid checking configuration on every request
3. **Request path transformation optimization**: Only transform when needed
4. **Prioritized evaluation**: Higher priority rules checked first
5. **Thread-safe interior mutability**: For concurrent access patterns

## Example Rules

### Basic Path Routing

```
Pattern: "^/api/users/(.*)$"
Target: "/users-service/user/$1"
```

This would transform `/api/users/123` to `/users-service/user/123`

### Multi-capture Transformation

```
Pattern: "^/api/([^/]+)/([^/]+)/(.*)$"
Target: "/services/$1/$2/$3"
```

This would transform `/api/auth/users/check` to `/services/auth/users/check`

### Path Stripping

```
Pattern: "^/public/(.*)$"
Target: "/$1"
```

This would transform `/public/index.html` to `/index.html`

## Debug Logging

To aid in debugging, the Gateway logs:

- Gateway initialization
- Rule loading and updates
- Path matching attempts
- Default fallback cases
- Response status codes

## Error Handling

If a request doesn't match any rules, it's sent to the default 404 handler:

```rust
// Default fallback if no rules match
let port_str = DEFAULT_PORT.p404;
let parts: Vec<&str> = port_str.split(':').collect();
let addr = (parts[0], parts[1].parse::<u16>().unwrap_or(80));
log::debug!("No matching rules, connecting to default {addr:?}");
let peer = Box::new(HttpPeer::new(addr, false, String::new()));
Ok(peer)
```

## Thread Safety

The Gateway maintains thread safety through:

1. **Read-Write Locks**: For rule storage and configuration ID
2. **Interior Mutability Pattern**: For the last check timestamp
3. **Immutable Rule Processing**: Once loaded, rules are treated as immutable
4. **Atomic Configuration Updates**: Rules are replaced atomically

## Integration with Server

The Gateway is instantiated in the server initialization:

```rust
// Create a gateway service for each address
for addr in addr.iter() {
    log::info!("Creating gateway service for address: {}", addr);
    let mut my_gateway_service = pingora::proxy::http_proxy_service(
        &my_server.configuration,
        GatewayApp::new(addr),
    );
    my_gateway_service.add_tcp(addr);
    my_gateway.push(Box::new(my_gateway_service));
}
```
