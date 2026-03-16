# Utilities

This reference explains how to use utility functions in TunnelHub SDK for promises, validations, and other common operations.

## Overview

TunnelHub SDK provides these utilities:

1. **Promise Utilities** - `promiseAllSettled`, `promiseWithConcurrency`
2. **Validation Utilities** - `validateMetadata`, `validateMetadataArray`
3. **SDK Helpers** - `SDK.log`, `SDK.verbose`, `SDK.testMode`

## Promise Utilities

### promiseAllSettled

```typescript
export const promiseAllSettled = (promises: Promise<any>[]): Promise<PromiseSettledResult[]>
```

Execute multiple promises and return their settled status, regardless of success or failure. This is useful when you want to continue processing even if some promises fail.

**Parameters:**
- `promises`: Array of promises to execute

**Returns:**
- `Promise<PromiseSettledResult[]>` - Array of results with status and value/reason

**Result Structure:**
```typescript
interface PromiseFulfilledResult {
  status: 'fulfilled';
  value: any;
}

interface PromiseRejectedResult {
  status: 'rejected';
  reason: any;
}

type PromiseSettledResult = PromiseFulfilledResult | PromiseRejectedResult;
```

**Example:**
```typescript
import { promiseAllSettled } from '@tunnelhub/sdk';

class UserSyncIntegration extends DeltaIntegrationFlow<User> {
  protected async processInsertQueue(): Promise<void> {
    // Process all inserts, continue even if some fail
    const promises = this.insertQueue.map(item => this.insertAction(item));

    const results = await promiseAllSettled(promises);

    let successCount = 0;
    let failureCount = 0;

    results.forEach((result, index) => {
      const item = this.insertQueue[index];

      if (result.status === 'fulfilled') {
        successCount++;
        console.log(`✓ Inserted user ${item.userId}`);
      } else {
        failureCount++;
        console.error(`✗ Failed to insert user ${item.userId}:`, result.reason);
      }
    });

    console.log(`Processed ${successCount} successful, ${failureCount} failed`);

    if (failureCount > 0) {
      this.errorIndicator = true;
    }
  }
}
```

**Use Cases:**
- Batch operations where individual failures shouldn't stop the entire batch
- Parallel processing with error handling
- Collecting results from multiple independent operations

**Advantages over Promise.all:**
- Doesn't reject on first failure
- Executes all promises regardless of individual results
- Provides detailed error information

### promiseWithConcurrency

```typescript
export const promiseWithConcurrency = <T, R>(params: {
  array: T[];
  mapFunction: (item: T, options?: Pick<ArrayOptions, 'signal'>) => Promise<R>;
  options: ArrayOptions;
}): Readable<R>
```

Process an array of items concurrently using Node.js streams. This provides controlled concurrency with backpressure handling.

**Parameters:**
- `array`: Array of items to process
- `mapFunction`: Async function to transform each item
- `options`: Stream processing options, including `concurrency` and `signal`

**Returns:**
- `Readable<R>` - Stream of transformed items

**Options:**
```typescript
interface ArrayOptions {
  concurrency?: number;  // Number of concurrent operations (default: varies by Node version)
  signal?: AbortSignal;  // Signal for cancellation
}
```

**Example:**
```typescript
import { promiseWithConcurrency } from '@tunnelhub/sdk';

class LargeDataSync extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    // Large dataset - process with controlled concurrency
    const allItems = [];

    const stream = promiseWithConcurrency({
      array: hugeDataset,
      mapFunction: async (item) => {
        return await this.fetchAndTransformItem(item);
      },
      options: { concurrency: 10 } // Process 10 items at a time
    });

    for await (const item of stream) {
      allItems.push(item);
    }

    return allItems;
  }
}
```

**Example with Pagination:**
```typescript
class PaginatedSync extends DeltaIntegrationFlow<MyType> {
  protected async loadSourceSystemData(): Promise<MyType[]> {
    let page = 1;
    let hasMore = true;
    const allItems = [];

    while (hasMore) {
      // Fetch pages concurrently
      const pagePromises = Array.from({ length: 5 }, (_, i) =>
        this.fetchPage(page + i)
      );

      const results = await promiseAllSettled(pagePromises);

      for (const result of results) {
        if (result.status === 'fulfilled' && result.value.items.length > 0) {
          allItems.push(...result.value.items);
        } else {
          hasMore = false;
          break;
        }
      }

      page += 5;

      if (allItems.length >= 10000) {
        hasMore = false;
      }
    }

    return allItems;
  }
}
```

**Example with Cancellation:**
```typescript
class CancelableSync extends DeltaIntegrationFlow<MyType> {
  private abortController = new AbortController();

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const allItems = [];

    const stream = promiseWithConcurrency({
      array: hugeDataset,
      mapFunction: async (item, options) => {
        if (options?.signal?.aborted) {
          return null;
        }

        return await this.processItem(item);
      },
      options: {
        concurrency: 20,
        signal: this.abortController.signal
      }
    });

    try {
      for await (const item of stream) {
        if (item !== null) {
          allItems.push(item);
        }
      }
    } catch (error) {
      if (error.name !== 'AbortError') {
        throw error;
      }
    }

    return allItems;
  }

  protected async beforeIntegration(): Promise<void> {
    // Cancel if running too long
    setTimeout(() => {
      this.abortController.abort();
    }, 300000); // 5 minutes
  }
}
```

**Use Cases:**
- Processing large datasets with memory efficiency
- Controlled concurrency to avoid overwhelming APIs
- Batch processing with backpressure
- Cancellable operations

**Advantages:**
- Memory efficient (streams vs arrays)
- Backpressure handling
- Configurable concurrency
- Cancellation support

## Validation Utilities

### validateMetadata

```typescript
export function validateMetadata(metadata: Metadata): void
```

Validate a single Metadata object to ensure it has valid fieldLabel values.

**Parameters:**
- `metadata`: Metadata object to validate

**Throws:**
- `Error` if fieldLabel is required or prohibited

**Prohibited Values:**
`fieldLabel` cannot be 'Action', 'Status', or 'Message' (reserved for UI).

**Example:**
```typescript
import { validateMetadata } from '@tunnelhub/sdk';

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected defineMetadata(): Array<Metadata> {
    const metadata = [
      { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
      { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
    ];

    // Validate all metadata items
    metadata.forEach(validateMetadata);

    return metadata;
  }
}
```

**Invalid Examples:**
```typescript
// These will throw errors
validateMetadata({ fieldName: 'id', fieldLabel: 'Action', fieldType: 'TEXT' });  // Prohibited
validateMetadata({ fieldName: 'id', fieldLabel: 'Status', fieldType: 'TEXT' });  // Prohibited
validateMetadata({ fieldName: 'id', fieldLabel: 'Message', fieldType: 'TEXT' }); // Prohibited
```

### validateMetadataArray

```typescript
export function validateMetadataArray(metadataArray: Metadata[]): void
```

Validate an array of Metadata objects.

**Parameters:**
- `metadataArray`: Array of metadata objects to validate

**Throws:**
- `Error` if input is not an array or any item is invalid

**Example:**
```typescript
import { validateMetadataArray } from '@tunnelhub/sdk';

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected defineMetadata(): Array<Metadata> {
    const metadata = [
      { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
      { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' },
      { fieldName: 'email', fieldLabel: 'Email', fieldType: 'TEXT' },
      { fieldName: 'active', fieldLabel: 'Active', fieldType: 'BOOLEAN' }
    ];

    // Validate entire array
    validateMetadataArray(metadata);

    return metadata;
  }
}
```

**Custom Error Handling:**
```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected defineMetadata(): Array<Metadata> {
    const metadata = [
      { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
      { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
    ];

    try {
      validateMetadataArray(metadata);
      return metadata;
    } catch (error) {
      console.error('Invalid metadata:', error.message);
      // Provide fallback metadata
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' }
      ];
    }
  }
}
```

**Use Cases:**
- Validate metadata before creating integration
- Ensure UI compatibility
- Prevent runtime errors from invalid field labels
- Data structure validation

## SDK Helpers

### SDK.log

```typescript
SDK.log(message: string): void
```

Log a message to console. Only outputs when `SDK.verbose` is `true`.

**Parameters:**
- `message`: Message to log

**Example:**
```typescript
import { SDK } from '@tunnelhub/sdk';

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
    SDK.log(`Inserting item: ${JSON.stringify(item)}`);

    try {
      const result = await fetch('https://api.example.com/items', {
        method: 'POST',
        body: JSON.stringify(item)
      });

      SDK.log(`Insert successful: ${item.id}`);

      return { message: 'Item created', data: result };
    } catch (error) {
      SDK.log(`Insert failed for item ${item.id}: ${error.message}`);

      return { message: error.message, data: null };
    }
  }
}
```

### SDK.verbose

```typescript
SDK.verbose: boolean
```

Enable or disable verbose logging. When `true`, `SDK.log()` outputs to console. When `false`, no output.

**Default**: `false`

**Example:**
```typescript
import { SDK } from '@tunnelhub/sdk';

// Enable verbose logging for debugging
SDK.verbose = true;

class MyIntegration extends DeltaIntegrationFlow<MyType> {
  protected async beforeIntegration(): Promise<void> {
    SDK.log(`Starting integration for tenant ${this.tenantId}`);
    SDK.log(`Expected items: ${this.sourceSystemData.length}`);
  }
}
```

**Dynamic Control:**
```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    // Enable verbose mode based on parameter
    SDK.verbose = AutomationParameter.getBooleanParameter(
      this.parameters,
      'debug_mode',
      false
    );
  }
}
```

### SDK.testMode

```typescript
SDK.testMode: boolean
```

Indicate whether the integration is running in test mode. Used for testing and mocking.

**Default**: `false`

**Example:**
```typescript
import { SDK } from '@tunnelhub/sdk';

SDK.testMode = true; // Enable test mode

const testEvent = {
  tenantId: 'test-tenant',
  environmentId: 'test-env',
  executionId: 'test-exec',
  automationId: 'test-auto',
  expirationPeriod: 30,
  systems: [],
  parameters: {}
} as ProcessorPayload;

const integration = new MyIntegration(testEvent);
await integration.doIntegration();
```

## Complete Examples

### Example 1: Concurrent Processing with Error Handling

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  promiseAllSettled
} from '@tunnelhub/sdk';

type Order = {
  orderId: string;
  customerId: string;
  amount: number;
};

class OrderSyncIntegration extends DeltaIntegrationFlow<Order> {
  protected async insertAction(item: Order): Promise<IntegrationMessageReturn> {
    const response = await fetch('https://erp.example.com/api/orders', {
      method: 'POST',
      body: JSON.stringify(item)
    });

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}`);
    }

    return { message: 'Order created' };
  }

  protected async processInsertQueue(): Promise<void> {
    // Process all orders concurrently
    const promises = this.insertQueue.map(item =>
      this.insertAction(item)
    );

    const results = await promiseAllSettled(promises);

    // Analyze results
    const successful = results.filter(r => r.status === 'fulfilled');
    const failed = results.filter(r => r.status === 'rejected');

    console.log(`Processed ${successful.length} orders successfully`);

    if (failed.length > 0) {
      console.error(`Failed to process ${failed.length} orders:`);

      failed.forEach((result, index) => {
        const item = this.insertQueue[successful.length + index];
        console.error(`  - ${item.orderId}: ${result.reason.message}`);
      });

      this.errorIndicator = true;
    }
  }

  protected async loadSourceSystemData(): Promise<Order[]> { /* ... */ }
  protected async loadTargetSystemData(): Promise<Order[]> { /* ... */ }
  protected async updateAction(oldItem: Order, newItem: Order): Promise<IntegrationMessageReturn> { /* ... */ }
  protected async deleteAction(item: Order): Promise<IntegrationMessageReturn> { /* ... */ }
  protected defineMetadata(): Array<Metadata> { /* ... */ }
}
```

### Example 2: Large Dataset Processing with Concurrency

```typescript
import {
  DeltaIntegrationFlow,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  promiseWithConcurrency
} from '@tunnelhub/sdk';

type Product = {
  productId: string;
  name: string;
  price: number;
};

class ProductSyncIntegration extends DeltaIntegrationFlow<Product> {
  protected async loadSourceSystemData(): Promise<Product[]> {
    // Simulate large dataset (50,000 items)
    const largeDataset = Array.from({ length: 50000 }, (_, i) => ({
      productId: `PROD-${i}`,
      name: `Product ${i}`,
      price: Math.random() * 100
    }));

    const allProducts = [];

    const stream = promiseWithConcurrency({
      array: largeDataset,
      mapFunction: async (item) => {
        // Simulate async processing (e.g., API call, transformation)
        await new Promise(resolve => setTimeout(resolve, 10));

        // Add additional data
        return {
          ...item,
          category: this.getCategory(item.productId),
          inStock: Math.random() > 0.1
        };
      },
      options: { concurrency: 50 } // Process 50 items at a time
    });

    for await (const product of stream) {
      allProducts.push(product);
    }

    console.log(`Loaded ${allProducts.length} products`);

    return allProducts;
  }

  private getCategory(productId: string): string {
    const hash = productId.split('-')[1];
    const categories = ['Electronics', 'Clothing', 'Books', 'Home', 'Toys'];
    return categories[hash % categories.length];
  }

  // ... other methods
}
```

### Example 3: Validating Metadata

```typescript
import {
  DeltaIntegrationFlow,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  validateMetadataArray
} from '@tunnelhub/sdk';

type Employee = {
  employeeId: string;
  name: string;
  email: string;
  department: string;
  salary: number;
};

class EmployeeSyncIntegration extends DeltaIntegrationFlow<Employee> {
  protected defineMetadata(): Array<Metadata> {
    const metadata = [
      { fieldName: 'employeeId', fieldLabel: 'Employee ID', fieldType: 'TEXT' },
      { fieldName: 'name', fieldLabel: 'Full Name', fieldType: 'TEXT' },
      { fieldName: 'email', fieldLabel: 'Email Address', fieldType: 'TEXT' },
      { fieldName: 'department', fieldLabel: 'Department', fieldType: 'TEXT' },
      { fieldName: 'salary', fieldLabel: 'Annual Salary', fieldType: 'NUMBER' }
    ];

    try {
      validateMetadataArray(metadata);
      console.log('Metadata validation passed');
      return metadata;
    } catch (error) {
      console.error('Invalid metadata:', error.message);

      // Remove problematic field and retry
      const fixedMetadata = metadata.filter(
        m => !['Action', 'Status', 'Message'].includes(m.fieldLabel)
      );

      validateMetadataArray(fixedMetadata);
      return fixedMetadata;
    }
  }

  // ... other methods
}
```

### Example 4: Conditional Verbose Logging

```typescript
import {
  DeltaIntegrationFlow,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  SDK,
  AutomationParameter
} from '@tunnelhub/sdk';

type Transaction = {
  transactionId: string;
  amount: number;
  type: string;
};

class TransactionSyncIntegration extends DeltaIntegrationFlow<Transaction> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['transactionId'], ['amount', 'type'], context);

    // Enable verbose logging based on parameter
    SDK.verbose = AutomationParameter.getBooleanParameter(
      this.parameters,
      'verbose_logging',
      false
    );
  }

  protected async insertAction(item: Transaction): Promise<IntegrationMessageReturn> {
    SDK.log(`Inserting transaction: ${item.transactionId}`);
    SDK.log(`Amount: ${item.amount}, Type: ${item.type}`);

    try {
      const response = await fetch('https://bank.example.com/api/transactions', {
        method: 'POST',
        body: JSON.stringify(item)
      });

      if (response.ok) {
        SDK.log(`Transaction ${item.transactionId} inserted successfully`);
        return { message: 'Transaction created' };
      } else {
        SDK.log(`Failed to insert transaction ${item.transactionId}`);
        return { message: `HTTP ${response.status}` };
      }
    } catch (error) {
      SDK.log(`Error inserting transaction ${item.transactionId}: ${error.message}`);
      return { message: error.message };
    }
  }

  // ... other methods
}
```

## Best Practices

### 1. Use promiseAllSettled for Batch Operations

```typescript
// Good: Continue processing even if some fail
const results = await promiseAllSettled(
  items.map(item => this.processItem(item))
);

// Analyze results and handle errors
const successful = results.filter(r => r.status === 'fulfilled');
const failed = results.filter(r => r.status === 'rejected');
```

### 2. Use promiseWithConcurrency for Large Datasets

```typescript
// Good: Process with controlled concurrency
const stream = promiseWithConcurrency({
  array: largeDataset,
  mapFunction: async (item) => await processItem(item),
  options: { concurrency: 20 }
});

for await (const result of stream) {
  // Process result
}
```

### 3. Always Validate Metadata

```typescript
// Good: Validate before using
validateMetadataArray(metadata);
return metadata;

// Also good: Handle validation errors
try {
  validateMetadataArray(metadata);
  return metadata;
} catch (error) {
  console.error('Invalid metadata:', error.message);
  return getFallbackMetadata();
}
```

### 4. Use SDK.log for Debugging

```typescript
// Good: Conditional logging
SDK.log(`Processing item ${item.id}`);
SDK.log(`Result: ${JSON.stringify(result)}`);

// Enable verbose mode when needed
SDK.verbose = true;
```

### 5. Control Concurrency Based on API Limits

```typescript
// Good: Adjust concurrency based on API limits
const apiConcurrency = AutomationParameter.getParameter(
  this.parameters,
  'api_concurrency'
);

const concurrency = apiConcurrency ? parseInt(apiConcurrency) : 10;

const stream = promiseWithConcurrency({
  array: items,
  mapFunction: async (item) => await apiCall(item),
  options: { concurrency }
});
```

## Common Patterns

### Pattern 1: Retry Failed Operations

```typescript
protected async processWithRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3
): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      SDK.log(`Attempt ${attempt} failed, retrying...`);

      // Exponential backoff
      await new Promise(resolve =>
        setTimeout(resolve, Math.pow(2, attempt) * 1000)
      );
    }
  }

  throw new Error('Max retries exceeded');
}

protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
  return await this.processWithRetry(async () => {
    const response = await fetch('https://api.example.com/items', {
      method: 'POST',
      body: JSON.stringify(item)
    });

    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    return { message: 'Item created' };
  });
}
```

### Pattern 2: Batch with Concurrency

```typescript
protected async processInBatches<T, R>(
  items: T[],
  batchSize: number,
  concurrency: number,
  processFn: (item: T) => Promise<R>
): Promise<R[]> {
  const allResults: R[] = [];

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);

    const stream = promiseWithConcurrency({
      array: batch,
      mapFunction: processFn,
      options: { concurrency }
    });

    for await (const result of stream) {
      allResults.push(result);
    }

    SDK.log(`Processed batch ${i / batchSize + 1}/${Math.ceil(items.length / batchSize)}`);
  }

  return allResults;
}
```

### Pattern 3: Rate Limiting

```typescript
protected async processWithRateLimit<T>(
  items: T[],
  processFn: (item: T) => Promise<void>,
  requestsPerSecond: number
): Promise<void> {
  const delay = 1000 / requestsPerSecond;
  let lastRequestTime = 0;

  for (const item of items) {
    const now = Date.now();
    const timeSinceLastRequest = now - lastRequestTime;

    if (timeSinceLastRequest < delay) {
      await new Promise(resolve =>
        setTimeout(resolve, delay - timeSinceLastRequest)
      );
    }

    await processFn(item);
    lastRequestTime = Date.now();
  }
}
```
