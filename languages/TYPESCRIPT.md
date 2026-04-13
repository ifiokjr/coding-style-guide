# TypeScript Style Guide

A comprehensive guide for writing readable, maintainable TypeScript code with emphasis on early returns, thoughtful whitespace, comprehensive documentation, and strict type safety.

## Table of Contents

1. [Documentation Standards](#documentation-standards)
2. [Early Returns and Control Flow](#early-returns-and-control-flow)
3. [Variable Declaration and Grouping](#variable-declaration-and-grouping)
4. [Interface and Type Patterns](#interface-and-type-patterns)
5. [Function Arguments](#function-arguments)
6. [Whitespace and Grouping](#whitespace-and-grouping)
7. [Error Handling](#error-handling)
8. [Module Structure](#module-structure)
9. [Async Patterns](#async-patterns)

---

## Documentation Standards

### Every Item Gets Documentation

Every function, constant, interface, class, and module must have JSDoc/TSDoc explaining:
- **What** it does
- **Why** it exists
- **How** to use it (for public APIs)

```typescript
/**
 * Maximum number of retry attempts for failed HTTP requests.
 *
 * @why
 * Network operations can fail transiently due to temporary issues
 * like connection resets or rate limiting. This constant provides
 * a sensible default that balances reliability with performance.
 *
 * @whenToChange
 * - Increase for critical operations where success is mandatory
 * - Decrease for time-sensitive operations where failure is acceptable
 */
const MAX_RETRIES = 3;

/**
 * Transforms raw API responses into strongly-typed domain objects.
 *
 * @why
 * External APIs return loosely-typed JSON that doesn't match our
 * internal domain models. This function centralizes the mapping
 * logic, ensuring all consumers receive consistent, validated data.
 *
 * @useCases
 * - Normalizing data from third-party REST APIs
 * - Transforming GraphQL responses to domain types
 * - Validating and sanitizing user-submitted data
 *
 * @example
 * ```typescript
 * const rawResponse = await fetch('/api/users/123');
 * const userData = await rawResponse.json();
 *
 * const user = normalizeUserData(userData);
 * // User is now fully typed with User interface
 * console.log(user.displayName); // Type-safe access
 * ```
 *
 * @param raw - The untyped response from the API
 * @returns A validated, strongly-typed User object
 * @throws {ValidationError} When the data doesn't match expected schema
 */
export function normalizeUserData(raw: unknown): User {
  // Implementation
}
```

### Module-Level Documentation

Every module file should have comprehensive documentation at the top:

```typescript
/**
 * @module sensor-data
 *
 * Sensor data processing and normalization.
 *
 * @why
 * IoT deployments use sensors from multiple vendors with different
 * data formats. This module provides a unified interface for:
 * - Ingesting raw sensor readings
 * - Normalizing to a common schema
 * - Validating data quality
 * - Transforming for downstream consumers
 *
 * @keyComponents
 * - {@link normalizeSensorData}: Converts vendor-specific formats to standard schema
 * - {@link SensorValidator}: Validates data quality and completeness
 * - {@link DataTransformer}: Applies business logic transformations
 *
 * @quickStart
 * ```typescript
 * import { normalizeSensorData, SensorConfig } from './sensor-data';
 *
 * const config = SensorConfig.create({
 *   vendor: 'acme',
 *   maxTemp: 100,
 * });
 *
 * const data = normalizeSensorData(rawData, config);
 * ```
 */

import { z } from 'zod';

// ... module contents
```

### Internal Documentation

Even non-exported items deserve documentation:

```typescript
/**
 * Parses vendor-specific timestamp formats into Unix timestamps.
 *
 * @why
 * Different vendors use different timestamp precisions (seconds, millis, micros)
 * and formats (Unix, ISO 8601, proprietary). This centralizes the parsing
 * logic so normalization functions don't need to handle format detection.
 *
 * @supportedFormats
 * - Unix seconds (10 digits)
 * - Unix milliseconds (13 digits)
 * - Unix microseconds (16 digits)
 * - ISO 8601 strings
 *
 * @param raw - The timestamp string from the vendor
 * @param vendor - The vendor identifier for format-specific parsing
 * @returns The Unix timestamp in milliseconds
 */
function parseVendorTimestamp(raw: string, vendor: string): number {
  // Implementation
}
```

---

## Early Returns and Control Flow

### The Rule: Exit Early, Stay Shallow

Deeply nested code is an orange flag. Prefer early returns to nesting.

**Avoid - Deep Nesting:**

```typescript
// Don't do this - hard to follow the logic
async function processUserRequest(request: Request): Promise<Response> {
  if (request.user) {
    if (request.user.isActive) {
      if (request.user.hasPermission('write')) {
        if (request.data.isValid()) {
          const result = await writeToDatabase(request.data);
          if (result.rowsAffected > 0) {
            return Response.success(result.id);
          } else {
            throw new Error('NoRowsAffected');
          }
        } else {
          throw new Error('InvalidData');
        }
      } else {
        throw new Error('NoPermission');
      }
    } else {
      throw new Error('UserInactive');
    }
  } else {
    throw new Error('NoUser');
  }
}
```

**Prefer - Early Returns:**

```typescript
/**
 * Processes a user request after validating all preconditions.
 *
 * @why
 * User requests must pass multiple validation layers (authentication,
 * authorization, data validation) before processing. This function
 * centralizes the validation logic and returns early on any failure,
 * making the "happy path" clear and the error cases explicit.
 *
 * @param request - The user request to process
 * @returns The response after successful processing
 * @throws {RequestError} When any validation fails
 */
async function processUserRequest(request: Request): Promise<Response> {
  // Extract user or fail fast
  if (!request.user) {
    throw new RequestError('NoUser', 'Request requires an authenticated user');
  }

  // Validate user state
  if (!request.user.isActive) {
    throw new RequestError('UserInactive', 'User account is disabled');
  }

  // Check permissions
  if (!request.user.hasPermission('write')) {
    throw new RequestError('NoPermission', 'User lacks required permission: write');
  }

  // Validate request data
  if (!request.data.isValid()) {
    throw new RequestError('InvalidData', 'Request data failed validation');
  }

  // Now we're on the happy path - all validations passed
  const result = await writeToDatabase(request.data);

  if (result.rowsAffected === 0) {
    throw new RequestError('NoRowsAffected', 'Database operation had no effect');
  }

  return Response.success(result.id);
}
```

### Switch Statements with Early Returns

```typescript
/**
 * Routes a payment to the appropriate processor based on method.
 *
 * @why
 * Payment processing varies significantly by method (card, bank transfer,
 * crypto). Using early returns in case handlers keeps each payment path
 * self-contained and prevents deep nesting.
 *
 * @param payment - The payment to process
 * @returns The payment receipt after processing
 */
async function processPayment(payment: Payment): Promise<Receipt> {
  switch (payment.method) {
    case 'credit_card': {
      const card = payment.details;

      // Validate card first
      if (card.isExpired()) {
        throw new PaymentError('CardExpired', 'Credit card has expired');
      }

      if (!card.hasValidCVV()) {
        throw new PaymentError('InvalidCVV', 'Card security code is invalid');
      }

      // Process the validated card
      return await processCardPayment(card, payment.amount);
    }

    case 'bank_transfer': {
      const details = payment.details;

      // Validate bank details
      if (!details.accountVerified) {
        throw new PaymentError('UnverifiedAccount', 'Bank account is not verified');
      }

      // Bank transfers take longer - create pending receipt
      return await createPendingReceipt(details, payment.amount);
    }

    case 'crypto': {
      const wallet = payment.details;

      // Validate wallet has sufficient balance
      const balance = await getWalletBalance(wallet.address);

      if (balance < payment.amount) {
        throw new PaymentError('InsufficientFunds', 'Wallet has insufficient balance');
      }

      // Crypto is irreversible - double-check amount
      if (payment.amount > CRYPTO_MAX_AMOUNT) {
        throw new PaymentError('AmountTooLarge', 'Crypto amount exceeds maximum');
      }

      return await processCryptoPayment(wallet, payment.amount);
    }

    default: {
      // Exhaustiveness check - TypeScript will error if we miss a case
      const _exhaustive: never = payment.method;
      throw new PaymentError('UnknownMethod', `Unknown payment method: ${_exhaustive}`);
    }
  }
}
```

### Array Methods with Early Continue

```typescript
/**
 * Processes log entries, skipping malformed entries.
 *
 * @why
 * Log files often contain corrupted or incomplete entries due to
 * crashes or disk issues. Rather than failing the entire batch,
 * we skip problematic entries and report them separately.
 *
 * @param entries - The raw log entries to process
 * @returns The processing result with valid and failed entries
 */
function processLogEntries(entries: LogEntry[]): ProcessingResult {
  const validEntries: NormalizedEntry[] = [];
  const failedEntries: FailedEntry[] = [];

  for (const entry of entries) {
    // Skip entries without timestamps - can't order them
    if (!entry.timestamp) {
      failedEntries.push({ entry, reason: 'Missing timestamp' });
      continue;
    }

    // Skip entries with invalid log levels
    if (!isValidLogLevel(entry.level)) {
      failedEntries.push({ entry, reason: 'Invalid log level' });
      continue;
    }

    // Skip entries where message is empty
    if (!entry.message?.trim()) {
      failedEntries.push({ entry, reason: 'Empty message' });
      continue;
    }

    // Entry passed all validations - process it
    const normalized = normalizeEntry(entry);
    validEntries.push(normalized);
  }

  return {
    processed: validEntries.length,
    failed: failedEntries.length,
    validEntries,
    failedEntries,
  };
}
```

---

## Variable Declaration and Grouping

### Variables at the Top

Declare variables at the start of functions when possible:

```typescript
/**
 * Configures the HTTP client with retry and timeout policies.
 *
 * @why
 * HTTP clients need consistent configuration across the application
 * for reliability (retries) and resource management (timeouts).
 * This function centralizes that configuration.
 *
 * @param config - The application configuration
 * @returns Configured axios instance
 */
function createHttpClient(config: AppConfig): AxiosInstance {
  // Group 1: Configuration extraction
  // These values control how the client behaves
  const timeoutSeconds = config.http.timeoutSecs;
  const retryAttempts = config.http.retryAttempts;
  const poolSize = config.http.connectionPoolSize;

  // Security: Force HTTPS in production environments
  // This prevents accidental cleartext transmission of sensitive data
  const enforceHttps = config.environment === 'production';

  // Group 2: Client configuration
  // Building the client happens in phases
  const client = axios.create({
    timeout: timeoutSeconds * 1000,
    maxRedirects: 5,
    httpsAgent: new https.Agent({
      maxSockets: poolSize,
    }),
  });

  // Add security interceptor for production
  if (enforceHttps) {
    client.interceptors.request.use((config) => {
      if (config.url?.startsWith('http:')) {
        throw new SecurityError('HTTP not allowed in production');
      }
      return config;
    });
  }

  // Group 3: Retry configuration
  // Configure axios-retry for resilient requests
  axiosRetry(client, {
    retries: retryAttempts,
    retryDelay: axiosRetry.exponentialDelay,
    retryCondition: isRetryableError,
  });

  return client;
}
```

### Complex Variable Declarations

Long declarations get breathing room:

```typescript
/**
 * Fetches user data from multiple sources and merges the results.
 *
 * @why
 * User data is distributed across multiple services (profile service,
 * preference service, authorization service). This function coordinates
 * the parallel fetching and merging of that data.
 *
 * @param userId - The ID of the user to fetch data for
 * @returns Merged user data from all services
 */
async function fetchUserData(userId: UserId): Promise<MergedUserData> {
  // Configuration for each service endpoint
  // These URLs come from environment configuration
  const profileUrl = `${config.profileServiceUrl}/users/${userId}`;
  const prefsUrl = `${config.prefsServiceUrl}/users/${userId}/preferences`;
  const authUrl = `${config.authServiceUrl}/users/${userId}/roles`;

  // Request configuration shared across all requests
  // Using a common timeout and headers for consistency
  const requestConfig: AxiosRequestConfig = {
    timeout: 5000,
    headers: {
      'X-Request-ID': generateRequestId(),
      'Accept': 'application/json',
    },
  };

  // Spawn concurrent fetches
  // Each fetch is independent, so we can run them in parallel
  const profilePromise = fetchProfileData(profileUrl, requestConfig);
  const prefsPromise = fetchPreferences(prefsUrl, requestConfig);
  const authPromise = fetchAuthorization(authUrl, requestConfig);

  // Await all results
  // If any fails, we fail fast - partial data is not useful
  const [profile, prefs, roles] = await Promise.all([
    profilePromise,
    prefsPromise,
    authPromise,
  ]);

  // Merge into unified view
  // This is the final result that callers actually want
  const merged = MergedUserData.fromParts(profile, prefs, roles);

  return merged;
}
```

---

## Interface and Type Patterns

### Using Zod for Runtime Validation

When types need runtime validation, use Zod schemas that derive TypeScript types:

```typescript
import { z } from 'zod';

/**
 * Configuration for the background job processor.
 *
 * @why
 * Background jobs need tunable parameters for different workloads.
 * The builder pattern (via object with defaults) allows callers to specify
 * only what they need while providing sensible defaults for everything else.
 *
 * @example
 * ```typescript
 * const config = JobProcessorConfig.parse({
 *   maxConcurrent: 8,
 *   pollInterval: 1000, // 1 second
 * });
 * // Other fields use defaults
 * ```
 */
export const JobProcessorConfigSchema = z.object({
  /**
   * Maximum number of concurrent jobs to process.
   * Higher values increase throughput but use more resources.
   */
  maxConcurrent: z.number().int().min(1).max(100).default(4),

  /**
   * How long to wait between polling for new jobs (milliseconds).
   */
  pollInterval: z.number().int().min(100).default(5000),

  /**
   * Maximum retry attempts for failed jobs.
   */
  maxRetries: z.number().int().min(0).default(3),

  /**
   * Whether to process jobs in priority order.
   */
  prioritize: z.boolean().default(true),

  /**
   * Optional callback for permanently failed jobs.
   */
  onFailure: z.function().optional(),

  /**
   * Optional custom serializer for job payloads.
   */
  serializer: z.custom<JobSerializer>().optional(),
});

export type JobProcessorConfig = z.infer<typeof JobProcessorConfigSchema>;
```

### Props Interfaces for Functions with Many Arguments

When a function takes more than 3 arguments, use an interface:

```typescript
/**
 * Arguments for creating a new user account.
 *
 * @why
 * User creation requires multiple pieces of data that are logically
 * grouped but distinct. Using an interface:
 * - Prevents argument order bugs
 * - Allows adding optional fields without changing function signature
 * - Self-documents what data is required
 */
interface CreateUserArgs {
  /** Unique identifier for the user (typically email or username). */
  identifier: string;

  /** User's display name shown in the UI. */
  displayName: string;

  /** Initial password - will be hashed before storage. */
  password: string;

  /** User's primary role determining initial permissions. */
  role: UserRole;

  /** Optional metadata for the user account. */
  metadata?: UserMetadata;

  /** Source of this user creation (web, api, import, etc). */
  source: CreationSource;
}

/**
 * Creates a new user account with validation and auditing.
 *
 * @why
 * User creation is a security-sensitive operation requiring:
 * - Password validation (complexity requirements)
 * - Identifier uniqueness checks
 * - Audit logging
 * - Initial permission setup
 *
 * This function coordinates all these concerns.
 *
 * @param args - The user creation arguments
 * @returns The created user
 * @throws {UserCreationError} When validation fails
 */
export async function createUser(args: CreateUserArgs): Promise<User> {
  // Validate identifier format
  if (!isValidIdentifier(args.identifier)) {
    throw new UserCreationError('InvalidIdentifier', 'Identifier must be valid email or username');
  }

  // Check password complexity
  if (!meetsPasswordPolicy(args.password)) {
    throw new UserCreationError('WeakPassword', 'Password does not meet security requirements');
  }

  // Verify identifier isn't already in use
  if (await userExists(args.identifier)) {
    throw new UserCreationError('AlreadyExists', `User ${args.identifier} already exists`);
  }

  // Hash password before storage (security: never store plaintext)
  const passwordHash = await hashPassword(args.password);

  // Create user record
  const user: User = {
    id: generateUserId(),
    identifier: args.identifier,
    displayName: args.displayName,
    passwordHash,
    role: args.role,
    metadata: args.metadata,
    createdAt: new Date(),
    createdVia: args.source,
  };

  // Persist to database
  await persistUser(user);

  // Audit log the creation
  await auditLog.record({
    type: 'UserCreated',
    userId: user.id,
    source: args.source,
    role: args.role,
  });

  return user;
}

// Usage:
// const user = await createUser({
//   identifier: 'alice@example.com',
//   displayName: 'Alice Smith',
//   password: await hashPassword(plainPassword),
//   role: UserRole.Standard,
//   source: CreationSource.WebRegistration,
// });
```

---

## Whitespace and Grouping

### Blank Lines Before Control Flow

```typescript
/**
 * Processes incoming messages from the queue.
 *
 * @why
 * Messages arrive from multiple sources and need to be routed
 * to appropriate handlers based on their type. This function
 * is the entry point for all message processing.
 *
 * @param msg - The message to process
 */
async function processMessage(msg: Message): Promise<void> {
  // Extract message type for routing decision
  const msgType = msg.header.messageType;

  // Security: Verify message signature before processing
  // This prevents tampered messages from being processed
  if (!verifySignature(msg)) {
    securityLog.warn('Invalid message signature', msg.header);
    throw new ProcessingError('InvalidSignature');
  }

  // Route to appropriate handler based on message type
  switch (msgType) {
    case 'order': {
      const order = msg.payload;

      // Validate order before processing
      if (order.amount <= 0) {
        throw new ProcessingError('InvalidAmount');
      }

      await handleOrder(order);
      break;
    }

    case 'inventory_update': {
      const update = msg.payload;

      // Skip stale updates
      if (update.timestamp < getLastSyncTime()) {
        console.debug('Skipping stale inventory update');
        return;
      }

      await handleInventoryUpdate(update);
      break;
    }

    case 'system_event': {
      // System events are always handled, no validation needed
      await handleSystemEvent(msg.payload);
      break;
    }
  }

  // Log successful processing for metrics
  metrics.increment('messages_processed', { type: msgType });
}
```

### Grouping Related Operations

```typescript
/**
 * Initializes the application with all required components.
 *
 * @why
 * Application startup involves multiple phases: configuration loading,
 * dependency initialization, database connections, and server startup.
 * This function orchestrates the entire process with proper error handling
 * and resource cleanup on failure.
 *
 * @param configPath - Path to the configuration file
 * @returns Initialized application instance
 */
export async function initializeApp(configPath: string): Promise<Application> {
  // === Phase 1: Configuration Loading ===
  // Load configuration from file and environment
  const config = await loadConfig(configPath);

  // Validate critical configuration values
  if (!config.database.url) {
    throw new StartupError('MissingDatabaseUrl', 'Database URL is required');
  }

  // === Phase 2: Logging Setup ===
  // Configure logging before other components so they can log their initialization
  initLogging(config.logging);
  console.info(`Starting application v${process.env.npm_package_version}`);

  // === Phase 3: Database Connection ===
  // Connect to database before initializing other services that depend on it
  const dbPool = await createDbPool(config.database);

  // Run migrations to ensure schema is up to date
  await runMigrations(dbPool);

  // === Phase 4: External Service Clients ===
  // Initialize clients for external services
  const httpClient = createHttpClient(config.http);
  const cacheClient = await createCacheClient(config.cache);

  // === Phase 5: Application Services ===
  // Build the service layer with all dependencies
  const services = new Services(dbPool, httpClient, cacheClient);

  // === Phase 6: Server Startup ===
  // Start the HTTP server with all routes configured
  const server = createServer(config.server.port, services);

  // Signal handler for graceful shutdown
  const shutdown = createShutdownHandler();

  return new Application(server, shutdown, services);
}
```

---

## Error Handling

### Custom Error Classes

```typescript
/**
 * Errors that can occur during payment processing.
 *
 * @why
 * Payment processing involves multiple external systems (payment gateways,
 * fraud detection, accounting) each with their own error types. This
 * unified error class provides a consistent interface for callers while
 * preserving detailed error information for debugging.
 */
class PaymentError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly details?: Record<string, unknown>
  ) {
    super(message);
    this.name = 'PaymentError';
  }

  /**
   * Check if this is a user-correctable error.
   *
   * @why
   * Some payment errors (e.g., expired card) can be fixed by the user,
   * while others (e.g., gateway downtime) require system intervention.
   * This property helps the UI decide whether to show a retry button
   * or an error message.
   */
  get isUserError(): boolean {
    return ['CardExpired', 'CardDeclined', 'InvalidCVV', 'InsufficientFunds'].includes(this.code);
  }

  /**
   * Get the appropriate HTTP status code for this error.
   *
   * @why
   * API endpoints need to return appropriate HTTP status codes based
   * on the error type. This method centralizes that mapping.
   */
  get httpStatus(): number {
    switch (this.code) {
      case 'CardExpired':
      case 'InvalidCVV':
      case 'InsufficientFunds':
        return 400; // Bad Request - user can fix

      case 'FraudDetected':
        return 403; // Forbidden

      case 'GatewayError':
      case 'NetworkError':
        return 503; // Service Unavailable - retry later

      default:
        return 500;
    }
  }
}

/**
 * Processes a payment with comprehensive error handling.
 *
 * @why
 * Payment processing must handle many failure modes gracefully:
 * - User errors (insufficient funds, expired card) - show to user
 * - System errors (network, database) - retry or fail gracefully
 * - Security errors (fraud) - log and escalate
 *
 * @param payment - The payment to process
 * @returns The payment receipt
 * @throws {PaymentError} When payment fails
 */
async function processPayment(payment: PaymentRequest): Promise<Receipt> {
  // Validate payment amount
  if (payment.amount <= 0) {
    throw new PaymentError('InvalidAmount', 'Payment amount must be positive');
  }

  // Check fraud rules
  const fraudCheck = await fraudService.check(payment);

  if (!fraudCheck.passed) {
    // Security: Log fraud attempts for manual review
    securityLog.alert('Fraud detected', fraudCheck);

    throw new PaymentError('FraudDetected', 'Payment blocked by fraud detection', {
      code: fraudCheck.code,
    });
  }

  // Process through payment gateway
  const gatewayResult = await gateway.charge(payment);

  switch (gatewayResult.status) {
    case 'approved': {
      // Record transaction in database
      const receipt = await persistTransaction(gatewayResult);
      return receipt;
    }

    case 'declined': {
      throw new PaymentError('CardDeclined', `Card declined: ${gatewayResult.declineReason}`);
    }

    case 'error': {
      // Log gateway errors for monitoring
      metrics.increment('payment_gateway_errors');
      throw new PaymentError('GatewayError', 'Payment gateway error');
    }

    default: {
      const _exhaustive: never = gatewayResult.status;
      throw new PaymentError('UnknownStatus', `Unknown gateway status: ${_exhaustive}`);
    }
  }
}
```

### Result Types for Explicit Error Handling

```typescript
/**
 * A Result type for operations that can fail.
 *
 * @why
 * Using Result types instead of exceptions makes error handling
 * explicit at the type level. Callers must handle both success
 * and failure cases, preventing unhandled errors.
 */
type Result<T, E> =
  | { ok: true; value: T }
  | { ok: false; error: E };

/**
 * Creates a successful Result.
 */
function ok<T, E>(value: T): Result<T, E> {
  return { ok: true, value };
}

/**
 * Creates a failed Result.
 */
function err<T, E>(error: E): Result<T, E> {
  return { ok: false, error };
}

/**
 * Validates user input before processing.
 *
 * @why
 * Input validation returns multiple possible error types.
 * Using a Result type forces callers to handle all error cases.
 *
 * @param input - The raw user input
 * @returns Result with validated data or validation errors
 */
function validateUserInput(input: unknown): Result<UserInput, ValidationErrors> {
  const errors: ValidationErrors = {};

  // Validate email
  if (!input || typeof input !== 'object') {
    return err({ _general: 'Input must be an object' });
  }

  const { email, age } = input as Record<string, unknown>;

  if (typeof email !== 'string') {
    errors.email = 'Email is required';
  } else if (!isValidEmail(email)) {
    errors.email = 'Invalid email format';
  }

  // Validate age
  if (typeof age !== 'number') {
    errors.age = 'Age is required';
  } else if (age < 0 || age > 150) {
    errors.age = 'Age must be between 0 and 150';
  }

  // Return early if any validation failed
  if (Object.keys(errors).length > 0) {
    return err(errors);
  }

  // All validation passed
  return ok({
    email: email as string,
    age: age as number,
  });
}

// Usage forces error handling:
// const result = validateUserInput(rawInput);
// if (!result.ok) {
//   // Handle error case
//   return;
// }
// // TypeScript knows result.value is valid here
// processUserInput(result.value);
```

---

## Async Patterns

### Parallel Async with Early Returns

```typescript
/**
 * Fetches user data with caching and fallback.
 *
 * @why
 * User data is frequently accessed but rarely changes. This function
 * implements a cache-first strategy with database fallback, minimizing
 * database load while ensuring fresh data is available.
 *
 * @param userId - The ID of the user to fetch
 * @returns The user data
 */
async function getUserCached(userId: UserId): Promise<User> {
  // Try cache first
  const cached = await cache.get(userId);

  if (cached) {
    // Cache hit - validate the cached data isn't stale
    if (!isStale(cached)) {
      metrics.increment('cache_hit');
      return cached;
    }

    // Stale data - log and continue to refresh
    console.debug(`Cache hit but data is stale for user ${userId}`);
  }

  // Cache miss or stale - fetch from database
  metrics.increment('cache_miss');

  const user = await db.fetchUser(userId);

  // Update cache for next time
  try {
    await cache.set(userId, user);
  } catch (e) {
    // Don't fail the request if cache update fails
    console.warn('Failed to update cache:', e);
  }

  return user;
}
```

### AbortController for Cancellable Operations

```typescript
/**
 * Fetches search results with debouncing and cancellation.
 *
 * @why
 * Search-as-you-type needs to cancel in-flight requests when the
 * user types more characters. AbortController provides a standard
 * way to cancel fetch operations.
 *
 * @param query - The search query
 * @param signal - AbortSignal for cancellation
 * @returns Search results
 */
async function fetchSearchResults(
  query: string,
  signal: AbortSignal
): Promise<SearchResults> {
  // Validate query before making request
  if (query.length < 2) {
    return { results: [], total: 0 };
  }

  // Check if already cancelled
  if (signal.aborted) {
    throw new Error('Request was cancelled');
  }

  const response = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
    signal,
    headers: { 'Accept': 'application/json' },
  });

  if (!response.ok) {
    throw new Error(`Search failed: ${response.status}`);
  }

  const data = await response.json();

  // Post-process results
  const results = data.hits.map(normalizeSearchResult);

  return { results, total: data.total };
}

// Usage in React component:
// useEffect(() => {
//   const controller = new AbortController();
//
//   fetchSearchResults(query, controller.signal)
//     .then(setResults)
//     .catch(err => {
//       if (err.name !== 'AbortError') {
//         setError(err);
//       }
//     });
//
//   return () => controller.abort();
// }, [query]);
```

---

## Module Structure

### File Organization

```typescript
// src/services/
// ├── index.ts      // Re-exports and module docs
// ├── payment.ts    // Payment processing
// ├── inventory.ts  // Inventory management
// └── user.ts       // User management

// src/services/index.ts
/**
 * @module services
 *
 * Business logic services for the application.
 *
 * @why
 * Services encapsulate the core business logic of the application.
 * They are independent of the transport layer (HTTP, gRPC, CLI) and
 * can be used from any entry point. Each service owns a specific
 * domain (payments, inventory, users) and provides a clean API
 * for that domain.
 *
 * @architecture
 * Services are designed to be:
 * - **Testable**: Business logic is pure and testable
 * - **Composable**: Services can depend on other services
 * - **Observable**: All operations are logged and measured
 * - **Transaction-safe**: Database operations use transactions
 *
 * @example
 * ```typescript
 * import { PaymentService, PaymentRequest } from './services';
 *
 * const service = new PaymentService(dbPool, config);
 * const receipt = await service.processPayment(request);
 * ```
 */

export * from './inventory';
export * from './payment';
export * from './user';

import { PaymentService } from './payment';
import { InventoryService } from './inventory';
import { UserService } from './user';

/**
 * Container for all application services.
 *
 * @why
 * Services have interdependencies (e.g., payments depend on inventory
 * to check stock). This container manages those dependencies and
 * ensures each service is only created once.
 */
export class Services {
  public readonly payment: PaymentService;
  public readonly inventory: InventoryService;
  public readonly user: UserService;

  constructor(
    private readonly db: DbPool,
    config: AppConfig
  ) {
    this.inventory = new InventoryService(db, config.inventory);
    this.user = new UserService(db, config.user);

    // Payment service depends on inventory service
    this.payment = new PaymentService(
      db,
      this.inventory,
      config.payment
    );
  }
}
```

### Barrel Exports with Clear Organization

```typescript
// src/types/index.ts
/**
 * @module types
 *
 * Shared TypeScript types and interfaces.
 *
 * @why
 * Centralizing type definitions prevents duplication and ensures
 * consistency across the codebase. Types are organized by domain
 * for easy discovery.
 */

// Domain types
export type { User, UserRole, UserPreferences } from './user';
export type { Order, OrderStatus, OrderLineItem } from './order';
export type { Payment, PaymentMethod, PaymentReceipt } from './payment';

// Infrastructure types
export type { DatabaseConfig, CacheConfig, HttpConfig } from './config';
export type { Logger, MetricsClient, CacheClient } from './infrastructure';

// Error types
export { ApiError, ValidationError, NotFoundError } from './errors';
export type { ErrorCode, ErrorDetails } from './errors';

// Utility types
export type { DeepPartial, OptionalId, Timestamped } from './utility';
```

---

## Additional Patterns

### Exhaustive Type Checking

```typescript
/**
 * Status values for order processing.
 */
type OrderStatus =
  | 'pending'
  | 'processing'
  | 'shipped'
  | 'delivered'
  | 'cancelled';

/**
 * Gets the human-readable description of an order status.
 *
 * @why
 * UI needs to display status in user-friendly language. Using
 * exhaustive checking ensures TypeScript will error if we add
 * a new status but forget to add its description.
 *
 * @param status - The order status
 * @returns Human-readable description
 */
function getStatusDescription(status: OrderStatus): string {
  switch (status) {
    case 'pending':
      return 'Waiting for payment confirmation';

    case 'processing':
      return 'Preparing items for shipment';

    case 'shipped':
      return 'Items are in transit';

    case 'delivered':
      return 'Items have been delivered';

    case 'cancelled':
      return 'Order has been cancelled';

    default: {
      // Compile-time exhaustiveness check
      const _exhaustive: never = status;
      return _exhaustive;
    }
  }
}
```

### Readonly and Immutability

```typescript
/**
 * Configuration for the API client (immutable).
 *
 * @why
 * Using readonly and Readonly ensures configuration can't be
 * accidentally mutated after creation. This prevents subtle bugs
 * where one part of the code changes config that others depend on.
 */
interface ApiClientConfig {
  readonly baseUrl: string;
  readonly timeout: number;
  readonly retries: number;
  readonly headers: Readonly<Record<string, string>>;
}

/**
 * Creates a frozen configuration object.
 *
 * @param config - Raw configuration values
 * @returns Immutable configuration
 */
function createImmutableConfig(config: Partial<ApiClientConfig>): ApiClientConfig {
  return Object.freeze({
    baseUrl: config.baseUrl ?? 'https://api.example.com',
    timeout: config.timeout ?? 5000,
    retries: config.retries ?? 3,
    headers: Object.freeze({ ...config.headers }),
  });
}
```

### Type Guards for Runtime Safety

```typescript
/**
 * Type guard for validating API response shape.
 *
 * @why
 * External APIs might return unexpected data. Type guards let us
 * validate at runtime while TypeScript narrows the type, giving us
 * both safety and type narrowing.
 *
 * @param data - The unknown data from the API
 * @returns True if data matches User shape
 */
function isUser(data: unknown): data is User {
  if (typeof data !== 'object' || data === null) {
    return false;
  }

  const obj = data as Record<string, unknown>;

  return (
    typeof obj.id === 'string' &&
    typeof obj.email === 'string' &&
    typeof obj.displayName === 'string' &&
    (obj.role === 'admin' || obj.role === 'user' || obj.role === 'guest')
  );
}

/**
 * Fetches a user with runtime type validation.
 *
 * @param userId - The ID of the user to fetch
 * @returns Validated user object
 * @throws {ApiError} When response doesn't match expected type
 */
async function fetchUser(userId: string): Promise<User> {
  const response = await fetch(`/api/users/${userId}`);
  const data = await response.json();

  if (!isUser(data)) {
    throw new ApiError(
      'InvalidResponse',
      `API returned unexpected user structure: ${JSON.stringify(data)}`
    );
  }

  // TypeScript now knows data is User
  return data;
}
```
