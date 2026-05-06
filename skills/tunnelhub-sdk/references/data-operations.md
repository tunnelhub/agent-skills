# Data Operations

This reference explains how to use `DataStore`, `ConversionTable`, `Sequences`, and `System` classes for data
operations in TunnelHub SDK.

## Overview

TunnelHub SDK provides four main classes for data operations:

1. **DataStore** - Read and write conversion table items stored in DynamoDB
2. **ConversionTable** - Resolve mapped values between systems
3. **Sequences** - Generate sequential IDs atomically
4. **System** - Retrieve system configuration

## DataStore & ConversionTable

### Purpose

Conversion tables allow mapping values between source and target systems. Use them to translate IDs, codes, statuses,
or any other values that differ between systems.

### When to Use

- Mapping department codes between systems
- Translating product categories
- Converting status values
- Mapping user roles or permissions
- Translating currency codes
- Any value transformation that requires a lookup table

### Constructor

```typescript
import { DataStore, ConversionTable } from '@tunnelhub/sdk';

const dataStore = new DataStore(event: ProcessorPayload);
const conversionTable = new ConversionTable(dataStore);
```

**Example:**

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private conversionTable: ConversionTable;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name', 'department'], context);

    const dataStore = new DataStore(this.executionEvent);
    this.conversionTable = new ConversionTable(dataStore);
  }
}
```

### Reading All Items with DataStore

```typescript
public async getConversionTableItems(externalCode: string): Promise<DataStoreItemInternal[]>
```

Retrieve all items from a conversion table using its `externalCode`.

**Parameters:**

- `externalCode`: Conversion table identifier stored in TunnelHub

**Returns:**

- `Promise<DataStoreItemInternal[]>` - Array of items with `fromValue`, `toValue`, and metadata fields

**Behavior:**

- Return `[]` when the conversion table does not exist
- Resolve the table header internally by `externalCode`

**Example:**

```typescript
const dataStore = new DataStore(this.executionEvent);
const items = await dataStore.getConversionTableItems('departments');

const salesMapping = items.find(item => item.fromValue === 'SALES');
```

### getValuesFromConversionTable

```typescript
public async getValuesFromConversionTable(tableName: string): Promise<DataStoreItemInternal[]>
```

Retrieve and cache all items from a conversion table.

**Parameters:**

- `tableName`: Name of the conversion table (externalCode)

**Returns:**

- `Promise<DataStoreItemInternal[]>` - Array of conversion table items

**Example:**

```typescript
class DepartmentMappingIntegration extends DeltaIntegrationFlow<Employee> {
  private conversionTable: ConversionTable;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['employeeId'], ['name', 'department'], context);

    const dataStore = new DataStore(this.executionEvent);
    this.conversionTable = new ConversionTable(dataStore);
  }

  protected async insertAction(item: Employee): Promise<IntegrationMessageReturn> {
    // Load all department mappings
    const departments = await this.conversionTable.getValuesFromConversionTable('departments');

    // Find matching department
    const targetDept = departments.find(d => d.fromValue === item.department);

    if (!targetDept) {
      throw new Error(`Department not found: ${item.department}`);
    }

    // Create employee with mapped department ID
    const response = await fetch('https://erp.example.com/api/employees', {
      method: 'POST',
      body: JSON.stringify({
        name: item.name,
        departmentId: targetDept.toValue,
      }),
    });

    return {message: 'Employee created', data: {id: response.data.id}};
  }
}
```

### getValueFromConversionTable

```typescript
public async getValueFromConversionTable(
  tableName: string,
  fromValue?: string,
  toValue?: string,
  notRequired = false,
): Promise<string | undefined>
```

Retrieve a mapped value from a conversion table.

**Parameters:**

- `tableName`: Name of the conversion table
- `fromValue`: Source value to look up
- `toValue`: Target value to reverse look up
- `notRequired`: Return `undefined` instead of throwing when a value is not found or when parameters are intentionally omitted

**Returns:**

- `Promise<string | undefined>` - Mapped value or `undefined`

**Behavior:**

- Pass exactly one of `fromValue` or `toValue`
- Throw `database.invalidConversionTableParams` when both or neither are provided and `notRequired` is `false`
- Throw `database.valueNotFoundInTable` when the value does not exist and `notRequired` is `false`

**Example:**

```typescript
class ProductCategoryIntegration extends DeltaIntegrationFlow<Product> {
  private conversionTable: ConversionTable;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['productId'], ['name', 'category'], context);

    const dataStore = new DataStore(this.executionEvent);
    this.conversionTable = new ConversionTable(dataStore);
  }

  protected async insertAction(item: Product): Promise<IntegrationMessageReturn> {
    // Map category code to target category ID
    const categoryId = await this.conversionTable.getValueFromConversionTable('product_categories', item.category);

    // Create product with mapped category ID
    const response = await fetch('https://ecommerce.example.com/api/products', {
      method: 'POST',
      body: JSON.stringify({
        name: item.name,
        categoryId,
        price: item.price,
      }),
    });

    return {message: 'Product created', data: {id: response.data.id}};
  }
}
```

### Writing Conversion Table Items with DataStore

Use `DataStore` when integrations need to create, update, or remove conversion table items directly.

### createConversionTableItem

```typescript
public async createConversionTableItem(input: CreateConversionTableItemInput): Promise<DataStoreItemInternal>
```

Create a new item in a conversion table.

**Input:**

```typescript
type CreateConversionTableItemInput = {
  externalCode: string;
  fromValue: string;
  toValue: string;
};
```

**Returns:**

- `Promise<DataStoreItemInternal>` - Created item with `uuid`, `createdAt`, `createdBy`, `fromValue`, and `toValue`

**Behavior:**

- Resolve the conversion table internally by `externalCode`
- Generate a `uuid` automatically
- Set `createdBy` to `SDK`
- Throw `database.conversionTableNotFound` when the table does not exist

**Example:**

```typescript
const dataStore = new DataStore(this.executionEvent);

const createdItem = await dataStore.createConversionTableItem({
  externalCode: 'departments',
  fromValue: 'SALES',
  toValue: 'DEPT-001',
});
```

### updateConversionTableItem

```typescript
public async updateConversionTableItem(input: UpdateConversionTableItemInput): Promise<DataStoreItemInternal>
```

Update one conversion table item by `itemUuid`.

**Input:**

```typescript
type UpdateConversionTableItemInput = {
  externalCode: string;
  itemUuid: string;
  fromValue?: string;
  toValue?: string;
};
```

**Returns:**

- `Promise<DataStoreItemInternal>` - Updated item with refreshed `updatedAt` and `updatedBy`

**Behavior:**

- Require at least one of `fromValue` or `toValue`
- Set `updatedBy` to `SDK`
- Throw `database.invalidConversionTableItemUpdateParams` when no update fields are provided
- Throw `database.conversionTableNotFound` when the table does not exist
- Throw `database.conversionTableItemNotFound` when the item does not exist in the resolved table

**Example:**

```typescript
await dataStore.updateConversionTableItem({
  externalCode: 'departments',
  itemUuid: createdItem.uuid,
  toValue: 'DEPT-010',
});
```

### deleteConversionTableItem

```typescript
public async deleteConversionTableItem(input: DeleteConversionTableItemInput): Promise<boolean>
```

Delete one conversion table item by `itemUuid`.

**Input:**

```typescript
type DeleteConversionTableItemInput = {
  externalCode: string;
  itemUuid: string;
};
```

**Returns:**

- `Promise<boolean>` - `true` when the item is deleted

**Behavior:**

- Resolve the table internally by `externalCode`
- Throw `database.conversionTableNotFound` when the table does not exist
- Throw `database.conversionTableItemNotFound` when the item does not exist in the resolved table

**Example:**

```typescript
await dataStore.deleteConversionTableItem({
  externalCode: 'departments',
  itemUuid: createdItem.uuid,
});
```

### Complete Example: Multiple Conversion Tables

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  DataStore,
  ConversionTable,
} from '@tunnelhub/sdk';

type Order = {
  orderId: string;
  customerId: string;
  productId: string;
  paymentMethod: string;
  status: string;
};

class OrderSyncIntegration extends DeltaIntegrationFlow<Order> {
  private customerTable: ConversionTable;
  private productTable: ConversionTable;
  private paymentMethodTable: ConversionTable;
  private statusTable: ConversionTable;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['orderId'], ['customerId', 'productId', 'paymentMethod', 'status'], context);

    const dataStore = new DataStore(this.executionEvent);

    this.customerTable = new ConversionTable(dataStore);
    this.productTable = new ConversionTable(dataStore);
    this.paymentMethodTable = new ConversionTable(dataStore);
    this.statusTable = new ConversionTable(dataStore);
  }

  protected async insertAction(item: Order): Promise<IntegrationMessageReturn> {
    // Map all fields using conversion tables
    const customerId = await this.customerTable.getValueFromConversionTable('customers', item.customerId);

    const productId = await this.productTable.getValueFromConversionTable('products', item.productId);

    const paymentMethodId = await this.paymentMethodTable.getValueFromConversionTable(
      'payment_methods',
      item.paymentMethod,
    );

    const statusId = await this.statusTable.getValueFromConversionTable('order_statuses', item.status);

    // Create order with mapped IDs
    const response = await fetch('https://erp.example.com/api/orders', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({
        externalId: item.orderId,
        customerId,
        productId,
        paymentMethodId,
        statusId,
      }),
    });

    const result = await response.json();

    return {
      message: 'Order created',
      data: {externalId: result.id},
    };
  }

  protected async updateAction(oldItem: Order, newItem: Order): Promise<IntegrationMessageReturn> {
    // Similar mapping logic
    const statusId = await this.statusTable.getValueFromConversionTable('order_statuses', newItem.status);

    const response = await fetch(`https://erp.example.com/api/orders/${oldItem.externalId}`, {
      method: 'PATCH',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({
        statusId,
      }),
    });

    return {message: 'Order updated'};
  }

  protected async loadSourceSystemData(): Promise<Order[]> {
    /* ... */
  }
  protected async loadTargetSystemData(): Promise<Order[]> {
    /* ... */
  }
  protected async deleteAction(item: Order): Promise<IntegrationMessageReturn> {
    /* ... */
  }
  protected defineMetadata(): Array<Metadata> {
    /* ... */
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new OrderSyncIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Integration completed successfully'};
};
```

## Sequences

### Purpose

Generate sequential IDs atomically. This ensures unique, incrementing numbers across distributed executions.

### When to Use

- Generate customer IDs
- Create invoice numbers
- Generate order numbers
- Create sequential document IDs
- Any requirement for auto-incrementing IDs

### Constructor

```typescript
import { Sequences } from '@tunnelhub/sdk';

const sequences = new Sequences(event: ProcessorPayload);
```

**Example:**

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private sequences: Sequences;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.sequences = new Sequences(this.executionEvent);
  }
}
```

### getSequenceNextValue

```typescript
async getSequenceNextValue(sequenceName: string): Promise<number>
```

Get the next value from a sequence. This operation is atomic and thread-safe.

**Parameters:**

- `sequenceName`: Name of the sequence

**Returns:**

- `Promise<number>` - Next sequence value

**Throws:**

- `Error` if sequence is not found

**Example:**

```typescript
class CustomerSyncIntegration extends DeltaIntegrationFlow<Customer> {
  private sequences: Sequences;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['customerId'], ['name', 'email'], context);

    this.sequences = new Sequences(this.executionEvent);
  }

  protected async insertAction(item: Customer): Promise<IntegrationMessageReturn> {
    // Generate sequential customer ID
    const customerId = await this.sequences.getSequenceNextValue('customer_sequence');

    // Create customer with generated ID
    const response = await fetch('https://erp.example.com/api/customers', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({
        customerId: `CUST-${customerId}`,
        name: item.name,
        email: item.email,
      }),
    });

    const result = await response.json();

    return {
      message: 'Customer created',
      data: {customerId: result.id},
    };
  }

  protected async loadSourceSystemData(): Promise<Customer[]> {
    /* ... */
  }
  protected async loadTargetSystemData(): Promise<Customer[]> {
    /* ... */
  }
  protected async updateAction(oldItem: Customer, newItem: Customer): Promise<IntegrationMessageReturn> {
    /* ... */
  }
  protected async deleteAction(item: Customer): Promise<IntegrationMessageReturn> {
    /* ... */
  }
  protected defineMetadata(): Array<Metadata> {
    /* ... */
  }
}
```

### Batch Processing with Sequences

When generating IDs for multiple items in batch operations:

```typescript
class BatchOrderIntegration extends BatchDeltaIntegrationFlow<Order> {
  private sequences: Sequences;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['orderId'], ['status'], context);

    this.sequences = new Sequences(this.executionEvent);
  }

  protected async batchInsertAction(items: Order[]): Promise<IntegrationMessageReturnBatch[]> {
    // Generate IDs for all items in the batch
    const orderIds = await Promise.all(items.map(() => this.sequences.getSequenceNextValue('order_sequence')));

    // Create orders with generated IDs
    const response = await fetch('https://erp.example.com/api/orders/batch', {
      method: 'POST',
      headers: {'Content-Type': 'application/json'},
      body: JSON.stringify({
        orders: items.map((item, index) => ({
          orderId: `ORD-${orderIds[index]}`,
          customerId: item.customerId,
          status: item.status,
        })),
      }),
    });

    const result = await response.json();

    return result.success.map((success: boolean, index: number) => ({
      message: success ? 'Order created' : result.errors[index],
      data: {orderId: `ORD-${orderIds[index]}`},
      status: success ? 'SUCCESS' : 'FAIL',
    }));
  }

  protected async loadSourceSystemData(): Promise<Order[]> {
    /* ... */
  }
  protected async loadTargetSystemData(): Promise<Order[]> {
    /* ... */
  }
  protected async batchUpdateAction(oldItems: Order[], newItems: Order[]): Promise<IntegrationMessageReturnBatch[]> {
    /* ... */
  }
  protected async batchDeleteAction(items: Order[]): Promise<IntegrationMessageReturnBatch[]> {
    /* ... */
  }
  protected defineMetadata(): Array<Metadata> {
    /* ... */
  }
}
```

### Multiple Sequences

Use different sequences for different entity types:

```typescript
class MultiEntityIntegration extends DeltaIntegrationFlow<Order> {
  private customerSequence: string;
  private productSequence: string;
  private orderSequence: string;
  private invoiceSequence: string;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['orderId'], ['status'], context);

    const sequences = new Sequences(this.executionEvent);

    // Initialize multiple sequences
    this.customerSequence = 'customer_sequence';
    this.productSequence = 'product_sequence';
    this.orderSequence = 'order_sequence';
    this.invoiceSequence = 'invoice_sequence';
  }

  protected async insertAction(item: Order): Promise<IntegrationMessageReturn> {
    // Generate IDs for different entities
    const customerId = await this.sequences.getSequenceNextValue(this.customerSequence);
    const orderId = await this.sequences.getSequenceNextValue(this.orderSequence);
    const invoiceId = await this.sequences.getSequenceNextValue(this.invoiceSequence);

    // Create order with all generated IDs
    const response = await fetch('https://erp.example.com/api/orders', {
      method: 'POST',
      body: JSON.stringify({
        orderId: `ORD-${orderId}`,
        customerId: `CUST-${customerId}`,
        invoiceId: `INV-${invoiceId}`,
        status: item.status,
      }),
    });

    return {message: 'Order created'};
  }

  // ... other methods
}
```

## System

### Purpose

Retrieve system configuration details, including connection parameters and credentials for external systems.

### When to Use

- Get API endpoints for systems
- Retrieve authentication credentials
- Access system-type-specific parameters
- Get connection details for databases, FTP, etc.

### Accessing Systems in Integration Flows

Systems are already available in the `this.systems` array in all integration flow classes (DeltaIntegrationFlow,
BatchDeltaIntegrationFlow, etc.).

#### Constructor Validation Pattern

Validate required systems in the constructor:

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private defaultApiSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const defaultApiSystem = this.systems.find(value => value.internalName === 'MY_HTTP_API');

    if (!defaultApiSystem || defaultApiSystem.type !== 'HTTP' || defaultApiSystem.parameters.authType !== 'BASIC') {
      throw new Error(`O sistema MY_HTTP_API precisa ser do tipo HTTP com autorização Basic`);
    }

    this.defaultApiSystem = defaultApiSystem;
  }
}
```

#### Using System in Actions

Access system configuration in insert/update/delete actions:

```typescript
class OrderSyncIntegration extends DeltaIntegrationFlow<Order> {
  private sourceSystem: TunnelHubSystem;
  private targetSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['orderId'], ['status'], context);

    this.sourceSystem = this.systems.find(s => s.internalName === 'SOURCE_API') as TunnelHubSystem;
    this.targetSystem = this.systems.find(s => s.internalName === 'TARGET_ERP') as TunnelHubSystem;

    if (!this.sourceSystem) {
      throw new Error('Source system SOURCE_API not found');
    }

    if (!this.targetSystem) {
      throw new Error('Target system TARGET_ERP not found');
    }
  }

  protected async insertAction(item: Order): Promise<IntegrationMessageReturn> {
    const system = this.targetSystem;

    if (system.type === 'HTTP' && system.parameters.authType === 'BASIC') {
      const auth = btoa(`${system.parameters.user}:${system.parameters.password}`);
      const response = await fetch(`${system.parameters.url}/orders`, {
        method: 'POST',
        headers: {
          Authorization: `Basic ${auth}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          externalId: item.orderId,
          status: item.status,
        }),
      });

      return {message: 'Order created'};
    }

    throw new Error('Unsupported system type or auth type');
  }

  // ... other methods
}
```

#### Using System Type Guards

Use type guards to safely access system-specific parameters:

```typescript
class DatabaseIntegration extends DeltaIntegrationFlow<User> {
  private dbSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['userId'], ['name'], context);

    const dbSystem = this.systems.find(s => s.internalName === 'USER_DATABASE');

    if (!dbSystem || dbSystem.type !== 'DATABASE') {
      throw new Error('Database system not found or invalid type');
    }

    this.dbSystem = dbSystem;
  }

  protected async loadSourceSystemData(): Promise<User[]> {
    if (this.dbSystem.type === 'DATABASE') {
      const {host, port, user, password, database} = this.dbSystem.parameters;
      // Connect to database...
    }

    return [];
  }
}
```

### System Types and Parameters

Different system types have different parameter structures. Access systems using `this.systems.find()`:

#### HTTP System

```typescript
const httpSystem = this.systems.find(s => s.internalName === 'my_http_api');

if (httpSystem && httpSystem.type === 'HTTP') {
  const url = httpSystem.parameters.url;

  // Authentication
  switch (httpSystem.parameters.authType) {
    case 'NONE':
      // No authentication
      break;
    case 'BASIC':
      const username = httpSystem.parameters.user;
      const password = httpSystem.parameters.password;
      break;
    case 'QUERY_STRING':
      const queryString = httpSystem.parameters.queryString;
      break;
    case 'HEADERS':
      const authHeader = httpSystem.parameters.authHeader;
      break;
  }
}
```

#### SOAP System

```typescript
const soapSystem = this.systems.find(s => s.internalName === 'my_soap_api');

if (soapSystem && soapSystem.type === 'SOAP') {
  const wsdlUrl = soapSystem.parameters.wsdlUrl;
  const serviceNamespace = soapSystem.parameters.serviceNamespace;

  // Authentication
  switch (soapSystem.parameters.authType) {
    case 'BASIC':
      const username = soapSystem.parameters.user;
      const password = soapSystem.parameters.password;
      break;
    case 'WS_SECURITY':
      const username = soapSystem.parameters.user;
      const password = soapSystem.parameters.password;
      const timestamp = soapSystem.parameters.timestamp;
      break;
    case 'CUSTOM_HEADER':
      const authHeader = soapSystem.parameters.authHeader;
      break;
  }
}
```

#### DATABASE System

```typescript
const dbSystem = this.systems.find(s => s.internalName === 'my_database');

if (dbSystem && dbSystem.type === 'DATABASE') {
  const dbType = dbSystem.parameters.databaseType;

  switch (dbType) {
    case 'MYSQL':
    case 'POSTGRESQL':
    case 'MSSQL':
    case 'REDSHIFT':
      const host = dbSystem.parameters.host;
      const port = dbSystem.parameters.port;
      const user = dbSystem.parameters.user;
      const password = dbSystem.parameters.password;
      const database = dbSystem.parameters.database;
      break;
    case 'ORACLE':
      const host = dbSystem.parameters.host;
      const port = dbSystem.parameters.port;
      const user = dbSystem.parameters.user;
      const password = dbSystem.parameters.password;
      const serviceName = dbSystem.parameters.serviceName;
      const sid = dbSystem.parameters.sid;
      break;
    case 'MONGODB':
    case 'JDBC':
      const uri = dbSystem.parameters.uri;
      const jdbcUrl = dbSystem.parameters.jdbcUrl;
      break;
  }
}
```

#### FTP System

```typescript
const ftpSystem = this.systems.find(s => s.internalName === 'my_ftp');

if (ftpSystem && ftpSystem.type === 'FTP') {
  const host = ftpSystem.parameters.host;
  const port = ftpSystem.parameters.port;
  const user = ftpSystem.parameters.user;
  const password = ftpSystem.parameters.password;
  const encryption = ftpSystem.parameters.encryption;
  const timeout = ftpSystem.parameters.timeout;
  const maxReconnectionAttempts = ftpSystem.parameters.maxReconnectionAttempts;
  const reconnectDelay = ftpSystem.parameters.reconnectDelay;
  const automaticallyDisconnect = ftpSystem.parameters.automaticallyDisconnect;
}
```

#### SFTP System

```typescript
const sftpSystem = this.systems.find(s => s.internalName === 'my_sftp');

if (sftpSystem && sftpSystem.type === 'SFTP') {
  const host = sftpSystem.parameters.host;
  const port = sftpSystem.parameters.port;
  const user = sftpSystem.parameters.user;
  const timeout = sftpSystem.parameters.timeout;
  const maxReconnectionAttempts = sftpSystem.parameters.maxReconnectionAttempts;
  const reconnectDelay = sftpSystem.parameters.reconnectDelay;
  const automaticallyDisconnect = sftpSystem.parameters.automaticallyDisconnect;

  // Authentication
  switch (sftpSystem.parameters.authType) {
    case 'PASSWORD':
      const password = sftpSystem.parameters.password;
      break;
    case 'SSH_KEY':
      const sshPrivateKey = sftpSystem.parameters.sshPrivateKey;
      break;
  }
}
```

### Complete Example: Dynamic System Configuration

```typescript
import {
  DeltaIntegrationFlow,
  IntegrationMessageReturn,
  ProcessorPayload,
  LambdaContext,
  Metadata,
  TunnelHubSystem,
} from '@tunnelhub/sdk';

type User = {
  userId: string;
  name: string;
  email: string;
};

class DynamicSystemIntegration extends DeltaIntegrationFlow<User> {
  private sourceSystem: TunnelHubSystem;
  private targetSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['userId'], ['name', 'email'], context);

    this.sourceSystem = this.systems.find(s => s.internalName === 'SOURCE_API') as TunnelHubSystem;
    this.targetSystem = this.systems.find(s => s.internalName === 'TARGET_ERP') as TunnelHubSystem;

    if (!this.sourceSystem) {
      throw new Error('Source system SOURCE_API not found');
    }

    if (!this.targetSystem) {
      throw new Error('Target system TARGET_ERP not found');
    }
  }

  protected async loadSourceSystemData(): Promise<User[]> {
    const headers: Record<string, string> = {};

    if (this.sourceSystem.type === 'HTTP') {
      if (this.sourceSystem.parameters.authType === 'BASIC') {
        const auth = btoa(`${this.sourceSystem.parameters.user}:${this.sourceSystem.parameters.password}`);
        headers['Authorization'] = `Basic ${auth}`;
      } else if (this.sourceSystem.parameters.authType === 'HEADERS') {
        headers['Authorization'] = this.sourceSystem.parameters.authHeader;
      }
    }

    const response = await fetch(`${this.sourceSystem.parameters.url}/users`, {headers});

    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<User[]> {
    const headers: Record<string, string> = {};

    if (this.targetSystem.type === 'HTTP') {
      if (this.targetSystem.parameters.authType === 'BASIC') {
        const auth = btoa(`${this.targetSystem.parameters.user}:${this.targetSystem.parameters.password}`);
        headers['Authorization'] = `Basic ${auth}`;
      } else if (this.targetSystem.parameters.authType === 'HEADERS') {
        headers['Authorization'] = this.targetSystem.parameters.authHeader;
      }
    }

    const response = await fetch(`${this.targetSystem.parameters.url}/users`, {headers});

    return await response.json();
  }

  protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
    const headers: Record<string, string> = {'Content-Type': 'application/json'};

    if (this.targetSystem.type === 'HTTP' && this.targetSystem.parameters.authType === 'HEADERS') {
      headers['Authorization'] = this.targetSystem.parameters.authHeader;
    }

    const response = await fetch(`${this.targetSystem.parameters.url}/users`, {
      method: 'POST',
      headers,
      body: JSON.stringify({
        externalId: item.userId,
        name: item.name,
        email: item.email,
      }),
    });

    const result = await response.json();

    return {
      message: 'User created',
      data: {id: result.id},
    };
  }

  protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
    const headers: Record<string, string> = {'Content-Type': 'application/json'};

    if (this.targetSystem.type === 'HTTP' && this.targetSystem.parameters.authType === 'HEADERS') {
      headers['Authorization'] = this.targetSystem.parameters.authHeader;
    }

    await fetch(`${this.targetSystem.parameters.url}/users/${oldItem.externalId}`, {
      method: 'PATCH',
      headers,
      body: JSON.stringify({
        name: newItem.name,
        email: newItem.email,
      }),
    });

    return {message: 'User updated'};
  }

  protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
    const headers: Record<string, string> = {};

    if (this.targetSystem.type === 'HTTP' && this.targetSystem.parameters.authType === 'HEADERS') {
      headers['Authorization'] = this.targetSystem.parameters.authHeader;
    }

    await fetch(`${this.targetSystem.parameters.url}/users/${item.externalId}`, {
      method: 'DELETE',
      headers,
    });

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
  const integration = new DynamicSystemIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Integration completed successfully'};
};
```

## Best Practices

### 1. Initialize Systems in Constructor

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private conversionTable: ConversionTable;
  private sequences: Sequences;
  private sourceSystem: TunnelHubSystem;
  private targetSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const dataStore = new DataStore(this.executionEvent);
    this.conversionTable = new ConversionTable(dataStore);
    this.sequences = new Sequences(this.executionEvent);

    this.sourceSystem = this.systems.find(s => s.internalName === 'SOURCE_API') as TunnelHubSystem;
    this.targetSystem = this.systems.find(s => s.internalName === 'TARGET_ERP') as TunnelHubSystem;

    if (!this.sourceSystem || !this.targetSystem) {
      throw new Error('Required systems not found');
    }
  }
}
```

### 2. Handle Missing Data Gracefully

```typescript
protected async insertAction(item: MyType): Promise<IntegrationMessageReturn> {
  const mappedValue = await this.conversionTable.getValueFromConversionTable('table', item.code, undefined, true);

  if (!mappedValue) {
    SDK.log(`Mapping not found for code: ${item.code}`);
    return {
      message: 'Mapping not found',
      data: null
    };
  }

  // Proceed with mapping
  // ...
}
```

### 3. Use Sequence Prefixes for Readability

```typescript
const customerId = await sequences.getSequenceNextValue('customer_sequence');
const formattedId = `CUST-${String(customerId).padStart(6, '0')}`;
// Result: CUST-000001, CUST-000002, ...
```

### 4. Validate System Configuration Early

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const apiSystem = this.systems.find(s => s.internalName === 'MY_API');

    if (!apiSystem) {
      throw new Error('Required system MY_API not found');
    }

    if (apiSystem.type !== 'HTTP') {
      throw new Error('MY_API must be HTTP type');
    }

    if (apiSystem.parameters.authType === 'BASIC') {
      if (!apiSystem.parameters.user || !apiSystem.parameters.password) {
        throw new Error('MY_API: user and password are required for BASIC auth');
      }
    }
  }
}
```
