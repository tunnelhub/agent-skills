# Testing Patterns

This reference explains how to test TunnelHub SDK integrations using Jest and AWS SDK mocks.

## Overview

Testing integrations involves:
1. Setting up test mode
2. Mocking external systems and APIs
3. Generating test data
4. Asserting integration behavior

## Test Mode Setup

### Enable Test Mode

```typescript
import { SDK } from '@tunnelhub/sdk';

// Enable test mode at the top of test file
SDK.testMode = true;
```

Test mode disables actual DynamoDB and S3 operations, allowing tests to run without external dependencies.

## Basic Test Structure

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  SDK
} from '@tunnelhub/sdk';

describe('MyIntegration', () => {
  SDK.testMode = true;

  const testEvent: ProcessorPayload = {
    tenantId: 'test-tenant',
    environmentId: 'test-env',
    executionId: 'test-exec',
    automationId: 'test-auto',
    expirationPeriod: 30,
    systems: [],
    parameters: {}
  };

  const testContext: LambdaContext = {
    callbackWaitsForEmptyEventLoop: false,
    functionVersion: '1',
    functionName: 'test-function',
    memoryLimitInMB: '512',
    logGroupName: '/aws/lambda/test',
    logStreamName: 'test-stream',
    invokedFunctionArn: 'arn:aws:lambda:us-east-1:123456789012:function:test',
    awsRequestId: 'test-request-id'
  };

  test('should complete integration without errors', async () => {
    class TestableIntegration extends MyIntegration {
      protected async loadSourceSystemData(): Promise<MyType[]> {
        return [
          { id: '1', name: 'Item 1' },
          { id: '2', name: 'Item 2' }
        ];
      }

      protected async loadTargetSystemData(): Promise<MyType[]> {
        return [];
      }

      protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
        return { message: 'Item inserted', data: {} };
      }

      protected async updateAction(oldItem: MyType, newItem: MyType): Promise<IntegrationMessageReturn> {
        return { message: 'Item updated', data: {} };
      }

      protected async deleteAction(item: MyType): Promise<IntegrationMessageReturn> {
        return { message: 'Item deleted', data: {} };
      }

      protected defineMetadata(): Array<Metadata> {
        return [
          { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
          { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
        ];
      }
    }

    const integration = new TestableIntegration(testEvent, testContext);
    await integration.doIntegration();

    expect(integration.hasAnyErrors()).toBeFalsy();
  });
});
```

## Mocking External Systems

### Mocking HTTP Requests

```typescript
describe('MyIntegration with HTTP mocking', () => {
  SDK.testMode = true;

  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should fetch data from source system', async () => {
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve([
          { id: '1', name: 'Item 1' }
        ])
      } as Response)
    ) as jest.Mock;

    class TestableIntegration extends MyIntegration {
      protected async loadSourceSystemData(): Promise<MyType[]> {
        const system = this.systems.find(s => s.internalName === 'source');

        const response = await fetch(`${system.parameters.url}/data`);
        return await response.json();
      }

      // ... other methods
    }

    const testEvent = {
      // ... event configuration
      systems: [{
        type: 'HTTP',
        internalName: 'source',
        parameters: { url: 'https://api.example.com', authType: 'NONE' }
      }]
    } as ProcessorPayload;

    const integration = new TestableIntegration(testEvent);

    const data = await integration.loadSourceSystemData();

    expect(fetch).toHaveBeenCalledWith('https://api.example.com/data');
    expect(data).toEqual([{ id: '1', name: 'Item 1' }]);
  });

  test('should handle HTTP errors', async () => {
    global.fetch = jest.fn(() =>
      Promise.resolve({
        ok: false,
        status: 500,
        statusText: 'Internal Server Error'
      } as Response)
    ) as jest.Mock;

    class TestableIntegration extends MyIntegration {
      protected async loadSourceSystemData(): Promise<MyType[]> {
        const response = await fetch('https://api.example.com/data');

        if (!response.ok) {
          throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        return await response.json();
      }

      // ... other methods
    }

    const integration = new TestableIntegration(testEvent);

    await expect(integration.loadSourceSystemData()).rejects.toThrow('HTTP 500: Internal Server Error');
  });
});
```

### Mocking AWS SDK

```typescript
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { mockClient } from 'aws-sdk-client-mock';

describe('MyIntegration with AWS mocking', () => {
  SDK.testMode = true;

  const ddbMock = mockClient(DynamoDBClient);

  afterEach(() => {
    ddbMock.reset();
  });

  test('should read from DynamoDB', async () => {
    ddbMock.on(GetItemCommand).resolves({
      Item: {
        PK: { S: 'test-key' },
        Value: { S: 'test-value' }
      }
    });

    // ... test implementation
  });
});
```

## Generating Test Data

### Using Faker

```typescript
import { faker } from '@faker-js/faker';

function createRandomUser() {
  return {
    userId: faker.string.uuid(),
    username: faker.internet.userName(),
    email: faker.internet.email(),
    avatar: faker.image.avatar(),
    password: faker.internet.password(),
    birthdate: faker.date.birthdate(),
    registeredAt: faker.date.past()
  };
}

describe('MyIntegration with test data', () => {
  SDK.testMode = true;

  test('should process 100 users', async () => {
    class TestableIntegration extends MyIntegration {
      protected async loadSourceSystemData(): Promise<MyType[]> {
        return faker.helpers.multiple(createRandomUser, { count: 100 });
      }

      // ... other methods
    }

    const integration = new TestableIntegration(testEvent);
    await integration.doIntegration();

    expect(integration['savedLogs'].length).toBe(100);
  });
});
```

### Factory Functions

```typescript
class TestDataFactory {
  static createUser(overrides: Partial<User> = {}): User {
    return {
      userId: faker.string.uuid(),
      name: faker.person.fullName(),
      email: faker.internet.email(),
      ...overrides
    };
  }

  static createUsers(count: number, overrides: Partial<User> = {}): User[] {
    return Array.from({ length: count }, () => this.createUser(overrides));
  }
}

describe('MyIntegration with factory', () => {
  SDK.testMode = true;

  test('should process users with custom attributes', async () => {
    const users = TestDataFactory.createUsers(10, {
      status: 'ACTIVE'
    });

    class TestableIntegration extends MyIntegration {
      protected async loadSourceSystemData(): Promise<MyType[]> {
        return users;
      }

      // ... other methods
    }

    const integration = new TestableIntegration(testEvent);
    await integration.doIntegration();

    expect(integration['savedLogs'].length).toBe(10);
  });
});
```

## Testing Specific Scenarios

### Test Insert Only (No Updates or Deletes)

```typescript
test('should insert only when source has new items', async () => {
  const sourceData = [
    { id: '1', name: 'New Item 1' },
    { id: '2', name: 'New Item 2' }
  ];

  const targetData = [];

  class TestableIntegration extends DeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
      return { message: 'Item inserted', data: {} };
    }

    protected async updateAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected async deleteAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration['savedLogs'].length).toBe(2);
  expect(integration.hasAnyErrors()).toBeFalsy();
});
```

### Test Update Only

```typescript
test('should update only when source has changed items', async () => {
  const sourceData = [
    { id: '1', name: 'Updated Item 1' }
  ];

  const targetData = [
    { id: '1', name: 'Old Item 1' }
  ];

  class TestableIntegration extends DeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async insertAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected async updateAction(oldItem: MyType, newItem: MyType): Promise<IntegrationMessageReturn> {
      expect(oldItem.name).toBe('Old Item 1');
      expect(newItem.name).toBe('Updated Item 1');
      return { message: 'Item updated', data: {} };
    }

    protected async deleteAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration['savedLogs'].length).toBe(1);
});
```

### Test Delete Only

```typescript
test('should delete only when target has items not in source', async () => {
  const sourceData = [];

  const targetData = [
    { id: '1', name: 'Deleted Item' }
  ];

  class TestableIntegration extends DeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async insertAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected async updateAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected async deleteAction(item: MyType): Promise<IntegrationMessageReturn> {
      expect(item.name).toBe('Deleted Item');
      return { message: 'Item deleted', data: {} };
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration['savedLogs'].length).toBe(1);
});
```

### Test Mixed Operations

```typescript
test('should handle insert, update, and delete in one execution', async () => {
  const sourceData = [
    { id: '1', name: 'Item 1 (Updated)' },      // Update
    { id: '2', name: 'Item 2' },                  // Same (noDelta)
    { id: '3', name: 'New Item 3' }               // Insert
  ];

  const targetData = [
    { id: '1', name: 'Old Item 1' },
    { id: '2', name: 'Item 2' },
    { id: '4', name: 'Deleted Item' }               // Delete
  ];

  class TestableIntegration extends DeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
      expect(item.id).toBe('3');
      return { message: 'Item inserted', data: {} };
    }

    protected async updateAction(oldItem: MyType, newItem: MyType): Promise<IntegrationMessageReturn> {
      expect(oldItem.name).toBe('Old Item 1');
      expect(newItem.name).toBe('Item 1 (Updated)');
      return { message: 'Item updated', data: {} };
    }

    protected async deleteAction(item: MyType): Promise<IntegrationMessageReturn> {
      expect(item.id).toBe('4');
      return { message: 'Item deleted', data: {} };
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration['savedLogs'].length).toBe(4);
  expect(integration.hasAnyErrors()).toBeFalsy();
});
```

### Test Error Handling

```typescript
test('should handle errors and continue processing', async () => {
  const sourceData = [
    { id: '1', name: 'Item 1' },
    { id: '2', name: 'Error Item' },
    { id: '3', name: 'Item 3' }
  ];

  const targetData = [];

  class TestableIntegration extends DeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
      if (item.name === 'Error Item') {
        return {
          message: 'Simulated error',
          data: null
        };
      }

      return { message: 'Item inserted', data: {} };
    }

    protected async updateAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected async deleteAction(): Promise<IntegrationMessageReturn> {
      throw new Error('Should not be called');
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration.hasAnyErrors()).toBeTruthy();
  expect(integration['savedLogs'].length).toBe(3);
});
```

## Testing Batch Operations

### Test Batch Insert

```typescript
test('should process items in batches', async () => {
  const sourceData = Array.from({ length: 250 }, (_, i) => ({
    id: `item-${i}`,
    name: `Item ${i}`
  }));

  const targetData = [];

  class TestableIntegration extends BatchDeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
      this.packageSize = 100; // 3 batches: 100, 100, 50
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async batchInsertAction(items: MyType[]): Promise<IntegrationMessageReturnBatch[]> {
      expect(items.length).toBeLessThanOrEqual(100);
      return items.map(item => ({
        message: 'Item inserted',
        data: {},
        status: 'SUCCESS'
      }));
    }

    protected async batchUpdateAction(): Promise<IntegrationMessageReturnBatch[]> {
      throw new Error('Should not be called');
    }

    protected async batchDeleteAction(): Promise<IntegrationMessageReturnBatch[]> {
      throw new Error('Should not be called');
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration['savedLogs'].length).toBe(250);
});
```

### Test Batch with Errors

```typescript
test('should handle errors in batch operations', async () => {
  const sourceData = [
    { id: '1', name: 'Item 1' },
    { id: '2', name: 'Error Item' },
    { id: '3', name: 'Item 3' }
  ];

  const targetData = [];

  class TestableIntegration extends BatchDeltaIntegrationFlow<MyType> {
    constructor(event: ProcessorPayload, context?: LambdaContext) {
      super(event, ['id'], ['name'], context);
      this.packageSize = 3;
    }

    protected async loadSourceSystemData(): Promise<MyType[]> {
      return sourceData;
    }

    protected async loadTargetSystemData(): Promise<MyType[]> {
      return targetData;
    }

    protected async batchInsertAction(items: MyType[]): Promise<IntegrationMessageReturnBatch[]> {
      return items.map(item => ({
        message: item.name === 'Error Item' ? 'Simulated error' : 'Item inserted',
        data: {},
        status: item.name === 'Error Item' ? 'FAIL' : 'SUCCESS'
      }));
    }

    protected async batchUpdateAction(): Promise<IntegrationMessageReturnBatch[]> {
      throw new Error('Should not be called');
    }

    protected async batchDeleteAction(): Promise<IntegrationMessageReturnBatch[]> {
      throw new Error('Should not be called');
    }

    protected defineMetadata(): Array<Metadata> {
      return [
        { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
        { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
      ];
    }
  }

  const integration = new TestableIntegration(testEvent);
  await integration.doIntegration();

  expect(integration.hasAnyErrors()).toBeTruthy();
  expect(integration['savedLogs'].length).toBe(3);
});
```

## Testing Logging Strategy

```typescript
describe('logging strategy', () => {
  SDK.testMode = true;

  let consoleLogSpy: jest.SpyInstance;

  beforeEach(() => {
    consoleLogSpy = jest.spyOn(SDK, 'log').mockImplementation(() => {});
  });

  afterEach(() => {
    consoleLogSpy.mockRestore();
  });

  test('should use realtime for small datasets', async () => {
    class TestableIntegration extends DeltaIntegrationFlow<MyType> {
      protected realtimeLoggingThreshold: number = 100;
      protected maxRealtimeItems: number = 1000;

      constructor(event: ProcessorPayload, context?: LambdaContext) {
        super(event, ['id'], ['name'], context);
      }

      protected async loadSourceSystemData(): Promise<MyType[]> {
        return Array.from({ length: 50 }, (_, i) => ({ id: `item-${i}`, name: `Item ${i}` }));
      }

      protected async loadTargetSystemData(): Promise<MyType[]> {
        return [];
      }

      protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
        return { message: 'Item inserted', data: {} };
      }

      protected async updateAction(): Promise<IntegrationMessageReturn> { throw new Error('Should not be called'); }
      protected async deleteAction(): Promise<IntegrationMessageReturn> { throw new Error('Should not be called'); }
      protected defineMetadata(): Array<Metadata> {
        return [
          { fieldName: 'id', fieldLabel: 'ID', fieldType: 'TEXT' },
          { fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT' }
        ];
      }
    }

    const integration = new TestableIntegration(testEvent);
    await integration.doIntegration();

    // Should see realtime mode in logs
    expect(consoleLogSpy).toHaveBeenCalledWith(expect.stringContaining('Realtime'));
  });
});
```

## Testing Parameter Management

```typescript
describe('parameter management', () => {
  SDK.testMode = true;

  test('should get parameter value', () => {
    const testEvent = {
      // ... event configuration
      parameters: {
        custom: [
          { name: 'api_key', value: 'test-api-key' },
          { name: 'retry_enabled', value: 'true' }
        ]
      }
    } as ProcessorPayload;

    const apiKey = AutomationParameter.getParameter(testEvent.parameters, 'api_key');
    expect(apiKey).toBe('test-api-key');

    const retryEnabled = AutomationParameter.getBooleanParameter(testEvent.parameters, 'retry_enabled');
    expect(retryEnabled).toBe(true);
  });

  test('should throw on missing required parameter', () => {
    const testEvent = {
      parameters: {
        custom: []
      }
    } as ProcessorPayload;

    expect(() => {
      AutomationParameter.getRequiredParameter(testEvent.parameters, 'missing_param');
    }).toThrow();
  });
});
```

## Best Practices

### 1. Use AAA Pattern (Arrange, Act, Assert)

```typescript
test('should insert item', async () => {
  // Arrange
  const item = { id: '1', name: 'Test Item' };

  // Act
  const result = await integration.insertAction(item);

  // Assert
  expect(result.message).toBe('Item inserted');
});
```

### 2. Use Descriptive Test Names

```typescript
// Good: Descriptive
test('should insert item when it does not exist in target system', async () => { /* ... */ });
test('should update item when it exists but has changed fields', async () => { /* ... */ });
test('should delete item when it exists in target but not in source', async () => { /* ... */ });

// Bad: Vague
test('insert test', async () => { /* ... */ });
test('update test', async () => { /* ... */ });
test('delete test', async () => { /* ... */ });
```

### 3. Test Both Success and Failure Scenarios

```typescript
test('should return success when API call succeeds', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({ ok: true, json: () => Promise.resolve({ id: '123' }) } as Response)
  ) as jest.Mock;

  const result = await integration.insertAction({ id: '1', name: 'Test' });

  expect(result.message).toBe('Item inserted');
});

test('should return error when API call fails', async () => {
  global.fetch = jest.fn(() =>
    Promise.resolve({ ok: false, status: 500, statusText: 'Internal Server Error' } as Response)
  ) as jest.Mock;

  const result = await integration.insertAction({ id: '1', name: 'Test' });

  expect(result.message).toContain('500');
});
```

### 4. Use Faker for Realistic Test Data

```typescript
import { faker } from '@faker-js/faker';

test('should process realistic user data', async () => {
  const users = Array.from({ length: 100 }, () => ({
    userId: faker.string.uuid(),
    username: faker.internet.userName(),
    email: faker.internet.email(),
    registeredAt: faker.date.past()
  }));

  // Test implementation
});
```

### 5. Clean Up After Tests

```typescript
afterEach(() => {
  jest.clearAllMocks();
  jest.restoreAllMocks();
});
```

### 6. Use Timeouts for Long-Running Tests

```typescript
test('should process large dataset', async () => {
  const integration = new LargeDatasetIntegration(testEvent);

  await integration.doIntegration();

  expect(integration.hasAnyErrors()).toBeFalsy();
}, 60000); // 60 second timeout
```
