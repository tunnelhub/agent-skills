# HTTP Interceptor

This reference explains how to configure HTTP request/response logging in TunnelHub SDK.

## Overview

The HTTP interceptor automatically logs all HTTP requests and responses made within an integration. This is useful for
debugging, monitoring, and auditing API calls.

### When to Use

- Debug API integration issues
- Monitor HTTP traffic
- Audit API calls for compliance
- Track request/response patterns
- Troubleshoot network issues

## setupInterceptor

```typescript
export const setupInterceptor = (options?: SetupOptions): FetchInterceptor | null
```

Configure the HTTP interceptor to log all fetch requests.

**Parameters:**

```typescript
type SetupOptions = {
  enableGlobalFetch?: boolean; // Intercept global fetch (default: true)
  enableUndiciFetch?: boolean; // Intercept undici fetch (default: false)
};
```

**Returns:**

- `FetchInterceptor | null` - Interceptor instance or null if already set up

## Usage

### Basic Setup

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

// Setup interceptor at the top of your integration file
setupInterceptor();

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  // All fetch calls will now be logged
  protected async loadSourceSystemData(): Promise<MyType[]> {
    const response = await fetch('https://api.example.com/data');
    return await response.json();
  }
}
```

### Global and Undici Fetch

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

// Intercept both global fetch and undici fetch
setupInterceptor({
  enableGlobalFetch: true,
  enableUndiciFetch: true,
});

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadData(): Promise<MyType[]> {
    // This will be logged
    const response1 = await fetch('https://api.example.com/data');

    // This will also be logged (if using undici)
    const response2 = await fetch('https://api.example.com/more-data');

    return await response1.json();
  }
}
```

### ⚠️ Important: Undici Version Compatibility

When using `enableUndiciFetch: true`, ensure that your project's `undici` package matches the SDK version (currently
`^7.10.0`).

**Why this matters:**

If your project has a different undici version, npm/pnpm will install both versions in `node_modules`, causing two
undici instances to exist. The interceptor will only patch the SDK's version, so fetch calls from other undici instances
won't be logged.

**Example of the problem:**

```json
// package.json - BAD
{
  "dependencies": {
    "@tunnelhub/sdk": "^3.2.0", // Uses undici@^7.10.0
    "undici": "^6.0.0" // Different version!
  }
}
```

**How to fix:**

```json
// package.json - GOOD
{
  "dependencies": {
    "@tunnelhub/sdk": "^3.2.0",  // Uses undici@^7.10.0
    "undici": "^7.10.0"            // Same version as SDK
  }
}

// OR - Remove undici from your package.json
// The SDK will provide its own version
{
  "dependencies": {
    "@tunnelhub/sdk": "^3.2.0"
    // No undici dependency - SDK uses its own
  }
}
```

**Checking for version conflicts:**

```bash
# Check installed undici versions
npm ls undici

# Output should show:
# @tunnelhub/sdk -> undici@7.10.0
# (No duplicate versions)
```

**Recommendation:**

- If you're not explicitly using undici in your code, remove it from your `package.json`
- The SDK provides undici as a dependency, so it will be available
- If you need undici, ensure your version matches exactly (e.g., `^7.10.0`)

### Integration Example

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  setupInterceptor,
} from '@tunnelhub/sdk';

// Setup interceptor once at module level
setupInterceptor({
  enableGlobalFetch: true,
  enableUndiciFetch: true,
});

type User = {
  userId: string;
  name: string;
  email: string;
};

class UserSyncIntegration extends DeltaIntegrationFlow<User> {
  protected async loadSourceSystemData(): Promise<User[]> {
    console.log('[Integration] Loading source system data...');

    const response = await fetch('https://crm.example.com/api/users', {
      headers: {
        Authorization: 'Bearer your-api-key',
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to load users: ${response.statusText}`);
    }

    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<User[]> {
    console.log('[Integration] Loading target system data...');

    const response = await fetch('https://erp.example.com/api/users', {
      headers: {
        Authorization: 'Bearer your-api-key',
        'Content-Type': 'application/json',
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to load ERP users: ${response.statusText}`);
    }

    return await response.json();
  }

  protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
    console.log(`[Integration] Inserting user: ${item.userId}`);

    const response = await fetch('https://erp.example.com/api/users', {
      method: 'POST',
      headers: {
        Authorization: 'Bearer your-api-key',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        externalId: item.userId,
        name: item.name,
        email: item.email,
      }),
    });

    if (!response.ok) {
      throw new Error(`Failed to create user: ${response.statusText}`);
    }

    console.log(`[Integration] User ${item.userId} created successfully`);

    return {message: 'User created'};
  }

  protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
    console.log(`[Integration] Updating user: ${item.userId}`);

    const response = await fetch(`https://erp.example.com/api/users/${oldItem.externalId}`, {
      method: 'PATCH',
      headers: {
        Authorization: 'Bearer your-api-key',
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        name: newItem.name,
        email: newItem.email,
      }),
    });

    if (!response.ok) {
      throw new Error(`Failed to update user: ${response.statusText}`);
    }

    console.log(`[Integration] User ${item.userId} updated successfully`);

    return {message: 'User updated'};
  }

  protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
    console.log(`[Integration] Deleting user: ${item.userId}`);

    const response = await fetch(`https://erp.example.com/api/users/${item.externalId}`, {
      method: 'DELETE',
      headers: {
        Authorization: 'Bearer your-api-key',
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to delete user: ${response.statusText}`);
    }

    console.log(`[Integration] User ${item.userId} deleted successfully`);

    return {message: 'User deleted'};
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'userId', fieldLabel: 'User ID', fieldType: 'TEXT'},
      {fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT'},
      {fieldName: 'email', fieldLabel: 'Email', fieldType: 'TEXT'},
    ];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new UserSyncIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Integration completed successfully'};
};
```

## What Gets Logged

### Request Logging

The interceptor logs:

- HTTP method (GET, POST, PUT, PATCH, DELETE, etc.)
- Full URL
- Request headers (excluding sensitive headers)
- Request body (if present)
- Timestamp

### Response Logging

The interceptor logs:

- HTTP status code
- Response headers (excluding sensitive headers)
- Response body (if present)
- Duration (time taken for request)
- Timestamp

### Error Logging

The interceptor logs:

- Error message
- Request details that caused the error
- Timestamp

## Sensitive Headers

The interceptor automatically masks sensitive headers:

- `authorization`
- `cookie`
- `set-cookie`
- `x-api-key`
- `x-auth-token`

These headers are masked in logs but their presence is recorded.

## Output Format

### Request Log Example

```
[HTTP] GET https://api.example.com/users
[HTTP] Headers: {
  "content-type": "application/json",
  "authorization": "***masked***"
}
[HTTP] Duration: 0ms
```

### Response Log Example

```
[HTTP] Response 200 OK
[HTTP] Headers: {
  "content-type": "application/json",
  "content-length": "1234"
}
[HTTP] Duration: 245ms
[HTTP] Body: [{"userId":"1","name":"John Doe","email":"john@example.com"}]
```

### Error Log Example

```
[HTTP] Error: FetchError: request to https://api.example.com/users failed, reason: connect ECONNREFUSED
[HTTP] Request: GET https://api.example.com/users
[HTTP] Headers: {
  "content-type": "application/json",
  "authorization": "***masked***"
}
```

## Best Practices

### 1. Setup Once

```typescript
// Good: Setup once at module level
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor();

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  // ...
}

// Bad: Setup multiple times
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async preProcessingCustomerRoutines(): Promise<void> {
    setupInterceptor(); // This won't work after first call
  }
}
```

### 2. Enable Verbose Mode for Detailed Logs

```typescript
import {SDK, setupInterceptor} from '@tunnelhub/sdk';

// Enable verbose logging
SDK.verbose = true;

setupInterceptor();

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  // Now all HTTP calls will be logged with full details
}
```

### 3. Use for Debugging API Issues

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor();

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    try {
      // This will log full request/response
      const response = await fetch('https://api.example.com/items', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify(item),
      });

      if (!response.ok) {
        // Check logs to see what went wrong
        console.error(`HTTP ${response.status}: Check interceptor logs for details`);
        throw new Error(`HTTP ${response.status}`);
      }

      return {message: 'Item created'};
    } catch (error) {
      // Interceptor logs will show the failed request
      throw error;
    }
  }
}
```

### 4. Monitor Performance

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private requestDurations: number[] = [];

  protected async postProcessingCustomerRoutines(): Promise<void> {
    // Analyze performance (logs include durations)
    if (this.requestDurations.length > 0) {
      const avgDuration = this.requestDurations.reduce((a, b) => a + b, 0) / this.requestDurations.length;

      const maxDuration = Math.max(...this.requestDurations);

      console.log(`[Performance] Average request duration: ${avgDuration}ms`);
      console.log(`[Performance] Max request duration: ${maxDuration}ms`);
    }
  }
}
```

## Limitations

1. **Singleton Pattern**: The interceptor can only be set up once per runtime. Subsequent calls will return `null`.

2. **Only Fetch**: Currently only intercepts `fetch` calls (global and undici). Other HTTP libraries (axios, got, etc.)
   are not intercepted.

3. **Undici Version Alignment**: When using `enableUndiciFetch`, ensure your project's undici version matches the SDK
   version. Different versions will cause duplicate installations in `node_modules`, and only the SDK's version will be
   intercepted. See "Undici Version Compatibility" section above for details.

4. **Performance Impact**: Logging adds overhead. Disable in production if performance is critical.

5. **Memory Usage**: Large response bodies are logged, which can increase memory usage.

## Conditional Setup

```typescript
import {setupInterceptor, SDK, AutomationParameter} from '@tunnelhub/sdk';

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    // Only enable interceptor in debug mode
    const debugMode = AutomationParameter.getBooleanParameter(this.parameters, 'debug_http_logging', false);

    if (debugMode) {
      setupInterceptor();
      SDK.verbose = true;
      console.log('[Integration] HTTP interceptor enabled');
    }
  }
}
```

## Common Use Cases

### Use Case 1: Debugging API Failures

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor();

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    try {
      const response = await fetch('https://api.example.com/items', {
        method: 'POST',
        body: JSON.stringify(item),
      });

      // Check logs for request details if this fails
      if (!response.ok) {
        const responseBody = await response.text();
        console.error('[Error] Response body:', responseBody);
        throw new Error(`HTTP ${response.status}: ${responseBody}`);
      }

      return {message: 'Item created'};
    } catch (error) {
      // Interceptor logs will show the failed request
      console.error('[Error] Request failed. Check interceptor logs.');
      throw error;
    }
  }
}
```

### Use Case 2: Monitoring API Response Times

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor();

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private slowRequests: string[] = [];

  protected async loadSourceSystemData(): Promise<MyType[]> {
    console.log('[Timing] Loading source data...');
    const response = await fetch('https://api.example.com/data');

    // Check logs for duration
    const duration = this.extractDurationFromLogs(response);

    if (duration > 5000) {
      this.slowRequests.push(`GET /data (${duration}ms)`);
    }

    return await response.json();
  }

  protected async postProcessingCustomerRoutines(): Promise<void> {
    if (this.slowRequests.length > 0) {
      console.warn('[Performance] Slow requests detected:');
      this.slowRequests.forEach(req => console.warn(`  - ${req}`));
    }
  }

  private extractDurationFromLogs(response: Response): number {
    // Extract duration from interceptor logs (implementation varies)
    return 0;
  }
}
```

### Use Case 3: Auditing API Calls

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor();

class AuditableIntegration extends DeltaIntegrationFlow<MyType> {
  private apiCalls: string[] = [];

  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    const timestamp = new Date().toISOString();

    const response = await fetch('https://api.example.com/items', {
      method: 'POST',
      body: JSON.stringify(item),
    });

    // Log for audit
    this.apiCalls.push(`[${timestamp}] POST /items - ID: ${item.id}, Status: ${response.status}`);

    return {message: 'Item created'};
  }

  protected async postProcessingCustomerRoutines(): Promise<void> {
    console.log('[Audit] API calls made during integration:');
    this.apiCalls.forEach(call => console.log(`  ${call}`));
  }
}
```
