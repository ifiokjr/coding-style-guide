# Rust Style Guide

A comprehensive guide for writing readable, maintainable Rust code with emphasis on early returns, thoughtful whitespace, and comprehensive documentation.

## Table of Contents

1. [Documentation Standards](#documentation-standards)
2. [Early Returns and Control Flow](#early-returns-and-control-flow)
3. [Variable Declaration and Grouping](#variable-declaration-and-grouping)
4. [Struct Patterns and Builders](#struct-patterns-and-builders)
5. [Function Arguments](#function-arguments)
6. [Whitespace and Grouping](#whitespace-and-grouping)
7. [Error Handling](#error-handling)
8. [Module Structure](#module-structure)

---

## Documentation Standards

### Every Item Gets Documentation

Every function, constant, struct, enum, and module must have documentation explaining:
- **What** it does
- **Why** it exists
- **How** to use it (for public APIs)

```rust
/// Maximum number of retry attempts for failed operations.
/// 
/// # Why This Exists
/// Network operations can fail transiently due to temporary issues
/// like connection resets or rate limiting. This constant provides
/// a sensible default that balances reliability with performance.
///
/// # When to Change
/// - Increase for critical operations where success is mandatory
/// - Decrease for time-sensitive operations where failure is acceptable
const MAX_RETRIES: u32 = 3;

/// Transforms raw sensor data into a standardized format.
///
/// # Why This Exists
/// Different sensor vendors provide data in different formats.
/// This function centralizes the normalization logic, ensuring
/// all downstream consumers receive consistent data regardless
/// of the source sensor.
///
/// # Use Cases
/// - Ingesting data from IoT devices with varying protocols
/// - Normalizing historical data from different sensor generations
/// - Preparing data for machine learning pipelines
///
/// # Examples
///
/// ```
/// use my_crate::normalize_sensor_data;
///
/// let raw = RawSensorData {
///     vendor_id: "acme_1000",
///     timestamp: 1234567890,
///     values: vec![23.5, 45.2, 89.1],
/// };
///
/// let normalized = normalize_sensor_data(raw)?;
/// assert_eq!(normalized.temperature_celsius, 23.5);
/// ```
pub fn normalize_sensor_data(raw: RawSensorData) -> Result<NormalizedData, SensorError> {
    // Implementation
}
```

### Module-Level Documentation

Every module file should have comprehensive documentation at the top:

```rust
//! Sensor data processing and normalization.
//!
//! # Why This Module Exists
//! IoT deployments use sensors from multiple vendors with different
//! data formats. This module provides a unified interface for:
//! - Ingesting raw sensor readings
//! - Normalizing to a common schema
//! - Validating data quality
//! - Transforming for downstream consumers
//!
//! # Key Components
//!
//! - [`normalize_sensor_data`]: Converts vendor-specific formats to standard schema
//! - [`SensorValidator`]: Validates data quality and completeness
//! - [`DataTransformer`]: Applies business logic transformations
//!
//! # Quick Start
//!
//! ```rust
//! use my_crate::sensor::{normalize_sensor_data, SensorConfig};
//!
//! let config = SensorConfig::builder()
//!     .vendor("acme")
//!     .max_temp(100.0)
//!     .build();
//!
//! let data = normalize_sensor_data(raw_data, &config)?;
//! ```

use std::collections::HashMap;

// ... module contents
```

### Internal Documentation

Even non-exported items deserve documentation:

```rust
/// Parses vendor-specific timestamp formats into Unix timestamps.
///
/// # Why This Exists
/// Different vendors use different timestamp precisions (seconds, millis, micros)
/// and formats (Unix, ISO 8601, proprietary). This centralizes the parsing
/// logic so normalization functions don't need to handle format detection.
///
/// # Supported Formats
/// - Unix seconds (10 digits)
/// - Unix milliseconds (13 digits)
/// - Unix microseconds (16 digits)
/// - ISO 8601 strings
fn parse_vendor_timestamp(raw: &str, vendor: &str) -> Result<u64, TimestampError> {
    // Implementation
}
```

---

## Early Returns and Control Flow

### The Rule: Exit Early, Stay Shallow

Deeply nested code is an orange flag. Prefer early returns to nesting.

**Avoid - Deep Nesting:**

```rust
// Don't do this - hard to follow the logic
fn process_user_request(request: Request) -> Result<Response, Error> {
    if let Some(user) = request.user {
        if user.is_active {
            if user.has_permission(Permission::Write) {
                if request.data.is_valid() {
                    let result = write_to_database(&request.data)?;
                    if result.rows_affected > 0 {
                        Ok(Response::success(result.id))
                    } else {
                        Err(Error::NoRowsAffected)
                    }
                } else {
                    Err(Error::InvalidData)
                }
            } else {
                Err(Error::NoPermission)
            }
        } else {
            Err(Error::UserInactive)
        }
    } else {
        Err(Error::NoUser)
    }
}
```

**Prefer - Early Returns:**

```rust
/// Processes a user request after validating all preconditions.
///
/// # Why This Exists
/// User requests must pass multiple validation layers (authentication,
/// authorization, data validation) before processing. This function
/// centralizes the validation logic and returns early on any failure,
/// making the "happy path" clear and the error cases explicit.
fn process_user_request(request: Request) -> Result<Response, Error> {
    // Extract user or fail fast
    let user = request.user.ok_or(Error::NoUser)?;
    
    // Validate user state
    if !user.is_active {
        return Err(Error::UserInactive);
    }
    
    // Check permissions
    if !user.has_permission(Permission::Write) {
        return Err(Error::NoPermission);
    }
    
    // Validate request data
    if !request.data.is_valid() {
        return Err(Error::InvalidData);
    }
    
    // Now we're on the happy path - all validations passed
    let result = write_to_database(&request.data)?;
    
    if result.rows_affected == 0 {
        return Err(Error::NoRowsAffected);
    }
    
    Ok(Response::success(result.id))
}
```

### Pattern Matching with Early Returns

```rust
/// Processes payment based on payment method.
///
/// # Why This Exists
/// Payment processing varies significantly by method (card, bank transfer,
/// crypto). Using early returns in match arms keeps each payment path
/// self-contained and prevents deep nesting when handling method-specific
/// validation.
fn process_payment(payment: Payment) -> Result<Receipt, PaymentError> {
    match payment.method {
        PaymentMethod::CreditCard(card) => {
            // Validate card first
            if card.is_expired() {
                return Err(PaymentError::CardExpired);
            }
            
            if !card.has_valid_cvv() {
                return Err(PaymentError::InvalidCVV);
            }
            
            // Process the validated card
            process_card_payment(&card, payment.amount)
        }
        
        PaymentMethod::BankTransfer(details) => {
            // Validate bank details
            if !details.account_verified {
                return Err(PaymentError::UnverifiedAccount);
            }
            
            // Bank transfers take longer - create pending receipt
            create_pending_receipt(&details, payment.amount)
        }
        
        PaymentMethod::Crypto(wallet) => {
            // Validate wallet has sufficient balance
            let balance = get_wallet_balance(&wallet.address)?;
            
            if balance < payment.amount {
                return Err(PaymentError::InsufficientFunds);
            }
            
            // Crypto is irreversible - double-check amount
            if payment.amount > CryptoPayment::MAX_AMOUNT {
                return Err(PaymentError::AmountTooLarge);
            }
            
            process_crypto_payment(&wallet, payment.amount)
        }
    }
}
```

### While Loops with Early Continue

```rust
/// Processes log entries, skipping malformed entries.
///
/// # Why This Exists
/// Log files often contain corrupted or incomplete entries due to
/// crashes or disk issues. Rather than failing the entire batch,
/// we skip problematic entries and report them separately.
fn process_log_entries(entries: Vec<LogEntry>) -> ProcessingResult {
    let mut valid_entries = Vec::new();
    let mut failed_entries = Vec::new();
    
    for entry in entries {
        // Skip entries without timestamps - can't order them
        if entry.timestamp.is_none() {
            failed_entries.push((entry, "Missing timestamp"));
            continue;
        }
        
        // Skip entries with invalid log levels
        if !entry.level.is_valid() {
            failed_entries.push((entry, "Invalid log level"));
            continue;
        }
        
        // Skip entries where message is empty
        if entry.message.trim().is_empty() {
            failed_entries.push((entry, "Empty message"));
            continue;
        }
        
        // Entry passed all validations - process it
        let normalized = normalize_entry(entry);
        valid_entries.push(normalized);
    }
    
    ProcessingResult {
        processed: valid_entries.len(),
        failed: failed_entries.len(),
        valid_entries,
        failed_entries,
    }
}
```

---

## Variable Declaration and Grouping

### Variables at the Top

Declare variables at the start of functions when possible:

```rust
/// Configures the HTTP client with retry and timeout policies.
///
/// # Why This Exists
/// HTTP clients need consistent configuration across the application
/// for reliability (retries) and resource management (timeouts).
/// This function centralizes that configuration.
fn create_http_client(config: &AppConfig) -> Result<Client, ClientError> {
    // Group 1: Configuration extraction
    // These values control how the client behaves
    let timeout_seconds = config.http.timeout_secs;
    let retry_attempts = config.http.retry_attempts;
    let pool_size = config.http.connection_pool_size;
    
    // Security: Force HTTPS in production environments
    // This prevents accidental cleartext transmission of sensitive data
    let enforce_https = config.environment.is_production();
    
    // Group 2: Builder configuration
    // Building the client happens in phases
    let mut builder = Client::builder()
        .timeout(Duration::from_secs(timeout_seconds))
        .pool_max_idle_per_host(pool_size);
    
    if enforce_https {
        builder = builder.https_only(true);
    }
    
    // Group 3: Final construction
    // All configuration is complete, build the client
    let client = builder
        .build()
        .map_err(ClientError::BuildFailed)?;
    
    Ok(client)
}
```

### Complex Variable Declarations

Long declarations get breathing room:

```rust
/// Fetches user data from multiple sources and merges the results.
///
/// # Why This Exists
/// User data is distributed across multiple services (profile service,
/// preference service, authorization service). This function coordinates
/// the parallel fetching and merging of that data.
async fn fetch_user_data(user_id: UserId) -> Result<MergedUserData, FetchError> {
    // Configuration for each service endpoint
    // These URLs come from environment configuration
    let profile_url = format!("{}/users/{}", config.profile_service_url, user_id);
    let prefs_url = format!("{}/users/{}/preferences", config.prefs_service_url, user_id);
    let auth_url = format!("{}/users/{}/roles", config.auth_service_url, user_id);
    
    // Request configuration shared across all requests
    // Using a common timeout and headers for consistency
    let request_builder = client
        .request(Method::GET, &profile_url)
        .timeout(Duration::from_secs(5))
        .header("X-Request-ID", generate_request_id());
    
    // Spawn concurrent fetches
    // Each fetch is independent, so we can run them in parallel
    let profile_fetch = fetch_profile_data(&profile_url, request_builder.clone());
    let prefs_fetch = fetch_preferences(&prefs_url, request_builder.clone());
    let auth_fetch = fetch_authorization(&auth_url, request_builder.clone());
    
    // Await all results
    // If any fails, we fail fast - partial data is not useful
    let (profile, prefs, roles) = tokio::try_join!(profile_fetch, prefs_fetch, auth_fetch)?;
    
    // Merge into unified view
    // This is the final result that callers actually want
    let merged = MergedUserData::from_parts(profile, prefs, roles);
    
    Ok(merged)
}
```

---

## Struct Patterns and Builders

### Using typed-builder for Complex Structs

When structs have many optional fields or sensible defaults, use `typed-builder`:

```rust
use typed_builder::TypedBuilder;

/// Configuration for the background job processor.
///
/// # Why This Exists
/// Background jobs need tunable parameters for different workloads.
/// The builder pattern allows callers to specify only what they need
/// while providing sensible defaults for everything else.
#[derive(TypedBuilder)]
pub struct JobProcessorConfig {
    /// Maximum number of concurrent jobs to process.
    /// Higher values increase throughput but use more resources.
    #[builder(default = 4)]
    pub max_concurrent: usize,
    
    /// How long to wait between polling for new jobs.
    #[builder(default = Duration::from_secs(5))]
    pub poll_interval: Duration,
    
    /// Maximum retry attempts for failed jobs.
    #[builder(default = 3)]
    pub max_retries: u32,
    
    /// Whether to process jobs in priority order.
    #[builder(default = true)]
    pub prioritize: bool,
    
    /// Optional hook called when a job fails permanently.
    #[builder(default, setter(strip_option))]
    pub on_failure: Option<Arc<dyn Fn(&Job, &JobError) + Send + Sync>>,
    
    /// Optional custom serializer for job payloads.
    #[builder(default, setter(strip_option))]
    pub serializer: Option<Box<dyn JobSerializer>>,
}

// Usage:
// let config = JobProcessorConfig::builder()
//     .max_concurrent(8)
//     .poll_interval(Duration::from_secs(1))
//     .build();
```

### Props Structs for Functions with Many Arguments

When a function takes more than 3 arguments, use a struct:

```rust
/// Arguments for creating a new user account.
///
/// # Why This Exists
/// User creation requires multiple pieces of data that are logically
/// grouped but distinct. Using a struct:
/// - Prevents argument order bugs
/// - Allows adding optional fields without changing function signature
/// - Self-documents what data is required
#[derive(Debug, Clone)]
pub struct CreateUserArgs<'a> {
    /// Unique identifier for the user (typically email or username).
    pub identifier: &'a str,
    
    /// User's display name shown in the UI.
    pub display_name: &'a str,
    
    /// Initial password - will be hashed before storage.
    pub password: &'a str,
    
    /// User's primary role determining initial permissions.
    pub role: UserRole,
    
    /// Optional metadata for the user account.
    pub metadata: Option<&'a UserMetadata>,
    
    /// Source of this user creation (web, api, import, etc).
    pub source: CreationSource,
}

/// Creates a new user account with validation and auditing.
///
/// # Why This Exists
/// User creation is a security-sensitive operation requiring:
/// - Password validation (complexity requirements)
/// - Identifier uniqueness checks
/// - Audit logging
/// - Initial permission setup
///
/// This function coordinates all these concerns.
pub fn create_user(args: CreateUserArgs<'_>) -> Result<User, UserCreationError> {
    // Validate identifier format
    if !is_valid_identifier(args.identifier) {
        return Err(UserCreationError::InvalidIdentifier);
    }
    
    // Check password complexity
    if !meets_password_policy(args.password) {
        return Err(UserCreationError::WeakPassword);
    }
    
    // Verify identifier isn't already in use
    if user_exists(args.identifier)? {
        return Err(UserCreationError::AlreadyExists);
    }
    
    // Hash password before storage (security: never store plaintext)
    let password_hash = hash_password(args.password)?;
    
    // Create user record
    let user = User {
        id: generate_user_id(),
        identifier: args.identifier.to_string(),
        display_name: args.display_name.to_string(),
        password_hash,
        role: args.role,
        metadata: args.metadata.map(|m| m.clone().into()),
        created_at: Utc::now(),
        created_via: args.source,
    };
    
    // Persist to database
    persist_user(&user)?;
    
    // Audit log the creation
    audit_log::record(AuditEvent::UserCreated {
        user_id: user.id,
        source: args.source,
        role: args.role,
    });
    
    Ok(user)
}

// Usage:
// let user = create_user(CreateUserArgs {
//     identifier: "alice@example.com",
//     display_name: "Alice Smith",
//     password: &hashed_pw,
//     role: UserRole::Standard,
//     metadata: None,
//     source: CreationSource::WebRegistration,
// })?;
```

### Lifetime Management in Props Structs

```rust
/// Arguments for validating a document against a schema.
///
/// # Why This Exists
/// Document validation needs both the document being validated
/// and the schema to validate against. Using references with
/// explicit lifetimes avoids unnecessary cloning of potentially
/// large documents and schemas.
pub struct ValidationArgs<'doc, 'schema> {
    /// The document to validate - borrowed to avoid cloning.
    pub document: &'doc Document,
    
    /// The schema to validate against - borrowed to avoid cloning.
    pub schema: &'schema ValidationSchema,
    
    /// Validation mode determining strictness.
    pub mode: ValidationMode,
    
    /// Optional context for error messages.
    pub context: Option<&'doc str>,
}

impl<'doc, 'schema> ValidationArgs<'doc, 'schema> {
    /// Creates args with standard validation mode.
    pub fn standard(document: &'doc Document, schema: &'schema ValidationSchema) -> Self {
        Self {
            document,
            schema,
            mode: ValidationMode::Standard,
            context: None,
        }
    }
}

/// Validates a document against a schema with detailed error reporting.
///
/// # Why This Exists
/// Schema validation is used throughout the application for:
/// - API request validation
/// - Configuration file validation
/// - Data import validation
///
/// This function provides consistent validation with detailed
/// error messages that help users fix their data.
pub fn validate_document<'doc, 'schema>(
    args: ValidationArgs<'doc, 'schema>
) -> Result<ValidationReport, ValidationError> {
    // Quick check: is the schema version compatible?
    if !args.schema.supports_version(args.document.version()) {
        return Err(ValidationError::IncompatibleSchema);
    }
    
    // Initialize validation state
    let mut report = ValidationReport::new(args.context);
    let mut validator = SchemaValidator::new(args.schema, args.mode);
    
    // Perform validation
    validator.validate(args.document, &mut report)?;
    
    // Return detailed report
    Ok(report)
}
```

---

## Whitespace and Grouping

### Blank Lines Before Control Flow

```rust
/// Processes incoming messages from the queue.
///
/// # Why This Exists
/// Messages arrive from multiple sources and need to be routed
/// to appropriate handlers based on their type. This function
/// is the entry point for all message processing.
async fn process_message(msg: Message) -> Result<(), ProcessingError> {
    // Extract message type for routing decision
    let msg_type = msg.header.message_type.clone();
    
    // Security: Verify message signature before processing
    // This prevents tampered messages from being processed
    if !verify_signature(&msg) {
        security_log::warn("Invalid message signature", &msg.header);
        return Err(ProcessingError::InvalidSignature);
    }
    
    // Route to appropriate handler based on message type
    match msg_type {
        MessageType::Order(order) => {
            // Validate order before processing
            if order.amount <= 0.0 {
                return Err(ProcessingError::InvalidAmount);
            }
            
            handle_order(order).await?
        }
        
        MessageType::InventoryUpdate(update) => {
            // Skip stale updates
            if update.timestamp < get_last_sync_time() {
                log::debug!("Skipping stale inventory update");
                return Ok(());
            }
            
            handle_inventory_update(update).await?
        }
        
        MessageType::SystemEvent(event) => {
            // System events are always handled, no validation needed
            handle_system_event(event).await?
        }
    }
    
    // Log successful processing for metrics
    metrics::increment_counter("messages_processed", &[("type", msg_type.as_str())]);
    
    Ok(())
}
```

### Grouping Related Operations

```rust
/// Initializes the application with all required components.
///
/// # Why This Exists
/// Application startup involves multiple phases: configuration loading,
/// dependency initialization, database connections, and server startup.
/// This function orchestrates the entire process with proper error handling
/// and resource cleanup on failure.
pub async fn initialize_app(config_path: &Path) -> Result<Application, StartupError> {
    // === Phase 1: Configuration Loading ===
    // Load configuration from file and environment
    let config = load_config(config_path)?;
    
    // Validate critical configuration values
    if config.database.url.is_empty() {
        return Err(StartupError::MissingDatabaseUrl);
    }
    
    // === Phase 2: Logging Setup ===
    // Configure logging before other components so they can log their initialization
    init_logging(&config.logging)?;
    log::info!("Starting application v{}", env!("CARGO_PKG_VERSION"));
    
    // === Phase 3: Database Connection ===
    // Connect to database before initializing other services that depend on it
    let db_pool = create_db_pool(&config.database).await?;
    
    // Run migrations to ensure schema is up to date
    run_migrations(&db_pool).await?;
    
    // === Phase 4: External Service Clients ===
    // Initialize clients for external services
    let http_client = create_http_client(&config.http)?;
    let cache_client = create_cache_client(&config.cache).await?;
    
    // === Phase 5: Application Services ===
    // Build the service layer with all dependencies
    let services = Arc::new(Services::new(
        db_pool.clone(),
        http_client,
        cache_client,
    ));
    
    // === Phase 6: Server Startup ===
    // Start the HTTP server with all routes configured
    let server = create_server(config.server.port, services.clone())?;
    
    // Signal handler for graceful shutdown
    let shutdown = create_shutdown_handler();
    
    Ok(Application {
        server,
        shutdown,
        services,
    })
}
```

---

## Error Handling

### Consistent Error Types

```rust
use thiserror::Error;

/// Errors that can occur during payment processing.
///
/// # Why This Exists
/// Payment processing involves multiple external systems (payment gateways,
/// fraud detection, accounting) each with their own error types. This
/// unified error type provides a consistent interface for callers while
/// preserving detailed error information for debugging.
#[derive(Error, Debug)]
pub enum PaymentError {
    #[error("Card declined: {reason}")]
    CardDeclined { reason: String },
    
    #[error("Card expired")]
    CardExpired,
    
    #[error("Insufficient funds: have {available}, need {required}")]
    InsufficientFunds { available: Decimal, required: Decimal },
    
    #[error("Fraud check failed: {code}")]
    FraudDetected { code: String },
    
    #[error("Gateway error: {0}")]
    GatewayError(#[from] GatewayError),
    
    #[error("Network error: {0}")]
    NetworkError(#[from] reqwest::Error),
    
    #[error("Database error: {0}")]
    DatabaseError(#[from] sqlx::Error),
}

/// Processes a payment with comprehensive error handling.
///
/// # Why This Exists
/// Payment processing must handle many failure modes gracefully:
/// - User errors (insufficient funds, expired card) - show to user
/// - System errors (network, database) - retry or fail gracefully
/// - Security errors (fraud) - log and escalate
async fn process_payment(payment: PaymentRequest) -> Result<Receipt, PaymentError> {
    // Validate payment amount
    if payment.amount <= 0.0 {
        return Err(PaymentError::InvalidAmount);
    }
    
    // Check fraud rules
    let fraud_check = fraud_service.check(&payment).await?;
    
    if !fraud_check.passed {
        // Security: Log fraud attempts for manual review
        security_log::alert("Fraud detected", &fraud_check);
        
        return Err(PaymentError::FraudDetected {
            code: fraud_check.code,
        });
    }
    
    // Process through payment gateway
    let gateway_result = gateway.charge(payment).await?;
    
    match gateway_result.status {
        GatewayStatus::Approved => {
            // Record transaction in database
            let receipt = persist_transaction(&gateway_result).await?;
            
            Ok(receipt)
        }
        
        GatewayStatus::Declined => {
            Err(PaymentError::CardDeclined {
                reason: gateway_result.decline_reason,
            })
        }
        
        GatewayStatus::Error => {
            // Log gateway errors for monitoring
            metrics::increment_counter("payment_gateway_errors");
            
            Err(PaymentError::GatewayError(gateway_result.error))
        }
    }
}
```

### Result Aliases for Common Patterns

```rust
/// Result type for database operations.
///
/// # Why This Exists
/// Database operations throughout the crate use the same error type.
/// This alias reduces repetition and makes function signatures clearer.
pub type DbResult<T> = Result<T, sqlx::Error>;

/// Result type for API operations.
///
/// # Why This Exists
/// API handlers all return JSON responses with consistent error formatting.
/// This alias ensures all endpoints use the same error type.
pub type ApiResult<T> = Result<Json<T>, ApiError>;

/// Finds a user by ID with consistent error handling.
///
/// # Why This Exists
/// This is the standard way to fetch a user by ID. It converts
/// database "not found" errors into domain-specific errors that
/// the API layer can convert to appropriate HTTP status codes.
pub async fn find_user_by_id(pool: &PgPool, id: UserId) -> DbResult<User> {
    let user = sqlx::query_as::<_, User>("SELECT * FROM users WHERE id = $1")
        .bind(id)
        .fetch_optional(pool)
        .await?;
    
    user.ok_or(sqlx::Error::RowNotFound)
}
```

---

## Module Structure

### File Organization

```rust
// src/services/
// ├── mod.rs          // Re-exports and module docs
// ├── payment.rs      // Payment processing
// ├── inventory.rs    // Inventory management
// └── user.rs         // User management

// src/services/mod.rs
//! Business logic services for the application.
//!
//! # Why This Module Exists
//! Services encapsulate the core business logic of the application.
//! They are independent of the transport layer (HTTP, gRPC, CLI) and
//! can be used from any entry point. Each service owns a specific
//! domain (payments, inventory, users) and provides a clean API
//! for that domain.
//!
//! # Architecture
//!
//! Services are designed to be:
//! - **Testable**: Business logic is pure and testable
//! - **Composable**: Services can depend on other services
//! - **Observable**: All operations are logged and measured
//! - **Transaction-safe**: Database operations use transactions
//!
//! # Usage Example
//!
//! ```rust
//! use my_app::services::{PaymentService, PaymentRequest};
//!
//! let service = PaymentService::new(db_pool, config)?;
//! let receipt = service.process_payment(request).await?;
//! ```

pub mod inventory;
pub mod payment;
pub mod user;

pub use inventory::{InventoryService, StockUpdate};
pub use payment::{PaymentReceipt, PaymentRequest, PaymentService};
pub use user::{UserService, CreateUserRequest};

use std::sync::Arc;

/// Container for all application services.
///
/// # Why This Exists
//! Services have interdependencies (e.g., payments depend on inventory
//! to check stock). This container manages those dependencies and
//! ensures each service is only created once.
pub struct Services {
    pub payment: Arc<PaymentService>,
    pub inventory: Arc<InventoryService>,
    pub user: Arc<UserService>,
}

impl Services {
    /// Creates all services with their dependencies wired correctly.
    pub fn new(db: DbPool, config: AppConfig) -> Result<Self, ServiceError> {
        let inventory = Arc::new(InventoryService::new(db.clone(), config.inventory)?);
        let user = Arc::new(UserService::new(db.clone(), config.user)?);
        
        // Payment service depends on inventory service
        let payment = Arc::new(PaymentService::new(
            db.clone(),
            inventory.clone(),
            config.payment,
        )?);
        
        Ok(Self {
            payment,
            inventory,
            user,
        })
    }
}
```

---

## Additional Patterns

### Iterator Chains with Early Returns

```rust
/// Filters and transforms raw records into validated domain objects.
///
/// # Why This Exists
/// Data ingestion pipelines receive raw records that need validation
/// before processing. This function filters invalid records and
/// transforms valid ones, returning early on any processing error.
fn process_records(raw_records: Vec<RawRecord>) -> Result<Vec<DomainRecord>, ProcessingError> {
    // Early return if no records to process
    if raw_records.is_empty() {
        return Ok(Vec::new());
    }
    
    // Pre-allocate with expected capacity
    let mut results = Vec::with_capacity(raw_records.len());
    
    // Process each record with early continue for invalid ones
    for record in raw_records {
        // Skip records with missing required fields
        if record.user_id.is_empty() {
            log::warn!("Skipping record with missing user_id: {:?}", record.id);
            continue;
        }
        
        // Skip records with invalid timestamps
        let Some(timestamp) = parse_timestamp(&record.timestamp) else {
            log::warn!("Skipping record with invalid timestamp: {:?}", record.timestamp);
            continue;
        };
        
        // Skip records that are too old
        if timestamp < get_cutoff_date() {
            log::debug!("Skipping stale record from {}", timestamp);
            continue;
        }
        
        // Transform and validate
        let domain_record = DomainRecord::try_from(record)?;
        
        // Additional validation
        if !domain_record.is_valid() {
            return Err(ProcessingError::ValidationFailed(domain_record.id));
        }
        
        results.push(domain_record);
    }
    
    // Log processing summary
    log::info!("Processed {} of {} records", results.len(), raw_records.len());
    
    Ok(results)
}
```

### Async/Await with Early Returns

```rust
/// Fetches user data with caching and fallback.
///
/// # Why This Exists
/// User data is frequently accessed but rarely changes. This function
/// implements a cache-first strategy with database fallback, minimizing
/// database load while ensuring fresh data is available.
async fn get_user_cached(user_id: UserId) -> Result<User, CacheError> {
    // Try cache first
    if let Some(cached) = cache.get(&user_id).await? {
        // Cache hit - validate the cached data isn't stale
        if !is_stale(&cached) {
            metrics::increment_counter("cache_hit");
            return Ok(cached);
        }
        
        // Stale data - log and continue to refresh
        log::debug!("Cache hit but data is stale for user {}", user_id);
    }
    
    // Cache miss or stale - fetch from database
    metrics::increment_counter("cache_miss");
    
    let user = db::fetch_user(user_id).await?;
    
    // Update cache for next time
    if let Err(e) = cache.set(user_id, &user).await {
        // Don't fail the request if cache update fails
        log::warn!("Failed to update cache: {}", e);
    }
    
    Ok(user)
}
```
