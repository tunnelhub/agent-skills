# Integration Flows

This reference provides detailed information about the four integration flow types available in TunnelHub SDK.

## Overview

TunnelHub SDK provides four abstract base classes for building integration automations:

1. **DeltaIntegrationFlow** - Synchronization with change tracking
2. **BatchDeltaIntegrationFlow** - Delta tracking with batch processing
3. **NoDeltaIntegrationFlow** - Simple one-way transfer
4. **NoDeltaBatchIntegrationFlow** - Batch one-way transfer

## DeltaIntegrationFlow

### Purpose

Tracks and synchronizes changes between source and target systems. Maintains state between executions to detect
insertions, updates, and deletions.

### When to Use

- Bidirectional data synchronization
- Change detection and tracking
- State persistence between executions
- Complex data mapping with updates

### Architecture

The flow follows this pattern:

1. Load source system data (`loadSourceSystemData`)
2. Load target system data (`loadTargetSystemData`)
3. Compare data using key fields and delta fields
4. Queue operations: insert, update, delete, noDelta
5. Process queues with action methods
6. Save logs based on logging strategy

### Required Methods

#### loadSourceSystemData

```typescript
protected async loadSourceSystemData(): Promise<T[]>
```

Load data from the source system. This is the system you're reading data from.

**Return**: Array of data items from source system

**Example:**

```typescript
protected async loadSourceSystemData(): Promise<User[]> {
  const system = this.systems.find(s => s.internalName === 'salesforce_crm');

  const response = await fetch(system.parameters.url, {
    headers: {
      'Authorization': `Bearer ${system.parameters.authHeader}`
    }
  });

  return await response.json();
}
```

#### loadTargetSystemData

```typescript
protected async loadTargetSystemData(): Promise<T[]>
```

Load data from the target system. This is the system you're writing to or comparing against.

**Return**: Array of data items from target system

**Example:**

```typescript
protected async loadTargetSystemData(): Promise<User[]> {
  const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'erp_api_key');
  const response = await fetch('https://erp.example.com/api/users', {
    headers: {
      'Authorization': `Bearer ${apiKey}`
    }
  });

  return await response.json();
}
```

#### insertAction

```typescript
protected async insertAction(item: T): Promise<IntegrationMessageReturn>
```

Insert a new item into the target system. Called for each item that exists in source but not in target.

**Parameters:**

- `item: T` - The item to insert

**Return**: `IntegrationMessageReturn` with status and optional data

**Example:**

```typescript
protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
  try {
    const response = await fetch('https://erp.example.com/api/users', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        externalId: item.userId,
        name: item.name,
        email: item.email
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const result = await response.json();

    return {
      message: 'User created successfully',
      data: { id: result.id } // Generated ID from target system
    };
  } catch (error) {
    return {
      message: `Failed to create user: ${error.message}`,
      data: null
    };
  }
}
```

#### updateAction

```typescript
protected async updateAction(oldItem: T, newItem: T): Promise<IntegrationMessageReturn>
```

Update an existing item in the target system. Called when an item exists in both systems but delta fields have changed.

**Parameters:**

- `oldItem: T` - The previous version from target system
- `newItem: T` - The current version from source system

**Return**: `IntegrationMessageReturn` with status and optional data

**Example:**

```typescript
protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
  try {
    const externalId = oldItem.externalId; // Use oldItem for ID
    const response = await fetch(`https://erp.example.com/api/users/${externalId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        name: newItem.name,
        email: newItem.email
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return {
      message: 'User updated successfully',
      data: { id: externalId }
    };
  } catch (error) {
    return {
      message: `Failed to update user: ${error.message}`,
      data: null
    };
  }
}
```

#### deleteAction

```typescript
protected async deleteAction(item: T): Promise<IntegrationMessageReturn>
```

Delete an item from the target system. Called when an item exists in target but not in source.

**Parameters:**

- `item: T` - The item to delete (from target system)

**Return**: `IntegrationMessageReturn` with status

**Example:**

```typescript
protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
  try {
    const externalId = item.externalId;
    const response = await fetch(`https://erp.example.com/api/users/${externalId}`, {
      method: 'DELETE'
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    return {
      message: 'User deleted successfully',
      data: null
    };
  } catch (error) {
    return {
      message: `Failed to delete user: ${error.message}`,
      data: null
    };
  }
}
```

#### defineMetadata

```typescript
protected defineMetadata(): Array<Metadata>
```

Define metadata for integration logs. This describes the data structure for UI display and filtering.

**Return**: Array of `Metadata` objects

**Example:**

```typescript
protected defineMetadata(): Array<Metadata> {
  return [
    {
      fieldName: 'userId',
      fieldLabel: 'User ID',
      fieldType: 'TEXT'
    },
    {
      fieldName: 'name',
      fieldLabel: 'Full Name',
      fieldType: 'TEXT'
    },
    {
      fieldName: 'email',
      fieldLabel: 'Email Address',
      fieldType: 'TEXT'
    },
    {
      fieldName: 'status',
      fieldLabel: 'Active',
      fieldType: 'BOOLEAN'
    },
    {
      fieldName: 'lastLogin',
      fieldLabel: 'Last Login Date',
      fieldType: 'DATETIME'
    }
  ];
}
```

**Field types:**

- `TEXT` - String values
- `NUMBER` - Numeric values
- `DATE` - Date without time
- `DATETIME` - Date with time
- `BOOLEAN` - True/false values

**Important**: `fieldLabel` cannot be 'Action', 'Status', or 'Message' (reserved values).

### Optional Override Methods

These methods provide hooks to customize flow behavior. All have default implementations and can be overridden when
needed.

#### hasInvalidDelta

```typescript
protected hasInvalidDelta(sourceSystemItem: T, targetSystemItem: T): boolean
```

Custom validation to detect changes. Default compares all `deltaFields` values.

**Override when:** Need custom change detection logic.

#### preSourceSystemSearchSort

```typescript
protected async preSourceSystemSearchSort(): Promise<void>
```

Prepares source system data for fast indexed search. Default creates index by key fields.

#### preTargetSystemSearchSort

```typescript
protected async preTargetSystemSearchSort(): Promise<void>
```

Prepares target system data for fast indexed search. Default creates index by key fields.

#### searchItemInTargetSystem

```typescript
protected searchItemInTargetSystem(item: T): T | null
```

Find item in target system data. Default uses indexed lookup by key fields.

#### searchItemInSourceSystem

```typescript
protected searchItemInSourceSystem(item: T): T | null
```

Find item in source system data. Default uses indexed lookup by key fields.

#### preProcessingCustomerRoutines

```typescript
protected async preProcessingCustomerRoutines(): Promise<void>
```

Execute custom logic before processing operation queues.

#### postProcessingCustomerRoutines

```typescript
protected async postProcessingCustomerRoutines(): Promise<void>
```

Execute custom logic after processing operation queues.

#### fillGeneratedId

```typescript
protected fillGeneratedId(item: T, data: any): void
```

Populate auto-generated ID from target system after insert. Called before item is added to delta image.

**Example:**

```typescript
protected fillGeneratedId(item: User, data: any): void {
  if (data?.id) {
    item.externalId = data.id;
  }
}
```

#### fillExternalIdAfterUpdate

```typescript
protected fillExternalIdAfterUpdate(item: {oldData: T; newData: T}, data: any): void
```

Populate ID values from target system after update. Used when ID is not available in source data.

#### noDeltaAction

```typescript
protected async noDeltaAction(item: T): Promise<void>
```

Handle items that exist in both systems but have no delta field changes.

### Complete Example

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter,
} from '@tunnelhub/sdk';

type User = {
  userId: string;
  name: string;
  email: string;
  status: boolean;
  lastLogin?: Date;
};

class SalesforceToErpSync extends DeltaIntegrationFlow<User> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['userId'], ['name', 'email', 'status'], context);
  }

  protected async loadSourceSystemData(): Promise<User[]> {
    const system = this.systems.find(s => s.internalName === 'salesforce_crm');

    const response = await fetch(`${system.parameters.url}/users`, {
      headers: {
        Authorization: `Bearer ${system.parameters.authHeader}`,
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to load users: ${response.statusText}`);
    }

    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<User[]> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'erp_api_key');

    const response = await fetch('https://erp.example.com/api/users', {
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (!response.ok) {
      throw new Error(`Failed to load ERP users: ${response.statusText}`);
    }

    return await response.json();
  }

  protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'erp_api_key');

    try {
      const response = await fetch('https://erp.example.com/api/users', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          externalId: item.userId,
          name: item.name,
          email: item.email,
          active: item.status,
        }),
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const result = await response.json();

      return {
        message: 'User created successfully',
        data: {externalId: result.id},
      };
    } catch (error) {
      return {
        message: `Failed to create user: ${error.message}`,
        data: null,
      };
    }
  }

  protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'erp_api_key');

    try {
      const response = await fetch(`https://erp.example.com/api/users/${oldItem.externalId}`, {
        method: 'PATCH',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          name: newItem.name,
          email: newItem.email,
          active: newItem.status,
        }),
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return {
        message: 'User updated successfully',
        data: {externalId: oldItem.externalId},
      };
    } catch (error) {
      return {
        message: `Failed to update user: ${error.message}`,
        data: null,
      };
    }
  }

  protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'erp_api_key');

    try {
      const response = await fetch(`https://erp.example.com/api/users/${item.externalId}`, {
        method: 'DELETE',
        headers: {
          Authorization: `Bearer ${apiKey}`,
        },
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      return {
        message: 'User deleted successfully',
        data: null,
      };
    } catch (error) {
      return {
        message: `Failed to delete user: ${error.message}`,
        data: null,
      };
    }
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'userId', fieldLabel: 'User ID', fieldType: 'TEXT'},
      {fieldName: 'name', fieldLabel: 'Full Name', fieldType: 'TEXT'},
      {fieldName: 'email', fieldLabel: 'Email Address', fieldType: 'TEXT'},
      {fieldName: 'status', fieldLabel: 'Active', fieldType: 'BOOLEAN'},
      {fieldName: 'lastLogin', fieldLabel: 'Last Login', fieldType: 'DATETIME'},
    ];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new SalesforceToErpSync(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Integration completed successfully'};
};
```

## BatchDeltaIntegrationFlow

### Purpose

Extends DeltaIntegrationFlow with batch processing capabilities. Designed for high-volume datasets where individual API
calls are inefficient.

### When to Use

- Processing >1000 items per execution
- Bulk API operations available
- Need to reduce HTTP overhead
- Performance is critical

### Optional Override Methods

BatchDeltaIntegrationFlow inherits all optional override methods from DeltaIntegrationFlow:

- `hasInvalidDelta`, `preSourceSystemSearchSort`, `preTargetSystemSearchSort`
- `searchItemInTargetSystem`, `searchItemInSourceSystem`
- `preProcessingCustomerRoutines`, `postProcessingCustomerRoutines`
- `fillGeneratedId`, `fillExternalIdAfterUpdate`
- `noDeltaAction`

See DeltaIntegrationFlow section for details.

### Additional Required Methods

#### batchInsertAction

```typescript
protected abstract batchInsertAction(items: T[]): Promise<IntegrationMessageReturnBatch[]>
```

Insert multiple items in a single batch operation.

**Parameters:**

- `items: T[]` - Array of items to insert (batch size = `this.packageSize`)

**Return**: Array of `IntegrationMessageReturnBatch` with status for each item

**Example:**

```typescript
protected async batchInsertAction(items: Product[]): Promise<IntegrationMessageReturnBatch[]> {
  const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'api_key');

  try {
    const response = await fetch('https://api.example.com/products/batch', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({
        products: items.map(item => ({
          externalId: item.productId,
          name: item.name,
          price: item.price
        }))
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const result = await response.json();

    return items.map((item, index) => ({
      message: result.success[index] ? 'Product created' : result.errors[index],
      data: result.data[index] ? { externalId: result.data[index].id } : null,
      status: result.success[index] ? 'SUCCESS' : 'FAIL'
    }));
  } catch (error) {
    return items.map(() => ({
      message: `Batch insert failed: ${error.message}`,
      data: null,
      status: 'FAIL'
    }));
  }
}
```

#### batchUpdateAction

```typescript
protected abstract batchUpdateAction(
  oldItems: T[],
  newItems: T[]
): Promise<IntegrationMessageReturnBatch[]>
```

Update multiple items in a single batch operation.

**Parameters:**

- `oldItems: T[]` - Previous versions from target system
- `newItems: T[]` - Current versions from source system

**Return**: Array of `IntegrationMessageReturnBatch` with status for each item

**Example:**

```typescript
protected async batchUpdateAction(
  oldItems: Product[],
  newItems: Product[]
): Promise<IntegrationMessageReturnBatch[]> {
  const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'api_key');

  try {
    const response = await fetch('https://api.example.com/products/batch', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({
        products: newItems.map((item, index) => ({
          id: oldItems[index].externalId,
          name: item.name,
          price: item.price
        }))
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const result = await response.json();

    return newItems.map((item, index) => ({
      message: result.success[index] ? 'Product updated' : result.errors[index],
      data: result.data[index] ? { externalId: result.data[index].id } : null,
      status: result.success[index] ? 'SUCCESS' : 'FAIL'
    }));
  } catch (error) {
    return newItems.map(() => ({
      message: `Batch update failed: ${error.message}`,
      data: null,
      status: 'FAIL'
    }));
  }
}
```

#### batchDeleteAction

```typescript
protected abstract batchDeleteAction(items: T[]): Promise<IntegrationMessageReturnBatch[]>
```

Delete multiple items in a single batch operation.

**Parameters:**

- `items: T[]` - Array of items to delete

**Return**: Array of `IntegrationMessageReturnBatch` with status for each item

**Example:**

```typescript
protected async batchDeleteAction(items: Product[]): Promise<IntegrationMessageReturnBatch[]> {
  const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'api_key');

  try {
    const response = await fetch('https://api.example.com/products/batch', {
      method: 'DELETE',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${apiKey}`
      },
      body: JSON.stringify({
        ids: items.map(item => item.externalId)
      })
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const result = await response.json();

    return items.map((item, index) => ({
      message: result.success[index] ? 'Product deleted' : result.errors[index],
      data: null,
      status: result.success[index] ? 'SUCCESS' : 'FAIL'
    }));
  } catch (error) {
    return items.map(() => ({
      message: `Batch delete failed: ${error.message}`,
      data: null,
      status: 'FAIL'
    }));
  }
}
```

### Configuration: packageSize

```typescript
constructor(event: ProcessorPayload, keyFields: Array<keyof T>, deltaFields: Array<keyof T>, context?: LambdaContext) {
  super(event, keyFields, deltaFields, context);
  this.packageSize = 100; // Process 100 items per batch
}
```

**Default**: 1 **Typical values**: 50-500 (depends on API limits)

**Considerations:**

- Higher values = fewer HTTP calls, more memory usage
- Lower values = better error isolation, more HTTP calls
- Check API documentation for batch size limits

### Complete Example

```typescript
import {
  BatchDeltaIntegrationFlow,
  IntegrationMessageReturnBatch,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter,
} from '@tunnelhub/sdk';

type Product = {
  productId: string;
  name: string;
  price: number;
  externalId?: string;
};

class ProductSyncIntegration extends BatchDeltaIntegrationFlow<Product> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['productId'], ['name', 'price'], context);
    this.packageSize = 200;
  }

  protected async loadSourceSystemData(): Promise<Product[]> {
    const system = this.systems.find(s => s.internalName === 'inventory_system');
    const response = await fetch(`${system.parameters.url}/products`);
    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<Product[]> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'ecommerce_api_key');
    const response = await fetch('https://ecommerce.example.com/api/products', {
      headers: {Authorization: `Bearer ${apiKey}`},
    });
    return await response.json();
  }

  protected async batchInsertAction(items: Product[]): Promise<IntegrationMessageReturnBatch[]> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'ecommerce_api_key');

    const response = await fetch('https://ecommerce.example.com/api/products/batch', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${apiKey}`,
      },
      body: JSON.stringify({
        products: items.map(item => ({
          externalId: item.productId,
          name: item.name,
          price: item.price,
        })),
      }),
    });

    const result = await response.json();

    return items.map((item, index) => ({
      message: result.success[index] ? 'Product created' : result.errors[index],
      data: result.data[index] ? {externalId: result.data[index].id} : null,
      status: result.success[index] ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async batchUpdateAction(
    oldItems: Product[],
    newItems: Product[],
  ): Promise<IntegrationMessageReturnBatch[]> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'ecommerce_api_key');

    const response = await fetch('https://ecommerce.example.com/api/products/batch', {
      method: 'PATCH',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${apiKey}`,
      },
      body: JSON.stringify({
        products: newItems.map((item, index) => ({
          id: oldItems[index].externalId,
          name: item.name,
          price: item.price,
        })),
      }),
    });

    const result = await response.json();

    return newItems.map((item, index) => ({
      message: result.success[index] ? 'Product updated' : result.errors[index],
      data: result.data[index] ? {externalId: result.data[index].id} : null,
      status: result.success[index] ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async batchDeleteAction(items: Product[]): Promise<IntegrationMessageReturnBatch[]> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'ecommerce_api_key');

    const response = await fetch('https://ecommerce.example.com/api/products/batch', {
      method: 'DELETE',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${apiKey}`,
      },
      body: JSON.stringify({
        ids: items.map(item => item.externalId),
      }),
    });

    const result = await response.json();

    return items.map((item, index) => ({
      message: result.success[index] ? 'Product deleted' : result.errors[index],
      data: null,
      status: result.success[index] ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'productId', fieldLabel: 'Product ID', fieldType: 'TEXT'},
      {fieldName: 'name', fieldLabel: 'Product Name', fieldType: 'TEXT'},
      {fieldName: 'price', fieldLabel: 'Price', fieldType: 'NUMBER'},
    ];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new ProductSyncIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Integration completed successfully'};
};
```

## NoDeltaIntegrationFlow

### Purpose

Simple one-way data transfer without state tracking or change detection.

### When to Use

- One-time data exports
- Streaming data ingestion
- Simple extract-and-load scenarios
- No need for synchronization

### Required Methods

#### loadSourceSystemData

```typescript
protected async loadSourceSystemData(): Promise<T[]>
```

Load data from the source system.

**Return**: Array of data items

#### sendData

```typescript
protected async sendData(item: T): Promise<IntegrationMessageReturn>
```

Send a single item to the target system.

**Parameters:**

- `item: T` - The item to send

**Return**: `IntegrationMessageReturn` with status

#### defineMetadata

```typescript
protected defineMetadata(): Array<Metadata>
```

Define metadata for logs (same as Delta flows).

### Optional Override Methods

#### preProcessingCustomerRoutines

```typescript
protected async preProcessingCustomerRoutines(): Promise<void>
```

Execute custom logic before processing data queue.

#### postProcessingCustomerRoutines

```typescript
protected async postProcessingCustomerRoutines(): Promise<void>
```

Execute custom logic after processing data queue.

#### isKnownFastIntegration

```typescript
protected isKnownFastIntegration(): boolean
```

Override to indicate this integration is known to process quickly, which can affect logging strategy selection.

> **Note**: This method affects logging strategy. Only override under direct supervision of a 4success TunnelHub
> specialist.

### Complete Example

```typescript
import {
  NoDeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter,
} from '@tunnelhub/sdk';

type LogEvent = {
  eventId: string;
  eventType: string;
  timestamp: Date;
  data: any;
};

class ExportToAnalytics extends NoDeltaIntegrationFlow<LogEvent> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
  }

  protected async loadSourceSystemData(): Promise<LogEvent[]> {
    const system = this.systems.find(s => s.internalName === 'app_logs');
    const response = await fetch(`${system.parameters.url}/events`);
    return await response.json();
  }

  protected async sendData(item: LogEvent): Promise<IntegrationMessageReturn> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'analytics_api_key');

    try {
      await fetch('https://analytics.example.com/api/events', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify(item),
      });

      return {
        message: 'Event sent successfully',
        data: null,
      };
    } catch (error) {
      return {
        message: `Failed to send event: ${error.message}`,
        data: null,
      };
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

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new ExportToAnalytics(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Export completed successfully'};
};
```

## NoDeltaBatchIntegrationFlow

### Purpose

Extends NoDeltaIntegrationFlow with batch processing for high-volume one-way transfers.

### When to Use

- Bulk data exports
- High-volume data ingestion
- Performance-critical one-way transfers

### Optional Override Methods

NoDeltaBatchIntegrationFlow inherits optional override methods from NoDeltaIntegrationFlow:

- `preProcessingCustomerRoutines`, `postProcessingCustomerRoutines`
- `isKnownFastIntegration`

See NoDeltaIntegrationFlow section for details.

### Additional Required Method

#### batchSendData

```typescript
protected abstract batchSendData(items: T[]): Promise<IntegrationMessageReturnBatch[]>
```

Send multiple items in a single batch operation.

**Parameters:**

- `items: T[]` - Array of items to send

**Return**: Array of `IntegrationMessageReturnBatch` with status for each item

### Complete Example

```typescript
import {
  NoDeltaBatchIntegrationFlow,
  IntegrationMessageReturnBatch,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  AutomationParameter,
} from '@tunnelhub/sdk';

type ExportItem = {
  id: string;
  data: any;
};

class BulkExportIntegration extends NoDeltaBatchIntegrationFlow<ExportItem> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
    this.packageSize = 500;
  }

  protected async loadSourceSystemData(): Promise<ExportItem[]> {
    const system = this.systems.find(s => s.internalName === 'source_db');
    const response = await fetch(`${system.parameters.url}/items`);
    return await response.json();
  }

  protected async batchSendData(items: ExportItem[]): Promise<IntegrationMessageReturnBatch[]> {
    const apiKey = AutomationParameter.getRequiredParameter(this.parameters, 'target_api_key');

    try {
      const response = await fetch('https://target.example.com/api/items/batch', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${apiKey}`,
        },
        body: JSON.stringify({
          items: items,
        }),
      });

      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      }

      const result = await response.json();

      return items.map((item, index) => ({
        message: result.success[index] ? 'Item exported' : result.errors[index],
        data: null,
        status: result.success[index] ? 'SUCCESS' : 'FAIL',
      }));
    } catch (error) {
      return items.map(() => ({
        message: `Batch export failed: ${error.message}`,
        data: null,
        status: 'FAIL',
      }));
    }
  }

  protected defineMetadata(): Array<Metadata> {
    return [{fieldName: 'id', fieldLabel: 'Item ID', fieldType: 'TEXT'}];
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new BulkExportIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Export completed successfully'};
};
```

## Choosing the Right Flow

### Decision Tree

```
Need to track changes between executions?
├─ Yes → DeltaIntegrationFlow
│  └─ Processing >1000 items per execution?
│     ├─ Yes → BatchDeltaIntegrationFlow
│     └─ No → DeltaIntegrationFlow
└─ No → NoDeltaIntegrationFlow
   └─ Processing >1000 items per execution?
      ├─ Yes → NoDeltaBatchIntegrationFlow
      └─ No → NoDeltaIntegrationFlow
```

### Comparison

| Flow                        | Change Tracking | Batch Support | State Persistence | Use Case              |
| --------------------------- | --------------- | ------------- | ----------------- | --------------------- |
| DeltaIntegrationFlow        | ✓               | ✗             | ✓                 | Bidirectional sync    |
| BatchDeltaIntegrationFlow   | ✓               | ✓             | ✓                 | High-volume sync      |
| NoDeltaIntegrationFlow      | ✗               | ✗             | ✗                 | One-way transfer      |
| NoDeltaBatchIntegrationFlow | ✗               | ✓             | ✗                 | Bulk one-way transfer |

### Performance Considerations

**Individual operations (Delta/NoDelta)**:

- Lower memory usage
- Better error isolation
- Higher HTTP overhead
- Slower for large datasets

**Batch operations (BatchDelta/NoDeltaBatch)**:

- Higher memory usage
- Lower HTTP overhead
- Faster for large datasets
- Batch-wide error impact

Choose batch when:

- API supports bulk operations
- Dataset is consistently large (>1000 items)
- Network latency is significant
- Memory is not a constraint

Choose individual when:

- API only supports single operations
- Error isolation is critical
- Dataset is small (<500 items)
- Memory is constrained
