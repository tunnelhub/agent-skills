---
name: tunnelhub-sdk
description:
  This skill should be used when creating, modifying, or maintaining integrations using the @tunnelhub/sdk package. It
  provides specialized guidance for implementing data synchronization flows, managing parameters, handling logging
  strategies, and working with AWS infrastructure components (DynamoDB, S3, Firehose, SQLite on EFS). Use this skill when
  working with integration flows (Delta, Batch Delta, No Delta, No Delta Batch), parameter management, data stores,
  conversion tables including CRUD operations, SQL Tables/Tabelas de Apoio, sequences, HTTP interceptors, or testing
  integration code.
---

# TunnelHub SDK

## Overview

This skill provides specialized guidance for developers working with the @tunnelhub/sdk package to create and maintain
data integration automations. It covers implementation of integration flows, parameter management, data operations,
SQL Tables/Tabelas de Apoio, logging strategies, testing patterns, and common utilities.

## When to Use

Use this skill when:

- Creating new integration automations using TunnelHub SDK
- Implementing or modifying integration flows (Delta, Batch Delta, No Delta, No Delta Batch)
- Managing integration parameters (static or dynamic)
- Working with data stores, conversion tables, SQL Tables/Tabelas de Apoio, or sequences
- Configuring logging strategies (realtime vs batch)
- Testing integration code
- Debugging existing integrations
- Configuring HTTP interceptors for API calls

## Choosing the Right Integration Flow

TunnelHub SDK provides four integration flow types. Choose based on synchronization requirements and data volume:

### DeltaIntegrationFlow

Use when you need bidirectional synchronization with change tracking.

**When to use:**

- Synchronize data between source and target systems
- Track and apply insert, update, and delete operations
- Maintain state between executions
- Detect changes based on delta fields

**Required methods:**

```typescript
protected async loadSourceSystemData(): Promise<T[]>
protected async loadTargetSystemData(): Promise<T[]>
protected async insertAction(item: T): Promise<IntegrationMessageReturn>
protected async updateAction(oldItem: T, newItem: T): Promise<IntegrationMessageReturn>
protected async deleteAction(item: T): Promise<IntegrationMessageReturn>
protected defineMetadata(): Array<Metadata>
```

**Example:**

```typescript
class UserSyncIntegration extends DeltaIntegrationFlow<User> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['userId'], ['name', 'email', 'status'], context);
  }

  protected async loadSourceSystemData(): Promise<User[]> {
    // Load data from source system
    const system = this.systems.find(s => s.internalName === 'source_system');
    return await fetchUsersFromSystem(system);
  }

  protected async loadTargetSystemData(): Promise<User[]> {
    // Load data from target system
    const system = this.systems.find(s => s.internalName === 'target_system');
    return await fetchUsersFromSystem(system);
  }

  protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
    try {
      const result = await createUserInTarget(item);
      return {success: true, message: 'User created', data: result};
    } catch (error) {
      return {success: false, message: error.message};
    }
  }

  protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
    try {
      const result = await updateUserInTarget(newItem);
      return {success: true, message: 'User updated', data: result};
    } catch (error) {
      return {success: false, message: error.message};
    }
  }

  protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
    try {
      await deleteUserInTarget(item);
      return {success: true, message: 'User deleted'};
    } catch (error) {
      return {success: false, message: error.message};
    }
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'userId', fieldLabel: 'User ID', fieldType: 'TEXT'},
      {fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT'},
      {fieldName: 'email', fieldLabel: 'Email', fieldType: 'TEXT'},
      {fieldName: 'status', fieldLabel: 'Status', fieldType: 'BOOLEAN'},
    ];
  }
}
```

### BatchDeltaIntegrationFlow

Use when processing large datasets with delta tracking.

**When to use:**

- High-volume data synchronization (>1000 items)
- Need batch operations for performance
- Want to reduce API calls
- Have delta tracking requirements

**Additional required methods (extends DeltaIntegrationFlow):**

```typescript
protected async batchInsertAction(items: T[]): Promise<IntegrationMessageReturnBatch[]>
protected async batchUpdateAction(oldItems: T[], newItems: T[]): Promise<IntegrationMessageReturnBatch[]>
protected async batchDeleteAction(items: T[]): Promise<IntegrationMessageReturnBatch[]>
```

**Configuration:**

```typescript
constructor(event: ProcessorPayload, keyFields: Array<keyof T>, deltaFields: Array<keyof T>, context?: LambdaContext) {
  super(event, keyFields, deltaFields, context);
  this.packageSize = 100; // Set batch size
}
```

**Example:**

```typescript
class ProductSyncIntegration extends BatchDeltaIntegrationFlow<Product> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['productId'], ['name', 'price', 'stock'], context);
    this.packageSize = 200;
  }

  protected async batchInsertAction(items: Product[]): Promise<IntegrationMessageReturnBatch[]> {
    const results = await bulkCreateProducts(items);
    return results.map(r => ({
      message: r.success ? 'Product created' : r.error,
      data: r.data,
      status: r.success ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async batchUpdateAction(
    oldItems: Product[],
    newItems: Product[],
  ): Promise<IntegrationMessageReturnBatch[]> {
    const results = await bulkUpdateProducts(newItems);
    return results.map(r => ({
      message: r.success ? 'Product updated' : r.error,
      data: r.data,
      status: r.success ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async batchDeleteAction(items: Product[]): Promise<IntegrationMessageReturnBatch[]> {
    const results = await bulkDeleteProducts(items);
    return results.map(r => ({
      message: r.success ? 'Product deleted' : r.error,
      data: r.data,
      status: r.success ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async loadSourceSystemData(): Promise<Product[]> {
    /* ... */
  }
  protected async loadTargetSystemData(): Promise<Product[]> {
    /* ... */
  }
  protected defineMetadata(): Array<Metadata> {
    /* ... */
  }
}
```

### NoDeltaIntegrationFlow

Use for simple one-way data transfer without state tracking.

**When to use:**

- One-time or streaming data transfers
- No need for change detection
- Simple data extraction and loading
- Stateless execution

**Required methods:**

```typescript
protected async loadSourceSystemData(): Promise<T[]>
protected async sendData(item: T): Promise<IntegrationMessageReturn>
protected defineMetadata(): Array<Metadata>
```

**Example:**

```typescript
class ExportToAnalytics extends NoDeltaIntegrationFlow<AnalyticsEvent> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
  }

  protected async loadSourceSystemData(): Promise<AnalyticsEvent[]> {
    const system = this.systems.find(s => s.internalName === 'source_system');
    return await fetchEventsFromSystem(system);
  }

  protected async sendData(item: AnalyticsEvent): Promise<IntegrationMessageReturn> {
    try {
      await sendToAnalytics(item);
      return {success: true, message: 'Event sent'};
    } catch (error) {
      return {success: false, message: error.message};
    }
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'eventId', fieldLabel: 'Event ID', fieldType: 'TEXT'},
      {fieldName: 'eventType', fieldLabel: 'Event Type', fieldType: 'TEXT'},
      {fieldName: 'timestamp', fieldLabel: 'Timestamp', fieldType: 'DATETIME'},
    ];
  }
}
```

### NoDeltaBatchIntegrationFlow

Use for batch one-way data transfer without delta tracking.

**When to use:**

- High-volume one-way transfers
- Need batch processing
- No state persistence required

**Additional required method (extends NoDeltaIntegrationFlow):**

```typescript
protected async batchSendData(items: T[]): Promise<IntegrationMessageReturnBatch[]>
```

**Example:**

```typescript
class BulkExportIntegration extends NoDeltaBatchIntegrationFlow<ExportItem> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
    this.packageSize = 500;
  }

  protected async batchSendData(items: ExportItem[]): Promise<IntegrationMessageReturnBatch[]> {
    const results = await bulkExportItems(items);
    return results.map(r => ({
      message: r.success ? 'Item exported' : r.error,
      data: r.data,
      status: r.success ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async loadSourceSystemData(): Promise<ExportItem[]> {
    /* ... */
  }
  protected defineMetadata(): Array<Metadata> {
    /* ... */
  }
}
```

## Constructor Pattern

All integration flows follow a consistent constructor pattern:

```typescript
constructor(
  event: ProcessorPayload,
  keyFields?: Array<keyof T>,
  deltaFields?: Array<keyof T>,
  context?: LambdaContext,
  debug?: boolean
) {
  super(event, keyFields, deltaFields, context, debug);
  // Custom initialization here
}
```

**Parameters:**

- `event: ProcessorPayload` - Execution event containing tenantId, automationId, executionId, etc.
- `keyFields: Array<keyof T>` - Fields used for record identification (Delta flows only)
- `deltaFields: Array<keyof T>` - Fields monitored for changes (Delta flows only)
- `context?: LambdaContext` - Lambda execution context (optional)
- `debug?: boolean` - Enable debug mode (optional)

## Key Fields and Delta Fields

**Key Fields**: Fields used to uniquely identify records for matching between source and target systems.

```typescript
// Using a single key field
['userId'][
  // Using multiple key fields for composite keys
  ('companyId', 'employeeId')
];
```

**Delta Fields**: Fields that trigger update operations when changed.

```typescript
// Monitor specific fields for changes
['name', 'email', 'status'];

// Monitor all fields except key fields
Object.keys(item).filter(f => !keyFields.includes(f));
```

## Parameter Management

### Static Methods (use within integration flows)

```typescript
import {AutomationParameter} from '@tunnelhub/sdk';

// Get parameter value (returns string | null)
const apiKey = AutomationParameter.getParameter(this.parameters, 'api_key');

// Get required parameter (throws error if not found)
const secretKey = AutomationParameter.getRequiredParameter(this.parameters, 'secret_key');

// Get boolean parameter (returns boolean)
const useRetry = AutomationParameter.getBooleanParameter(this.parameters, 'retry_enabled', false);
```

### Instance Methods (for dynamic persistence)

```typescript
// Create parameter manager
const paramManager = new AutomationParameter(this.tenantId, this.automationId, this.environmentId);

// Get parameter from DynamoDB
const lastSync = await paramManager.getParameter('last_sync_date', null);

// Save parameter to DynamoDB
await paramManager.saveParameter('last_sync_date', new Date().toISOString());
```

**Use cases:**

- Static methods: Access configuration from `this.parameters` (passed in event)
- Instance methods: Persist and retrieve values between executions

## Data Operations

### DataStore & ConversionTable

Use `ConversionTable` to read mapped values and `DataStore` to read or maintain conversion table items.

```typescript
import {DataStore, ConversionTable} from '@tunnelhub/sdk';

const dataStore = new DataStore(this.executionEvent);
const conversionTable = new ConversionTable(dataStore);

// Get all items from conversion table
const allMappings = await conversionTable.getValuesFromConversionTable('departments');

// Get the mapped value for a source value
const departmentId = await conversionTable.getValueFromConversionTable('departments', 'SALES');

// Create, update, or delete conversion table items directly
const createdItem = await dataStore.createConversionTableItem({
  externalCode: 'departments',
  fromValue: 'SALES',
  toValue: 'DEPT-001',
});

await dataStore.updateConversionTableItem({
  externalCode: 'departments',
  itemUuid: createdItem.uuid,
  toValue: 'DEPT-010',
});

await dataStore.deleteConversionTableItem({
  externalCode: 'departments',
  itemUuid: createdItem.uuid,
});
```

### Sequences

Generate sequential IDs atomically.

```typescript
import {Sequences} from '@tunnelhub/sdk';

const sequences = new Sequences(this.executionEvent);

// Get next value from sequence
const nextId = await sequences.getSequenceNextValue('customer_sequence');
// Returns: 1001, 1002, 1003, ...
```

### SQL Tables / Tabelas de Apoio

Use `SqlTables` for structured relational support data associated with the automation. Prefer SQL Tables when the data
has typed columns, a primary key, pagination, filters, CSV import/export in the platform, or must be visible and
maintainable in the product. Keep using `DataStore`/`ConversionTable` for simple from/to mappings.

```typescript
import {SqlTables} from '@tunnelhub/sdk';

const sqlTables = new SqlTables(this.executionEvent);

const inserted = await sqlTables.insertRow('customer-cache', {
  document: '12345678900',
  name: 'Customer example',
  active: true,
});

const page = await sqlTables.queryRows('customer-cache', {
  current: 1,
  pageSize: 20,
  filter: {
    document: ['12345678900'],
  },
});

await sqlTables.updateRow('customer-cache', inserted.rowId, {
  name: 'Updated customer',
});

await sqlTables.deleteRow('customer-cache', inserted.rowId);
```

SQL Tables require managed runtime support: `runtime: nodejs24.x`, `runInVpc: true`,
`configuration.sqlTables.enabled: true`, `TH_SQL_TABLES_ENABLED=true`, and the SQL Tables EFS mount provided by
TunnelHub. For the full API contract, validation behavior, runtime requirements, and troubleshooting, see
`references/sql-tables.md`.

### System Configuration

Systems are already available in the `this.systems` array in all integration flows.

```typescript
// Find system by internal name
const crmSystem = this.systems.find(s => s.internalName === 'salesforce_crm');

if (crmSystem) {
  const apiUrl = crmSystem.parameters.url;
  const authType = crmSystem.parameters.authType;

  // Access system-type-specific parameters
  if (crmSystem.type === 'HTTP' && crmSystem.parameters.authType === 'BASIC') {
    const username = crmSystem.parameters.user;
    const password = crmSystem.parameters.password;
  }
}
```

## Utilities

### Promise Utilities

```typescript
import {promiseAllSettled, promiseWithConcurrency} from '@tunnelhub/sdk';

// Execute multiple promises, ignore failures
const results = await promiseAllSettled([this.insertAction(item1), this.insertAction(item2), this.insertAction(item3)]);

results.forEach(r => {
  if (r.status === 'fulfilled') {
    console.log('Success:', r.value);
  } else {
    console.error('Failed:', r.reason);
  }
});

// Process array with concurrency control
const stream = promiseWithConcurrency({
  array: largeDataset,
  mapFunction: async item => {
    return await processItem(item);
  },
  options: {concurrency: 10},
});

for await (const result of stream) {
  console.log(result);
}
```

### Validation Utilities

```typescript
import {validateMetadata, validateMetadataArray} from '@tunnelhub/sdk';

// Validate single metadata
validateMetadata({
  fieldName: 'id',
  fieldLabel: 'ID',
  fieldType: 'TEXT',
});

// Validate array of metadata
validateMetadataArray([
  {fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT'},
  {fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT'},
]);
```

**Important**: `fieldLabel` cannot be 'Action', 'Status', or 'Message' (reserved values).

### HTTP Interceptor

Configure automatic HTTP request/response logging.

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor({
  enableGlobalFetch: true,
  enableUndiciFetch: true,
});
```

Place this at the top of your integration file to intercept all HTTP calls.

## Logging Strategy

The SDK automatically chooses between realtime (DynamoDB) and batch (Firehose) logging based on data volume.

**Default behavior:**

- ≤100 items → Realtime logging
- > 1000 items → Batch logging (DynamoDB protection)
- 100-500 items with ≥80% noDelta ratio → Realtime (fast integration optimization)

**Customize thresholds:**

```typescript
class CustomIntegration extends DeltaIntegrationFlow<MyType> {
  protected realtimeLoggingThreshold: number = 200;
  protected maxRealtimeItems: number = 2000;
  protected highNoDeltaRatioThreshold: number = 0.9; // 90%
}
```

For detailed configuration and examples, see `references/logging-strategy.md`.

## Executing the Integration

```typescript
const integration = new MyIntegration(event, context);

try {
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    console.log('Integration completed with errors');
  } else {
    console.log('Integration completed successfully');
  }
} catch (error) {
  console.error('Integration failed:', error);
}
```

## Error Handling Pattern

```typescript
protected async insertAction(item: T): Promise<IntegrationMessageReturn> {
  try {
    const result = await apiCall(item);
    return {
      message: 'Success',
      data: result
    };
  } catch (error) {
    SDK.log(`Insert failed for item: ${JSON.stringify(item)}`);
    return {
      message: error.message,
      data: null
    };
  }
}
```

## SDK Helper Methods

```typescript
// Check if integration has errors
if (this.hasAnyErrors()) {
  console.log('Errors detected during execution');
}

// Enable verbose logging
SDK.verbose = true;

// Log messages (only when verbose is true)
SDK.log('Debug information');
```

## Testing Pattern

```typescript
import {SDK} from '@tunnelhub/sdk';

// Enable test mode
SDK.testMode = true;

// Create test integration
const testEvent = {
  tenantId: 'test-tenant',
  environmentId: 'test-env',
  executionId: 'test-exec',
  automationId: 'test-auto',
  expirationPeriod: 30,
  systems: [],
  parameters: {},
} as ProcessorPayload;

const integration = new MyIntegration(testEvent);
await integration.doIntegration();

expect(integration.hasAnyErrors()).toBeFalsy();
```

## System Types Reference

TunnelHub supports these system types:

- **DATABASE**: MySQL, PostgreSQL, Oracle, MSSQL, Redshift, MongoDB
- **HTTP**: REST APIs with various auth types (NONE, BASIC, QUERY_STRING, HEADERS)
- **FTP/SFTP**: File transfer with encryption options
- **LDAP**: Directory services
- **MAIL**: Email servers
- **SAPRFC**: SAP RFC connections
- **SMB**: Windows file shares
- **SOAP**: SOAP web services

Access system parameters via `this.systems` in integration flows.

## Resources

This skill includes detailed reference documentation for specific topics:

- **references/integration-flows.md** - Complete details on all four flow types, methods, and examples
- **references/logging-strategy.md** - Smart logging strategy configuration and optimization
- **references/parameters-management.md** - Parameter management patterns and use cases
- **references/data-operations.md** - DataStore read/write operations, conversion tables, SQL Tables, sequences, system configuration
- **references/sql-tables.md** - SQL Tables/Tabelas de Apoio helper, runtime requirements, row operations, validation, troubleshooting
- **references/utilities.md** - Promise utilities, validations, and helpers
- **references/http-interceptor.md** - HTTP request/response logging configuration
- **references/system-configuration.md** - System types and parameter structures
- **references/testing-patterns.md** - Testing patterns, mocking, and test data generation

Load these reference files when you need detailed information on specific topics.
