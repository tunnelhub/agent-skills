# Parameter Management

This reference explains how to use the `AutomationParameter` class to manage integration parameters in TunnelHub SDK.

## Overview

`AutomationParameter` provides two ways to handle parameters:

1. **Static Methods**: Access parameters from the event (`this.parameters`)
2. **Instance Methods**: Persist and retrieve parameters from DynamoDB

### When to Use Each

| Method Type | Use Case | Example |
|-------------|----------|---------|
| Static | Configuration passed in event | API keys, system IDs, feature flags |
| Instance | Persistent state between executions | Last sync date, cursors, incremental values |

## Static Methods

Static methods access parameters from `this.parameters`, which is passed in the `ProcessorPayload` event. These parameters are typically configured in the TunnelHub UI.

### getParameter

```typescript
public static getParameter(
  parameters: AutomatedIntegrationParameters,
  parameterName: string
): string | null
```

Retrieve a parameter value by name.

**Parameters:**
- `parameters`: The parameters object (usually `this.parameters`)
- `parameterName`: Name of the parameter to retrieve

**Returns:**
- `string | null` - Parameter value or null if not found

**Example:**
```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    // Get API key (may be null)
    const apiKey = AutomationParameter.getParameter(this.parameters, 'api_key');

    if (!apiKey) {
      throw new Error('API key is required');
    }

    // Use the API key
    const response = await fetch('https://api.example.com/data', {
      headers: { 'Authorization': `Bearer ${apiKey}` }
    });

    return await response.json();
  }
}
```

### getRequiredParameter

```typescript
public static getRequiredParameter(
  parameters: AutomatedIntegrationParameters,
  parameterName: string
): string
```

Retrieve a required parameter by name. Throws an error if the parameter is not found.

**Parameters:**
- `parameters`: The parameters object (usually `this.parameters`)
- `parameterName`: Name of the parameter to retrieve

**Returns:**
- `string` - Parameter value

**Throws:**
- `Error` if parameter is not found

**Example:**
```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    // This will throw if 'webhook_url' is not configured
    const webhookUrl = AutomationParameter.getRequiredParameter(
      this.parameters,
      'webhook_url'
    );

    await fetch(webhookUrl, {
      method: 'POST',
      body: JSON.stringify(item)
    });

    return { message: 'Item sent to webhook' };
  }
}
```

**Use case**: When a parameter is mandatory for the integration to work.

### getBooleanParameter

```typescript
public static getBooleanParameter(
  parameters: AutomatedIntegrationParameters,
  paramName: string,
  defaultValue: boolean = true
): boolean
```

Retrieve a boolean parameter value, converting string representations to boolean.

**Parameters:**
- `parameters`: The parameters object (usually `this.parameters`)
- `paramName`: Name of the parameter to retrieve
- `defaultValue`: Default value if parameter is not found (default: `true`)

**Returns:**
- `boolean` - Parameter value as boolean

**Conversion Rules:**
- `'1'` or `'true'` (case-insensitive) → `true`
- `'0'` or `'false'` (case-insensitive) → `false`
- `null` or `undefined` → `defaultValue`
- Any other value → `defaultValue`

**Example:**
```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected realtimeLoggingThreshold: number =
    AutomationParameter.getBooleanParameter(this.parameters, 'verbose_logging', false)
      ? 500
      : 100;

  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    const useRetry = AutomationParameter.getBooleanParameter(
      this.parameters,
      'retry_enabled',
      true
    );

    if (useRetry) {
      return await this.insertWithRetry(item);
    } else {
      return await this.insertWithoutRetry(item);
    }
  }
}
```

**Use case**: Feature flags, optional behaviors, configuration switches.

## Instance Methods

Instance methods interact with DynamoDB to persist and retrieve parameters between executions. This is useful for stateful integrations.

### Constructor

```typescript
constructor(
  tenantId: string,
  automationId: string,
  environmentId: string
)
```

Create a new parameter manager instance.

**Parameters:**
- `tenantId`: Tenant identifier (from `this.tenantId`)
- `automationId`: Automation identifier (from `this.automationId`)
- `environmentId`: Environment identifier (from `this.environmentId`)

**Example:**
```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }
}
```

### getParameter (Instance)

```typescript
public async getParameter(
  parameterName: string,
  defaultValue: string | null = null
): Promise<string | null>
```

Retrieve a parameter value from DynamoDB.

**Parameters:**
- `parameterName`: Name of the parameter to retrieve
- `defaultValue`: Default value if parameter is not found (default: `null`)

**Returns:**
- `Promise<string | null>` - Parameter value or default value

**Example:**
```typescript
class IncrementalSyncIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    const paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );

    // Get last sync date (or null if first run)
    const lastSyncDate = await paramManager.getParameter('last_sync_date', null);

    let url = 'https://api.example.com/data';

    if (lastSyncDate) {
      url += `?modified_since=${encodeURIComponent(lastSyncDate)}`;
    }

    const response = await fetch(url);
    return await response.json();
  }

  protected async afterIntegration(): Promise<void> {
    // Save current sync date for next execution
    const paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );

    await paramManager.saveParameter(
      'last_sync_date',
      new Date().toISOString()
    );
  }
}
```

**Use cases:**
- Incremental sync timestamps
- Pagination cursors
- Last processed IDs
- Execution counters

### saveParameter

```typescript
public async saveParameter(
  parameterName: string,
  parameterValue: string
): Promise<boolean>
```

Save or update a parameter value in DynamoDB.

**Parameters:**
- `parameterName`: Name of the parameter to save
- `parameterValue`: Value to save (will be converted to string)

**Returns:**
- `Promise<boolean>` - `true` if successful, `false` otherwise

**Throws:**
- `Error` if automation is not found in DynamoDB

**Example:**
```typescript
class CursorBasedSyncIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    const paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );

    // Get cursor from last execution
    const cursor = await paramManager.getParameter('cursor', '0');

    let url = `https://api.example.com/data?cursor=${cursor}`;
    const response = await fetch(url);

    const result = await response.json();

    // Save new cursor for next execution
    await paramManager.saveParameter('cursor', result.nextCursor);

    return result.items;
  }
}
```

**Use cases:**
- Saving state for next execution
- Tracking progress
- Persisting checkpoints

## Complete Examples

### Example 1: Static Parameters (Configuration)

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter
} from '@tunnelhub/sdk';

type User = {
  id: string;
  name: string;
  email: string;
};

class UserSyncIntegration extends DeltaIntegrationFlow<User> {
  private sourceApiKey: string;
  private targetApiKey: string;
  private useRetry: boolean;
  private batchSize: number;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name', 'email'], context);

    // Load static parameters from event
    this.sourceApiKey = AutomationParameter.getRequiredParameter(
      this.parameters,
      'source_api_key'
    );

    this.targetApiKey = AutomationParameter.getRequiredParameter(
      this.parameters,
      'target_api_key'
    );

    this.useRetry = AutomationParameter.getBooleanParameter(
      this.parameters,
      'retry_enabled',
      true
    );

    const batchSizeStr = AutomationParameter.getParameter(this.parameters, 'batch_size');
    this.batchSize = batchSizeStr ? parseInt(batchSizeStr) : 100;
  }

  protected async loadSourceSystemData(): Promise<User[]> {
    const response = await fetch('https://source.example.com/users', {
      headers: { 'Authorization': `Bearer ${this.sourceApiKey}` }
    });

    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<User[]> {
    const response = await fetch('https://target.example.com/users', {
      headers: { 'Authorization': `Bearer ${this.targetApiKey}` }
    });

    return await response.json();
  }

  protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
    return this.useRetry
      ? await this.insertWithRetry(item)
      : await this.insertWithoutRetry(item);
  }

  private async insertWithRetry(item: User): Promise<IntegrationMessageReturn> {
    for (let attempt = 1; attempt <= 3; attempt++) {
      try {
        const response = await fetch('https://target.example.com/users', {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${this.targetApiKey}`,
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(item)
        });

        if (!response.ok) throw new Error(`HTTP ${response.status}`);

        return { message: 'User created' };
      } catch (error) {
        if (attempt === 3) throw error;
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
      }
    }
  }

  private async insertWithoutRetry(item: User): Promise<IntegrationMessageReturn> {
    const response = await fetch('https://target.example.com/users', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.targetApiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(item)
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return { message: 'User created' };
  }

  protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
    // Similar to insertAction
    return { message: 'User updated' };
  }

  protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
    // Similar to insertAction
    return { message: 'User deleted' };
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      { fieldName: 'id', fieldLabel: 'User ID', fieldType: 'TEXT' },
      { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' },
      { fieldName: 'email', fieldLabel: 'Email', fieldType: 'TEXT' }
    ];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new UserSyncIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return { statusCode: 200, body: 'Integration completed successfully' };
};
```

### Example 2: Instance Parameters (Stateful)

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter
} from '@tunnelhub/sdk';

type Order = {
  orderId: string;
  customerEmail: string;
  amount: number;
  status: string;
  createdAt: Date;
};

class OrderSyncIntegration extends DeltaIntegrationFlow<Order> {
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['orderId'], ['status', 'amount'], context);

    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }

  protected async loadSourceSystemData(): Promise<Order[]> {
    // Get last sync date from DynamoDB
    const lastSyncDate = await this.paramManager.getParameter('last_sync_date', null);

    let url = 'https://ecommerce.example.com/api/orders';

    if (lastSyncDate) {
      url += `?modified_since=${encodeURIComponent(lastSyncDate)}`;
    }

    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'ecommerce_api_key');

    const response = await fetch(url, {
      headers: { 'Authorization': `Bearer ${apiKey}` }
    });

    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<Order[]> {
    const crmApiKey = AutomationParameter.getRequiredParameter(this.parameters, 'crm_api_key');

    const response = await fetch('https://crm.example.com/api/orders', {
      headers: { 'Authorization': `Bearer ${crmApiKey}` }
    });

    return await response.json();
  }

  protected async insertAction(item: Order): Promise<IntegrationMessageReturn> {
    const crmApiKey = AutomationParameter.getRequiredParameter(this.parameters, 'crm_api_key');

    const response = await fetch('https://crm.example.com/api/orders', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${crmApiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        externalId: item.orderId,
        customerEmail: item.customerEmail,
        amount: item.amount,
        status: item.status,
        createdAt: item.createdAt
      })
    });

    if (!response.ok) {
      throw new Error(`Failed to create order: ${response.statusText}`);
    }

    return { message: 'Order created in CRM' };
  }

  protected async updateAction(oldItem: Order, newItem: Order): Promise<IntegrationMessageReturn> {
    const crmApiKey = AutomationParameter.getRequiredParameter(this.parameters, 'crm_api_key');

    const response = await fetch(`https://crm.example.com/api/orders/${oldItem.externalId}`, {
      method: 'PATCH',
      headers: {
        'Authorization': `Bearer ${crmApiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify({
        status: newItem.status,
        amount: newItem.amount
      })
    });

    if (!response.ok) {
      throw new Error(`Failed to update order: ${response.statusText}`);
    }

    return { message: 'Order updated in CRM' };
  }

  protected async deleteAction(item: Order): Promise<IntegrationMessageReturn> {
    const crmApiKey = AutomationParameter.getRequiredParameter(this.parameters, 'crm_api_key');

    const response = await fetch(`https://crm.example.com/api/orders/${item.externalId}`, {
      method: 'DELETE',
      headers: { 'Authorization': `Bearer ${crmApiKey}` }
    });

    if (!response.ok) {
      throw new Error(`Failed to delete order: ${response.statusText}`);
    }

    return { message: 'Order deleted from CRM' };
  }

  protected async afterIntegration(): Promise<void> {
    // Save current sync date for next execution
    const currentDate = new Date().toISOString();
    await this.paramManager.saveParameter('last_sync_date', currentDate);

    console.log(`[OrderSync] Last sync date saved: ${currentDate}`);
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      { fieldName: 'orderId', fieldLabel: 'Order ID', fieldType: 'TEXT' },
      { fieldName: 'customerEmail', fieldLabel: 'Customer Email', fieldType: 'TEXT' },
      { fieldName: 'amount', fieldLabel: 'Amount', fieldType: 'NUMBER' },
      { fieldName: 'status', fieldLabel: 'Status', fieldType: 'TEXT' },
      { fieldName: 'createdAt', fieldLabel: 'Created At', fieldType: 'DATETIME' }
    ];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new OrderSyncIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return { statusCode: 200, body: 'Integration completed successfully' };
};
```

### Example 3: Mixed Usage (Static + Instance)

```typescript
import {
  NoDeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter
} from '@tunnelhub/sdk';

type LogEntry = {
  id: string;
  message: string;
  timestamp: Date;
  level: string;
};

class LogExportIntegration extends NoDeltaIntegrationFlow<LogEntry> {
  private targetApiKey: string;
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);

    // Static parameter: API key for target system
    this.targetApiKey = AutomationParameter.getRequiredParameter(
      this.parameters,
      'target_api_key'
    );

    // Instance parameter: Manage state between executions
    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }

  protected async loadSourceSystemData(): Promise<LogEntry[]> {
    // Get last processed log ID from previous execution
    const lastLogId = await this.paramManager.getParameter('last_log_id', null);

    const sourceApiKey = AutomationParameter.getRequiredParameter(this.parameters, 'source_api_key');

    let url = 'https://logs.example.com/api/entries';

    if (lastLogId) {
      url += `?after=${lastLogId}`;
    }

    const response = await fetch(url, {
      headers: { 'Authorization': `Bearer ${sourceApiKey}` }
    });

    const result = await response.json();

    // Save the latest log ID for next execution
    if (result.entries.length > 0) {
      const latestId = result.entries[result.entries.length - 1].id;
      await this.paramManager.saveParameter('last_log_id', latestId);
    }

    return result.entries;
  }

  protected async sendData(item: LogEntry): Promise<IntegrationMessageReturn> {
    const response = await fetch('https://target.example.com/api/logs', {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${this.targetApiKey}`,
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(item)
    });

    if (!response.ok) {
      throw new Error(`Failed to send log: ${response.statusText}`);
    }

    return { message: 'Log entry sent' };
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      { fieldName: 'id', fieldLabel: 'Log ID', fieldType: 'TEXT' },
      { fieldName: 'message', fieldLabel: 'Message', fieldType: 'TEXT' },
      { fieldName: 'timestamp', fieldLabel: 'Timestamp', fieldType: 'DATETIME' },
      { fieldName: 'level', fieldLabel: 'Level', fieldType: 'TEXT' }
    ];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new LogExportIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return { statusCode: 200, body: 'Export completed successfully' };
};
```

## Best Practices

### 1. Parameter Naming Conventions

Use descriptive, snake_case names for parameters:

```typescript
// Good
AutomationParameter.getParameter(this.parameters, 'api_key');
AutomationParameter.getParameter(this.parameters, 'last_sync_date');
AutomationParameter.getParameter(this.parameters, 'retry_enabled');
AutomationParameter.getParameter(this.parameters, 'batch_size');

// Avoid
AutomationParameter.getParameter(this.parameters, 'key');
AutomationParameter.getParameter(this.parameters, 'date');
AutomationParameter.getParameter(this.parameters, 'flag');
```

### 2. Type Conversion

Always convert parameter values to the appropriate type:

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    // String parameter
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'api_key');

    // Boolean parameter
    const enabled = AutomationParameter.getBooleanParameter(this.parameters, 'enabled', false);

    // Number parameter
    const limitStr = AutomationParameter.getParameter(this.parameters, 'limit');
    const limit = limitStr ? parseInt(limitStr) : 100;

    // Date parameter
    const dateStr = await paramManager.getParameter('last_date');
    const date = dateStr ? new Date(dateStr) : null;

    // JSON parameter
    const filtersStr = AutomationParameter.getParameter(this.parameters, 'filters');
    const filters = filtersStr ? JSON.parse(filtersStr) : {};
  }
}
```

### 3. Error Handling

Always handle parameter retrieval errors gracefully:

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    // Required parameter - let it throw
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'api_key');

    // Optional parameter - provide default
    const timeoutStr = AutomationParameter.getParameter(this.parameters, 'timeout');
    const timeout = timeoutStr ? parseInt(timeoutStr) : 30000;

    // Boolean parameter with sensible default
    const debugMode = AutomationParameter.getBooleanParameter(this.parameters, 'debug', false);

    // Use parameters
    // ...
  }
}
```

### 4. Parameter Validation

Validate parameter values before use:

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    const batchSizeStr = AutomationParameter.getParameter(this.parameters, 'batch_size');
    let batchSize = batchSizeStr ? parseInt(batchSizeStr) : 100;

    // Validate range
    if (batchSize < 1 || batchSize > 1000) {
      throw new Error('batch_size must be between 1 and 1000');
    }

    // Use validated parameter
    // ...
  }
}
```

### 5. Instance Parameter Management

Always save instance parameters in `afterIntegration()` to ensure they're persisted even if errors occur:

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }

  protected async afterIntegration(): Promise<void> {
    // Save state regardless of success/failure
    await this.paramManager.saveParameter('last_execution_time', new Date().toISOString());

    const errorCount = this['savedLogs'].filter(l => l.status === 'FAIL').length;
    await this.paramManager.saveParameter('error_count', errorCount.toString());

    console.log('[MyIntegration] State saved');
  }
}
```

## Common Use Cases

### Use Case 1: API Configuration

```typescript
class ApiIntegration extends DeltaIntegrationFlow<MyType> {
  private apiConfig: {
    baseUrl: string;
    apiKey: string;
    timeout: number;
    retries: number;
  };

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.apiConfig = {
      baseUrl: AutomationParameter.getRequiredParameter(this.parameters, 'api_base_url'),
      apiKey: AutomationParameter.getRequiredParameter(this.parameters, 'api_key'),
      timeout: parseInt(
        AutomationParameter.getParameter(this.parameters, 'api_timeout') || '30000'
      ),
      retries: parseInt(
        AutomationParameter.getParameter(this.parameters, 'api_retries') || '3'
      )
    };
  }
}
```

### Use Case 2: Feature Flags

```typescript
class FeatureAwareIntegration extends DeltaIntegrationFlow<MyType> {
  protected realtimeLoggingThreshold: number =
    AutomationParameter.getBooleanParameter(this.parameters, 'verbose_logging', false)
      ? 500
      : 100;

  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    const skipValidation = AutomationParameter.getBooleanParameter(
      this.parameters,
      'skip_validation',
      false
    );

    if (!skipValidation) {
      await this.validateItem(item);
    }

    return await this.insertItem(item);
  }
}
```

### Use Case 3: Incremental Sync

```typescript
class IncrementalSyncIntegration extends DeltaIntegrationFlow<MyType> {
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['updatedAt'], context);

    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const lastSyncDate = await this.paramManager.getParameter('last_sync_date', null);

    let url = 'https://api.example.com/items';
    if (lastSyncDate) {
      url += `?modified_since=${encodeURIComponent(lastSyncDate)}`;
    }

    const response = await fetch(url);
    return await response.json();
  }

  protected async afterIntegration(): Promise<void> {
    await this.paramManager.saveParameter(
      'last_sync_date',
      new Date().toISOString()
    );
  }
}
```

### Use Case 4: Cursor Pagination

```typescript
class CursorPaginationIntegration extends DeltaIntegrationFlow<MyType> {
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const cursor = await this.paramManager.getParameter('cursor', '0');

    const response = await fetch(`https://api.example.com/items?cursor=${cursor}`);
    const result = await response.json();

    // Save new cursor
    await this.paramManager.saveParameter('cursor', result.nextCursor);

    return result.items;
  }
}
```

### Use Case 5: Execution Metrics

```typescript
class MetricsAwareIntegration extends DeltaIntegrationFlow<MyType> {
  private paramManager: AutomationParameter;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.paramManager = new AutomationParameter(
      this.tenantId,
      this.automationId,
      this.environmentId
    );
  }

  protected async afterIntegration(): Promise<void> {
    const startTime = Date.now();

    // Calculate metrics
    const duration = Date.now() - startTime;
    const totalItems = this.sourceSystemData.length;
    const insertCount = this['insertQueue'].length;
    const updateCount = this['updateQueue'].length;
    const deleteCount = this['deleteQueue'].length;

    // Save metrics
    await this.paramManager.saveParameter('last_duration_ms', duration.toString());
    await this.paramManager.saveParameter('last_total_items', totalItems.toString());
    await this.paramManager.saveParameter('last_insert_count', insertCount.toString());
    await this.paramManager.saveParameter('last_update_count', updateCount.toString());
    await this.paramManager.saveParameter('last_delete_count', deleteCount.toString());

    console.log(`[Metrics] Duration: ${duration}ms, Items: ${totalItems}`);
  }
}
```
