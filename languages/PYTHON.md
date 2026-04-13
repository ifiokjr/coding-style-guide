# Python Style Guide

A comprehensive guide for writing readable, maintainable Python code with emphasis on early returns, thoughtful whitespace, comprehensive documentation, and type hints.

## Table of Contents

1. [Documentation Standards](#documentation-standards)
2. [Early Returns and Control Flow](#early-returns-and-control-flow)
3. [Variable Declaration and Grouping](#variable-declaration-and-grouping)
4. [Dataclass Patterns](#dataclass-patterns)
5. [Function Arguments](#function-arguments)
6. [Whitespace and Grouping](#whitespace-and-grouping)
7. [Error Handling](#error-handling)
8. [Module Structure](#module-structure)
9. [Type Hints](#type-hints)

---

## Documentation Standards

### Every Item Gets Documentation

Every function, constant, class, and module must have docstrings explaining:
- **What** it does
- **Why** it exists
- **How** to use it (for public APIs)

Use Google-style docstrings for consistency.

```python
# Maximum number of retry attempts for failed operations.
#
# Why This Exists:
#     Network operations can fail transiently due to temporary issues
#     like connection resets or rate limiting. This constant provides
#     a sensible default that balances reliability with performance.
#
# When to Change:
#     - Increase for critical operations where success is mandatory
#     - Decrease for time-sensitive operations where failure is acceptable
MAX_RETRIES: int = 3


def normalize_user_data(raw: dict[str, Any]) -> User:
    """Transforms raw API responses into strongly-typed domain objects.

    Why This Exists:
        External APIs return loosely-typed JSON that doesn't match our
        internal domain models. This function centralizes the mapping
        logic, ensuring all consumers receive consistent, validated data.

    Use Cases:
        - Normalizing data from third-party REST APIs
        - Transforming GraphQL responses to domain types
        - Validating and sanitizing user-submitted data

    Args:
        raw: The untyped response from the API.

    Returns:
        A validated, strongly-typed User object.

    Raises:
        ValidationError: When the data doesn't match expected schema.

    Example:
        >>> raw_response = fetch_user_api(123)
        >>> user = normalize_user_data(raw_response)
        >>> print(user.display_name)  # Type-safe access
        'Alice Smith'
    """
    ...
```

### Module-Level Documentation

Every module file should have comprehensive documentation at the top:

```python
"""Sensor data processing and normalization.

Why This Module Exists:
    IoT deployments use sensors from multiple vendors with different
    data formats. This module provides a unified interface for:
    - Ingesting raw sensor readings
    - Normalizing to a common schema
    - Validating data quality
    - Transforming for downstream consumers

Key Components:
    normalize_sensor_data: Converts vendor-specific formats to standard schema
    SensorValidator: Validates data quality and completeness
    DataTransformer: Applies business logic transformations

Quick Start:
    >>> from myapp.sensor import normalize_sensor_data, SensorConfig
    >>>
    >>> config = SensorConfig(vendor="acme", max_temp=100.0)
    >>> data = normalize_sensor_data(raw_data, config)
"""

from __future__ import annotations

from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Iterator

# ... module contents
```

### Internal Documentation

Even non-exported items deserve documentation:

```python
def _parse_vendor_timestamp(raw: str, vendor: str) -> int:
    """Parses vendor-specific timestamp formats into Unix timestamps.

    Why This Exists:
        Different vendors use different timestamp precisions (seconds,
        millis, micros) and formats (Unix, ISO 8601, proprietary).
        This centralizes the parsing logic so normalization functions
        don't need to handle format detection.

    Supported Formats:
        - Unix seconds (10 digits)
        - Unix milliseconds (13 digits)
        - Unix microseconds (16 digits)
        - ISO 8601 strings

    Args:
        raw: The timestamp string from the vendor.
        vendor: The vendor identifier for format-specific parsing.

    Returns:
        The Unix timestamp in milliseconds.
    """
    ...
```

---

## Early Returns and Control Flow

### The Rule: Exit Early, Stay Shallow

Deeply nested code is an orange flag. Prefer early returns to nesting.

**Avoid - Deep Nesting:**

```python
# Don't do this - hard to follow the logic
def process_user_request(request: Request) -> Response:
    if request.user is not None:
        if request.user.is_active:
            if request.user.has_permission("write"):
                if request.data.is_valid():
                    result = write_to_database(request.data)
                    if result.rows_affected > 0:
                        return Response.success(result.id)
                    else:
                        raise RequestError("No rows affected")
                else:
                    raise RequestError("Invalid data")
            else:
                raise RequestError("No permission")
        else:
            raise RequestError("User inactive")
    else:
        raise RequestError("No user")
```

**Prefer - Early Returns:**

```python
def process_user_request(request: Request) -> Response:
    """Processes a user request after validating all preconditions.

    Why This Exists:
        User requests must pass multiple validation layers
        (authentication, authorization, data validation) before
        processing. This function centralizes the validation logic
        and returns early on any failure, making the "happy path"
        clear and the error cases explicit.

    Args:
        request: The user request to process.

    Returns:
        The response after successful processing.

    Raises:
        RequestError: When any validation fails.
    """
    # Extract user or fail fast
    if request.user is None:
        raise RequestError("No user", "Request requires an authenticated user")

    # Validate user state
    if not request.user.is_active:
        raise RequestError("User inactive", "User account is disabled")

    # Check permissions
    if not request.user.has_permission("write"):
        raise RequestError(
            "No permission", "User lacks required permission: write"
        )

    # Validate request data
    if not request.data.is_valid():
        raise RequestError("Invalid data", "Request data failed validation")

    # Now we're on the happy path - all validations passed
    result = write_to_database(request.data)

    if result.rows_affected == 0:
        raise RequestError("No rows affected", "Database operation had no effect")

    return Response.success(result.id)
```

### Pattern Matching (Python 3.10+)

```python
from enum import Enum, auto


class PaymentMethod(Enum):
    CREDIT_CARD = auto()
    BANK_TRANSFER = auto()
    CRYPTO = auto()


def process_payment(payment: Payment) -> Receipt:
    """Routes a payment to the appropriate processor based on method.

    Why This Exists:
        Payment processing varies significantly by method (card, bank
        transfer, crypto). Using match with early returns keeps each
        payment path self-contained and prevents deep nesting.

    Args:
        payment: The payment to process.

    Returns:
        The payment receipt after processing.
    """
    match payment.method:
        case PaymentMethod.CREDIT_CARD:
            card = payment.details

            # Validate card first
            if card.is_expired():
                raise PaymentError("Card expired", "Credit card has expired")

            if not card.has_valid_cvv():
                raise PaymentError("Invalid CVV", "Card security code is invalid")

            # Process the validated card
            return process_card_payment(card, payment.amount)

        case PaymentMethod.BANK_TRANSFER:
            details = payment.details

            # Validate bank details
            if not details.account_verified:
                raise PaymentError(
                    "Unverified account", "Bank account is not verified"
                )

            # Bank transfers take longer - create pending receipt
            return create_pending_receipt(details, payment.amount)

        case PaymentMethod.CRYPTO:
            wallet = payment.details

            # Validate wallet has sufficient balance
            balance = get_wallet_balance(wallet.address)

            if balance < payment.amount:
                raise PaymentError(
                    "Insufficient funds", "Wallet has insufficient balance"
                )

            # Crypto is irreversible - double-check amount
            if payment.amount > CRYPTO_MAX_AMOUNT:
                raise PaymentError(
                    "Amount too large", "Crypto amount exceeds maximum"
                )

            return process_crypto_payment(wallet, payment.amount)
```

### Iteration with Early Continue

```python
def process_log_entries(entries: list[LogEntry]) -> ProcessingResult:
    """Processes log entries, skipping malformed entries.

    Why This Exists:
        Log files often contain corrupted or incomplete entries due to
        crashes or disk issues. Rather than failing the entire batch,
        we skip problematic entries and report them separately.

    Args:
        entries: The raw log entries to process.

    Returns:
        The processing result with valid and failed entries.
    """
    valid_entries: list[NormalizedEntry] = []
    failed_entries: list[FailedEntry] = []

    for entry in entries:
        # Skip entries without timestamps - can't order them
        if entry.timestamp is None:
            failed_entries.append((entry, "Missing timestamp"))
            continue

        # Skip entries with invalid log levels
        if not is_valid_log_level(entry.level):
            failed_entries.append((entry, "Invalid log level"))
            continue

        # Skip entries where message is empty
        if not entry.message or not entry.message.strip():
            failed_entries.append((entry, "Empty message"))
            continue

        # Entry passed all validations - process it
        normalized = normalize_entry(entry)
        valid_entries.append(normalized)

    return ProcessingResult(
        processed=len(valid_entries),
        failed=len(failed_entries),
        valid_entries=valid_entries,
        failed_entries=failed_entries,
    )
```

---

## Variable Declaration and Grouping

### Variables at the Top

Declare variables at the start of functions when possible:

```python
def create_http_client(config: AppConfig) -> Client:
    """Configures the HTTP client with retry and timeout policies.

    Why This Exists:
        HTTP clients need consistent configuration across the application
        for reliability (retries) and resource management (timeouts).
        This function centralizes that configuration.

    Args:
        config: The application configuration.

    Returns:
        Configured HTTP client.
    """
    # Group 1: Configuration extraction
    # These values control how the client behaves
    timeout_seconds = config.http.timeout_secs
    retry_attempts = config.http.retry_attempts
    pool_size = config.http.connection_pool_size

    # Security: Force HTTPS in production environments
    # This prevents accidental cleartext transmission of sensitive data
    enforce_https = config.environment.is_production()

    # Group 2: Client configuration
    # Building the client happens in phases
    client = httpx.Client(
        timeout=timeout_seconds,
        limits=httpx.Limits(max_connections=pool_size),
    )

    if enforce_https:
        # Add middleware to reject HTTP in production
        client.event_hooks["request"].append(_enforce_https)

    # Group 3: Transport configuration
    # Configure retries for resilience
    transport = httpx.HTTPTransport(retries=retry_attempts)
    client._transport = transport

    return client
```

### Complex Variable Declarations

Long declarations get breathing room:

```python
async def fetch_user_data(user_id: UserId) -> MergedUserData:
    """Fetches user data from multiple sources and merges the results.

    Why This Exists:
        User data is distributed across multiple services (profile service,
        preference service, authorization service). This function coordinates
        the parallel fetching and merging of that data.

    Args:
        user_id: The ID of the user to fetch data for.

    Returns:
        Merged user data from all services.
    """
    # Configuration for each service endpoint
    # These URLs come from environment configuration
    profile_url = f"{config.profile_service_url}/users/{user_id}"
    prefs_url = f"{config.prefs_service_url}/users/{user_id}/preferences"
    auth_url = f"{config.auth_service_url}/users/{user_id}/roles"

    # Request configuration shared across all requests
    # Using a common timeout and headers for consistency
    headers = {
        "X-Request-ID": generate_request_id(),
        "Accept": "application/json",
    }
    timeout = httpx.Timeout(5.0, connect=2.0)

    # Spawn concurrent fetches
    # Each fetch is independent, so we can run them in parallel
    async with httpx.AsyncClient(headers=headers, timeout=timeout) as client:
        profile_task = fetch_profile_data(client, profile_url)
        prefs_task = fetch_preferences(client, prefs_url)
        auth_task = fetch_authorization(client, auth_url)

        # Await all results
        # If any fails, we fail fast - partial data is not useful
        profile, prefs, roles = await asyncio.gather(
            profile_task, prefs_task, auth_task
        )

    # Merge into unified view
    # This is the final result that callers actually want
    merged = MergedUserData.from_parts(profile, prefs, roles)

    return merged
```

---

## Dataclass Patterns

### Using dataclasses for Complex Objects

When classes hold data with many optional fields, use dataclasses:

```python
from dataclasses import dataclass, field
from typing import Callable


@dataclass(frozen=True)
class JobProcessorConfig:
    """Configuration for the background job processor.

    Why This Exists:
        Background jobs need tunable parameters for different workloads.
        Using a dataclass with defaults allows callers to specify only
        what they need while providing sensible defaults for everything else.

    Attributes:
        max_concurrent: Maximum number of concurrent jobs to process.
            Higher values increase throughput but use more resources.
        poll_interval: How long to wait between polling for new jobs (seconds).
        max_retries: Maximum retry attempts for failed jobs.
        prioritize: Whether to process jobs in priority order.
        on_failure: Optional callback called when a job fails permanently.
        serializer: Optional custom serializer for job payloads.
    """

    max_concurrent: int = 4
    poll_interval: float = 5.0
    max_retries: int = 3
    prioritize: bool = True
    on_failure: Callable[[Job, JobError], None] | None = None
    serializer: JobSerializer | None = field(default=None, repr=False)


# Usage:
# config = JobProcessorConfig(max_concurrent=8, poll_interval=1.0)
# Other fields use defaults
```

### Dataclasses for Function Arguments

When a function takes more than 3 arguments, use a dataclass:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class CreateUserArgs:
    """Arguments for creating a new user account.

    Why This Exists:
        User creation requires multiple pieces of data that are logically
        grouped but distinct. Using a dataclass:
        - Prevents argument order bugs
        - Allows adding optional fields without changing function signature
        - Self-documents what data is required
        - Can be frozen to prevent accidental mutation

    Attributes:
        identifier: Unique identifier for the user (typically email).
        display_name: User's display name shown in the UI.
        password: Initial password - will be hashed before storage.
        role: User's primary role determining initial permissions.
        metadata: Optional metadata for the user account.
        source: Source of this user creation (web, api, import, etc).
    """

    identifier: str
    display_name: str
    password: str
    role: UserRole
    metadata: dict[str, Any] | None = None
    source: CreationSource = CreationSource.WEB


def create_user(args: CreateUserArgs) -> User:
    """Creates a new user account with validation and auditing.

    Why This Exists:
        User creation is a security-sensitive operation requiring:
        - Password validation (complexity requirements)
        - Identifier uniqueness checks
        - Audit logging
        - Initial permission setup

        This function coordinates all these concerns.

    Args:
        args: The user creation arguments.

    Returns:
        The created user.

    Raises:
        UserCreationError: When validation fails.
    """
    # Validate identifier format
    if not is_valid_identifier(args.identifier):
        raise UserCreationError("Invalid identifier")

    # Check password complexity
    if not meets_password_policy(args.password):
        raise UserCreationError("Weak password")

    # Verify identifier isn't already in use
    if user_exists(args.identifier):
        raise UserCreationError("Already exists")

    # Hash password before storage (security: never store plaintext)
    password_hash = hash_password(args.password)

    # Create user record
    user = User(
        id=generate_user_id(),
        identifier=args.identifier,
        display_name=args.display_name,
        password_hash=password_hash,
        role=args.role,
        metadata=args.metadata or {},
        created_at=datetime.now(timezone.utc),
        created_via=args.source,
    )

    # Persist to database
    persist_user(user)

    # Audit log the creation
    audit_log.record(
        AuditEvent.USER_CREATED,
        user_id=user.id,
        source=args.source,
        role=args.role,
    )

    return user


# Usage:
# user = create_user(CreateUserArgs(
#     identifier="alice@example.com",
#     display_name="Alice Smith",
#     password=hashed_pw,
#     role=UserRole.STANDARD,
#     source=CreationSource.WEB_REGISTRATION,
# ))
```

### Builder Pattern Alternative

For complex objects where you want a builder-style API:

```python
from dataclasses import dataclass, field
from typing import Self


@dataclass(frozen=True)
class QueryBuilder:
    """Builder for constructing database queries.

    Why This Exists:
        Complex database queries have many optional components
        (filters, joins, ordering, pagination). Using a builder
        pattern allows constructing queries incrementally.

    Example:
        >>> query = (
        ...     QueryBuilder()
        ...     .select("users")
        ...     .where("active", "=", True)
        ...     .order_by("created_at", "desc")
        ...     .limit(10)
        ...     .build()
        ... )
    """

    table: str = ""
    columns: list[str] = field(default_factory=list)
    where_clauses: list[tuple[str, str, Any]] = field(default_factory=list)
    order_by_columns: list[tuple[str, str]] = field(default_factory=list)
    limit_value: int | None = None
    offset_value: int | None = None

    def select(self, table: str, columns: list[str] | None = None) -> Self:
        """Sets the table and optional column selection."""
        return self.__class__(
            **{**self.__dict__, "table": table, "columns": columns or ["*"]}
        )

    def where(self, column: str, op: str, value: Any) -> Self:
        """Adds a WHERE clause condition."""
        new_clauses = [*self.where_clauses, (column, op, value)]
        return self.__class__(**{**self.__dict__, "where_clauses": new_clauses})

    def order_by(self, column: str, direction: str = "asc") -> Self:
        """Adds an ORDER BY clause."""
        new_order = [*self.order_by_columns, (column, direction)]
        return self.__class__(**{**self.__dict__, "order_by_columns": new_order})

    def limit(self, count: int) -> Self:
        """Sets the LIMIT value."""
        return self.__class__(**{**self.__dict__, "limit_value": count})

    def offset(self, count: int) -> Self:
        """Sets the OFFSET value."""
        return self.__class__(**{**self.__dict__, "offset_value": count})

    def build(self) -> str:
        """Builds the final SQL query string."""
        # Implementation...
        pass
```

---

## Whitespace and Grouping

### Blank Lines Before Control Flow

```python
async def process_message(msg: Message) -> None:
    """Processes incoming messages from the queue.

    Why This Exists:
        Messages arrive from multiple sources and need to be routed
        to appropriate handlers based on their type. This function
        is the entry point for all message processing.

    Args:
        msg: The message to process.
    """
    # Extract message type for routing decision
    msg_type = msg.header.message_type

    # Security: Verify message signature before processing
    # This prevents tampered messages from being processed
    if not verify_signature(msg):
        security_log.warn("Invalid message signature", msg.header)
        raise ProcessingError("Invalid signature")

    # Route to appropriate handler based on message type
    match msg_type:
        case MessageType.ORDER:
            order = msg.payload

            # Validate order before processing
            if order.amount <= 0:
                raise ProcessingError("Invalid amount")

            await handle_order(order)

        case MessageType.INVENTORY_UPDATE:
            update = msg.payload

            # Skip stale updates
            if update.timestamp < get_last_sync_time():
                log.debug("Skipping stale inventory update")
                return

            await handle_inventory_update(update)

        case MessageType.SYSTEM_EVENT:
            # System events are always handled, no validation needed
            await handle_system_event(msg.payload)

    # Log successful processing for metrics
    metrics.increment("messages_processed", {"type": msg_type.value})
```

### Grouping Related Operations

```python
async def initialize_app(config_path: Path) -> Application:
    """Initializes the application with all required components.

    Why This Exists:
        Application startup involves multiple phases: configuration loading,
        dependency initialization, database connections, and server startup.
        This function orchestrates the entire process with proper error handling
        and resource cleanup on failure.

    Args:
        config_path: Path to the configuration file.

    Returns:
        Initialized application instance.
    """
    # === Phase 1: Configuration Loading ===
    # Load configuration from file and environment
    config = load_config(config_path)

    # Validate critical configuration values
    if not config.database.url:
        raise StartupError("Missing database URL")

    # === Phase 2: Logging Setup ===
    # Configure logging before other components so they can log their initialization
    init_logging(config.logging)
    log.info(f"Starting application v{__version__}")

    # === Phase 3: Database Connection ===
    # Connect to database before initializing other services that depend on it
    db_pool = await create_db_pool(config.database)

    # Run migrations to ensure schema is up to date
    await run_migrations(db_pool)

    # === Phase 4: External Service Clients ===
    # Initialize clients for external services
    http_client = create_http_client(config.http)
    cache_client = await create_cache_client(config.cache)

    # === Phase 5: Application Services ===
    # Build the service layer with all dependencies
    services = Services(db_pool, http_client, cache_client)

    # === Phase 6: Server Startup ===
    # Start the HTTP server with all routes configured
    server = create_server(config.server.port, services)

    # Signal handler for graceful shutdown
    shutdown = create_shutdown_handler()

    return Application(server, shutdown, services)
```

---

## Error Handling

### Custom Exception Hierarchies

```python
class PaymentError(Exception):
    """Base exception for payment processing errors.

    Why This Exists:
        Payment processing involves multiple external systems (payment gateways,
        fraud detection, accounting) each with their own error types. This
        unified exception hierarchy provides a consistent interface for callers
        while preserving detailed error information for debugging.
    """

    def __init__(self, code: str, message: str, details: dict[str, Any] | None = None):
        super().__init__(message)
        self.code = code
        self.details = details or {}

    @property
    def is_user_error(self) -> bool:
        """Check if this is a user-correctable error.

        Why This Exists:
            Some payment errors (e.g., expired card) can be fixed by the user,
            while others (e.g., gateway downtime) require system intervention.
            This property helps the UI decide whether to show a retry button
            or an error message.
        """
        return self.code in {
            "CARD_EXPIRED",
            "CARD_DECLINED",
            "INVALID_CVV",
            "INSUFFICIENT_FUNDS",
        }


class CardExpiredError(PaymentError):
    """Card has expired."""

    def __init__(self):
        super().__init__("CARD_EXPIRED", "Credit card has expired")


class FraudDetectedError(PaymentError):
    """Payment blocked by fraud detection."""

    def __init__(self, code: str):
        super().__init__("FRAUD_DETECTED", "Payment blocked by fraud detection", {"code": code})


async def process_payment(payment: Payment) -> Receipt:
    """Processes a payment with comprehensive error handling.

    Why This Exists:
        Payment processing must handle many failure modes gracefully:
        - User errors (insufficient funds, expired card) - show to user
        - System errors (network, database) - retry or fail gracefully
        - Security errors (fraud) - log and escalate

    Args:
        payment: The payment to process.

    Returns:
        The payment receipt.

    Raises:
        PaymentError: When payment fails.
    """
    # Validate payment amount
    if payment.amount <= 0:
        raise PaymentError("INVALID_AMOUNT", "Payment amount must be positive")

    # Check fraud rules
    fraud_check = await fraud_service.check(payment)

    if not fraud_check.passed:
        # Security: Log fraud attempts for manual review
        security_log.alert("Fraud detected", fraud_check)
        raise FraudDetectedError(fraud_check.code)

    # Process through payment gateway
    gateway_result = await gateway.charge(payment)

    match gateway_result.status:
        case GatewayStatus.APPROVED:
            # Record transaction in database
            receipt = await persist_transaction(gateway_result)
            return receipt

        case GatewayStatus.DECLINED:
            raise PaymentError(
                "CARD_DECLINED", f"Card declined: {gateway_result.decline_reason}"
            )

        case GatewayStatus.ERROR:
            # Log gateway errors for monitoring
            metrics.increment("payment_gateway_errors")
            raise PaymentError("GATEWAY_ERROR", "Payment gateway error")
```

### Result Types for Explicit Error Handling

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")
E = TypeVar("E")


@dataclass(frozen=True)
class Ok(Generic[T]):
    """Represents a successful result."""

    value: T


@dataclass(frozen=True)
class Err(Generic[E]):
    """Represents a failed result."""

    error: E


# Type alias for Result
type Result[T, E] = Ok[T] | Err[E]


def validate_user_input(raw: Any) -> Result[UserInput, ValidationErrors]:
    """Validates user input before processing.

    Why This Exists:
        Input validation returns multiple possible error types.
        Using a Result type forces callers to handle all error cases.

    Args:
        raw: The raw user input.

    Returns:
        Result with validated data or validation errors.
    """
    errors: ValidationErrors = {}

    # Validate email
    if not isinstance(raw, dict):
        return Err({"_general": "Input must be an object"})

    email = raw.get("email")
    age = raw.get("age")

    if not isinstance(email, str):
        errors["email"] = "Email is required"
    elif not is_valid_email(email):
        errors["email"] = "Invalid email format"

    # Validate age
    if not isinstance(age, int):
        errors["age"] = "Age is required"
    elif age < 0 or age > 150:
        errors["age"] = "Age must be between 0 and 150"

    # Return early if any validation failed
    if errors:
        return Err(errors)

    # All validation passed
    return Ok(UserInput(email=email, age=age))


# Usage forces error handling:
# result = validate_user_input(raw_input)
# if isinstance(result, Err):
#     # Handle error case
#     return
# # Type checker knows result.value is valid here
# process_user_input(result.value)
```

---

## Type Hints

### Comprehensive Type Annotations

```python
from typing import TypedDict, NotRequired, Required


class UserResponse(TypedDict):
    """Response structure from the user API.

    Why This Exists:
        Using TypedDict for API responses ensures compile-time checking
        of response shapes. This catches typos in field names and
        ensures required fields are present.
    """

    id: Required[str]
    email: Required[str]
    display_name: Required[str]
    role: Required[str]
    preferences: NotRequired[dict[str, Any]]
    last_login: NotRequired[str | None]


def parse_user_response(data: dict[str, Any]) -> User:
    """Parses API response into domain User object.

    Why This Exists:
        API responses are untyped dictionaries. This function validates
        the response structure and converts to a typed User object,
        catching errors early.

    Args:
        data: The raw API response.

    Returns:
        A validated User object.

    Raises:
        ValidationError: When the response doesn't match expected structure.
    """
    # Validate required fields
    if not (user_id := data.get("id")):
        raise ValidationError("Missing required field: id")

    if not (email := data.get("email")):
        raise ValidationError("Missing required field: email")

    if not (display_name := data.get("display_name")):
        raise ValidationError("Missing required field: display_name")

    if not (role := data.get("role")):
        raise ValidationError("Missing required field: role")

    # Optional fields with defaults
    preferences = data.get("preferences", {})
    last_login = parse_iso_datetime(data.get("last_login"))

    return User(
        id=user_id,
        email=email,
        display_name=display_name,
        role=UserRole(role),
        preferences=preferences,
        last_login=last_login,
    )
```

### Protocol for Interface Definition

```python
from typing import Protocol, runtime_checkable


@runtime_checkable
class Serializer(Protocol):
    """Protocol for serialization implementations.

    Why This Exists:
        Using Protocols instead of ABCs allows structural subtyping.
        Any class that implements these methods is considered a
        Serializer, without needing to inherit from a base class.
    """

    def serialize(self, obj: Any) -> bytes:
        """Serialize an object to bytes."""
        ...

    def deserialize(self, data: bytes, cls: type[T]) -> T:
        """Deserialize bytes to an object of the given type."""
        ...


class JsonSerializer:
    """JSON implementation of the Serializer protocol."""

    def serialize(self, obj: Any) -> bytes:
        return json.dumps(obj).encode()

    def deserialize(self, data: bytes, cls: type[T]) -> T:
        raw = json.loads(data)
        return cls(**raw)  # Simplified - real implementation would validate


# Any class that matches the protocol works
serializer: Serializer = JsonSerializer()
```

---

## Module Structure

### File Organization

```python
# src/services/
# ├── __init__.py     # Re-exports and module docs
# ├── payment.py      # Payment processing
# ├── inventory.py    # Inventory management
# └── user.py         # User management

# src/services/__init__.py
"""Business logic services for the application.

Why This Module Exists:
    Services encapsulate the core business logic of the application.
    They are independent of the transport layer (HTTP, gRPC, CLI) and
    can be used from any entry point. Each service owns a specific
    domain (payments, inventory, users) and provides a clean API
    for that domain.

Architecture:
    Services are designed to be:
    - **Testable**: Business logic is pure and testable
    - **Composable**: Services can depend on other services
    - **Observable**: All operations are logged and measured
    - **Transaction-safe**: Database operations use transactions

Usage Example:
    >>> from myapp.services import PaymentService, PaymentRequest
    >>>
    >>> service = PaymentService(db_pool, config)
    >>> receipt = await service.process_payment(request)
"""

from .inventory import InventoryService, StockUpdate
from .payment import PaymentService, PaymentReceipt, PaymentRequest
from .user import CreateUserRequest, UserService

__all__ = [
    "CreateUserRequest",
    "InventoryService",
    "PaymentReceipt",
    "PaymentRequest",
    "PaymentService",
    "StockUpdate",
    "UserService",
]


class Services:
    """Container for all application services.

    Why This Exists:
        Services have interdependencies (e.g., payments depend on inventory
        to check stock). This container manages those dependencies and
        ensures each service is only created once.
    """

    def __init__(self, db: DbPool, config: AppConfig) -> None:
        self.inventory = InventoryService(db, config.inventory)
        self.user = UserService(db, config.user)

        # Payment service depends on inventory service
        self.payment = PaymentService(db, self.inventory, config.payment)

    async def shutdown(self) -> None:
        """Gracefully shuts down all services."""
        await self.inventory.close()
        await self.payment.close()
        await self.user.close()
```

### Imports Organization

```python
"""Example module showing import organization."""

from __future__ import annotations

# Standard library imports
import asyncio
import json
from dataclasses import dataclass, field
from datetime import datetime, timezone
from pathlib import Path
from typing import TYPE_CHECKING, Any, Self

# Third-party imports
import httpx
from pydantic import BaseModel, Field

# Local imports
from myapp.config import AppConfig
from myapp.errors import ServiceError, ValidationError
from myapp.utils import generate_request_id

if TYPE_CHECKING:
    # Imports only needed for type checking, not at runtime
    from collections.abc import AsyncIterator, Callable
    from myapp.models import User, Order

    # Avoid circular imports
    from myapp.database import DbPool
```

---

## Additional Patterns

### Context Managers for Resource Management

```python
from contextlib import asynccontextmanager
from typing import AsyncGenerator


@asynccontextmanager
async def database_transaction(pool: DbPool) -> AsyncGenerator[Connection, None]:
    """Context manager for database transactions.

    Why This Exists:
        Transactions must be committed on success or rolled back on failure.
        Using a context manager ensures this happens automatically, even if
        an exception is raised.

    Args:
        pool: The database connection pool.

    Yields:
        A database connection with an active transaction.

    Example:
        >>> async with database_transaction(pool) as conn:
        ...     await conn.execute("INSERT INTO users ...")
        ...     await conn.execute("INSERT INTO profiles ...")
        ... # Automatically commits or rolls back
    """
    conn = await pool.acquire()
    tx = await conn.begin()

    try:
        yield conn
        await tx.commit()
    except Exception:
        await tx.rollback()
        raise
    finally:
        await pool.release(conn)


# Usage:
# async with database_transaction(pool) as conn:
#     await conn.execute("INSERT INTO users ...")
#     await conn.execute("INSERT INTO profiles ...")
```

### Generator Functions for Streaming

```python
from collections.abc import Iterator


def stream_large_dataset(query: Query) -> Iterator[Record]:
    """Streams large datasets without loading everything into memory.

    Why This Exists:
        Loading large datasets into memory can cause OOM errors.
        Using a generator allows processing records one at a time,
        keeping memory usage constant regardless of dataset size.

    Args:
        query: The database query to execute.

    Yields:
        Individual records from the query.

    Example:
        >>> for record in stream_large_dataset(query):
        ...     process_record(record)  # Memory stays constant
    """
    # Validate query before execution
    if query.limit is None:
        log.warning("Streaming query without LIMIT - potential memory issue")

    cursor = execute_query(query)

    try:
        for row in cursor:
            # Skip rows that don't need processing
            if not should_process(row):
                continue

            # Transform before yielding
            record = transform_row(row)

            # Additional validation
            if not record.is_valid():
                log.warning(f"Skipping invalid record: {record.id}")
                continue

            yield record
    finally:
        cursor.close()
```

### Decorators for Cross-Cutting Concerns

```python
from functools import wraps
from typing import Callable, ParamSpec, TypeVar

P = ParamSpec("P")
T = TypeVar("T")


def retry_on_error(
    max_retries: int = 3,
    exceptions: tuple[type[Exception], ...] = (Exception,),
) -> Callable[[Callable[P, T]], Callable[P, T]]:
    """Decorator that retries function calls on specific exceptions.

    Why This Exists:
        Network operations and external service calls often fail
        transiently. Retrying with exponential backoff improves
        reliability without manual try/except blocks.

    Args:
        max_retries: Maximum number of retry attempts.
        exceptions: Tuple of exception types to catch and retry.

    Returns:
        Decorated function with retry logic.

    Example:
        >>> @retry_on_error(max_retries=3, exceptions=(ConnectionError,))
        ... def fetch_data(url: str) -> dict:
        ...     return requests.get(url).json()
    """

    def decorator(func: Callable[P, T]) -> Callable[P, T]:
        @wraps(func)
        def wrapper(*args: P.args, **kwargs: P.kwargs) -> T:
            last_exception: Exception | None = None

            for attempt in range(max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e

                    if attempt < max_retries:
                        wait = 2**attempt  # Exponential backoff
                        log.warning(f"Attempt {attempt + 1} failed, retrying in {wait}s")
                        time.sleep(wait)

            raise last_exception or RuntimeError("All retries exhausted")

        return wrapper

    return decorator
```
