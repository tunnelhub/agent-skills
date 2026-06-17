# SQL Tables / Tabelas de Apoio

This reference explains how to use `SqlTables` from `@tunnelhub/sdk` to read and mutate structured relational support
data during managed TunnelHub automation executions.

## Overview

SQL Tables, also called Tabelas de Apoio, are small relational tables managed by TunnelHub and stored as SQLite files on
EFS. The platform owns table metadata, schema changes, CSV import/export, access control, and transport between
environments. Automations use the SDK helper only to operate rows at runtime.

Use SQL Tables when data has its own structure, typed columns, a primary key, filters, pagination, or must be visible and
maintainable in the product.

Keep using `DataStore` and `ConversionTable` when the requirement is a simple from/to value mapping.

## Runtime Requirements

SQL Tables only work in a managed runtime with SQLite-on-EFS support enabled.

Required `tunnelhub.yml` configuration:

```yaml
configuration:
  runtimeEngine: LAMBDA
  entrypoint: index.handler
  runtime: nodejs24.x
  memorySize: 1024
  timeout: 60
  runInVpc: true
  sqlTables:
    enabled: true
```

Requirements:

- Use `runtime: nodejs24.x` because the helper depends on `node:sqlite`
- Use `runInVpc: true`
- Use `runtimeEngine: LAMBDA` or `ECS_FARGATE`
- Enable `configuration.sqlTables.enabled: true`
- Execute inside TunnelHub managed runtime with `TH_SQL_TABLES_ENABLED=true`
- Ensure the SQL Tables EFS mount is provided by the platform
- Ensure the event includes `tenantId` and `environmentId`

Do not use `SqlTables` in local scripts, unmanaged runtimes, or older Node.js runtimes unless the test explicitly mocks
the helper.

## Data Model

Each SQL Table has platform metadata:

```typescript
type SqlTableMetadata = {
  tenantId: string;
  environmentId: string;
  packageId: string;
  externalCode: string;
  description: string;
  limitedUsers?: string[];
  schemaDefinition: SqlTableColumnDefinition[];
  primaryKey: string[];
  indexesDefinition: SqlTableIndexDefinition[];
  efsFilePath?: string;
  operationalStatus: 'PROVISIONING' | 'READY' | 'ERROR' | 'DELETING';
  operationalStatusReason?: string;
};
```

Supported column types:

- `TEXT`
- `INTEGER`
- `REAL`
- `NUMERIC`
- `BOOLEAN`
- `DATETIME`

Columns can be nullable, have defaults, be unique, or be auto-incremented depending on the platform schema. Besides the
business primary key, TunnelHub maintains a technical `rowId` used by `updateRow` and `deleteRow`.

## Constructor

```typescript
import {SqlTables} from '@tunnelhub/sdk';

const sqlTables = new SqlTables(event);
```

Inside an integration flow, prefer `this.executionEvent`:

```typescript
class CustomerCacheIntegration extends NoDeltaIntegrationFlow<Customer> {
  private readonly sqlTables: SqlTables;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
    this.sqlTables = new SqlTables(this.executionEvent);
  }
}
```

The constructor fails early when runtime support is not configured, including missing `tenantId`, missing
`environmentId`, `TH_SQL_TABLES_ENABLED !== 'true'`, missing mount path, or Node.js version lower than 24.

## insertRow

```typescript
public async insertRow<T extends Record<string, unknown> = Record<string, unknown>>(
  externalCode: string,
  values: T,
): Promise<{rowId: string; values: T}>
```

Insert a row and return the generated technical `rowId` plus normalized values read back from SQLite.

Example:

```typescript
const inserted = await sqlTables.insertRow('customer-cache', {
  document: '12345678900',
  name: 'Customer example',
  active: true,
  lastSeenAt: '2026-06-17T12:00:00Z',
});
```

Behavior:

- Resolve table metadata by `externalCode`
- Require table `operationalStatus` to be `READY`
- Reject unknown columns and `rowId` in the payload
- Require non-null primary key columns and non-nullable columns unless they have defaults or `autoIncrement`
- Apply default values when configured in the schema
- Normalize values by column type before insertion
- Map SQLite constraint failures to SDK SQL Table errors

## queryRows

```typescript
public async queryRows<T extends Record<string, unknown> = Record<string, unknown>>(
  externalCode: string,
  request?: {
    current?: number;
    pageSize?: number;
    filter?: Record<string, Array<string | number | boolean>>;
    sorter?: {field: string; order: 'ascend' | 'descend'};
  },
): Promise<{
  success: boolean;
  current: number;
  pageSize: number;
  total: number;
  data: Array<{rowId: string; values: T}>;
}>
```

List rows with pagination, optional filters, and optional ordering.

Example:

```typescript
const page = await sqlTables.queryRows<CustomerCacheRow>('customer-cache', {
  current: 1,
  pageSize: 20,
  filter: {
    document: ['12345678900'],
    active: [true],
  },
  sorter: {
    field: 'lastSeenAt',
    order: 'descend',
  },
});

for (const row of page.data) {
  SDK.log(`Customer ${row.values.document} has rowId ${row.rowId}`);
}
```

Behavior:

- Default `current` is `1`
- Default `pageSize` is `20`
- Reject non-integer, zero, or negative pagination values
- Build filters as `IN` conditions for each field
- Reject filters for fields not present in the schema
- Normalize filter values using the column type rules
- Use the explicit sorter when provided
- Reject sorters for fields not present in the schema or order values outside `ascend`/`descend`
- Default ordering is primary key columns plus `rowId ASC`

Always provide a reasonable `pageSize`; avoid assuming a SQL Table can be loaded entirely in one call.

## updateRow

```typescript
public async updateRow<T extends Record<string, unknown> = Record<string, unknown>>(
  externalCode: string,
  rowId: string,
  patch: Partial<T>,
): Promise<{rowId: string; values: T}>
```

Patch an existing row by technical `rowId`.

Example:

```typescript
const updated = await sqlTables.updateRow('customer-cache', inserted.rowId, {
  name: 'Updated customer',
  active: false,
});
```

Behavior:

- Require table `operationalStatus` to be `READY`
- Require the `rowId` to exist
- Reject unknown columns and `rowId` in the patch
- Reject updates to columns that are part of the SQL Table primary key
- Normalize patch values by column type
- Return the full row values after update

Use `rowId` from `insertRow` or `queryRows`. Do not try to update by business primary key directly with this helper.

## deleteRow

```typescript
public async deleteRow(externalCode: string, rowId: string): Promise<boolean>
```

Delete an existing row by technical `rowId`.

Example:

```typescript
await sqlTables.deleteRow('customer-cache', inserted.rowId);
```

Behavior:

- Require table `operationalStatus` to be `READY`
- Require the `rowId` to exist
- Return `true` after a successful delete

## Type Normalization Rules

Use values that match the SQL Table schema. The SDK normalizes acceptable inputs and rejects invalid payloads.

Rules:

- `TEXT`: Convert to string
- `BOOLEAN`: Accept `true`, `false`, `'true'`, `'false'`, `1`, `0`, `'1'`, `'0'`
- `INTEGER`: Accept finite safe integers or numeric strings representing safe integers
- `REAL` and `NUMERIC`: Accept finite numbers or numeric strings
- `DATETIME`: Accept values parseable by `Date` and containing timezone information, then store as ISO string
- `null`, `undefined`, and empty non-text values are accepted only for nullable non-primary-key columns

Important `DATETIME` examples:

```typescript
// Valid: includes timezone
await sqlTables.insertRow('events', {eventId: '1', occurredAt: '2026-06-17T12:00:00Z'});

// Invalid: no timezone suffix
await sqlTables.insertRow('events', {eventId: '2', occurredAt: '2026-06-17T12:00:00'});
```

## External Code Rules

The helper normalizes `externalCode` by trimming and lowercasing it. Valid values match:

```text
^[a-z][a-z0-9-]{0,63}$
```

Use stable, domain-oriented external codes such as `customer-cache`, `processed-orders`, or `employee-state`.

## Integration Pattern

Example using SQL Tables as an operational cache:

```typescript
type Customer = {
  document: string;
  name: string;
  active: boolean;
};

class ExportCustomers extends NoDeltaIntegrationFlow<Customer> {
  private readonly sqlTables: SqlTables;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
    this.sqlTables = new SqlTables(this.executionEvent);
  }

  protected async loadSourceSystemData(): Promise<Customer[]> {
    return fetchCustomersFromSource();
  }

  protected async sendData(customer: Customer): Promise<IntegrationMessageReturn> {
    const existing = await this.sqlTables.queryRows('customer-cache', {
      current: 1,
      pageSize: 1,
      filter: {document: [customer.document]},
    });

    if (existing.data.length > 0) {
      await this.sqlTables.updateRow('customer-cache', existing.data[0].rowId, {
        name: customer.name,
        active: customer.active,
      });
      return {message: 'Customer cache updated', data: customer};
    }

    await this.sqlTables.insertRow('customer-cache', customer);
    return {message: 'Customer cache inserted', data: customer};
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'document', fieldLabel: 'Document', fieldType: 'TEXT'},
      {fieldName: 'name', fieldLabel: 'Name', fieldType: 'TEXT'},
      {fieldName: 'active', fieldLabel: 'Active', fieldType: 'BOOLEAN'},
    ];
  }
}
```

## Troubleshooting

### Runtime not configured

Cause: Missing Node.js 24, `TH_SQL_TABLES_ENABLED`, `tenantId`, `environmentId`, or SQL Tables EFS mount.

Fix: Check `tunnelhub.yml`, deploy validation, automation runtime engine, VPC settings, and whether SQL Tables are enabled
for the automation.

### Table not found

Cause: Wrong `externalCode`, table not associated with the automation/package, or metadata not transported to the target
environment.

Fix: Confirm the table exists in the same tenant/environment and use the platform `externalCode`, not the display name.

### Table not ready

Cause: Table is still provisioning, deleting, or in error state.

Fix: Check the SQL Table operational status in the platform before running row mutations.

### Invalid row payload

Cause: Unknown column, missing required column, invalid type, invalid `DATETIME`, or attempt to write `rowId`.

Fix: Compare the payload with `schemaDefinition`; include timezone in datetime values.

### Primary key update not allowed

Cause: `updateRow` patch includes a primary key column.

Fix: Delete and recreate the row if the business key must change, or model mutable fields outside the primary key.

### Constraint violation

Cause: Duplicate primary key or unique index, not-null violation, or SQLite constraint error.

Fix: Query first by business key when implementing upsert-like behavior, and handle duplicates explicitly.

### Database locked or busy

Cause: Concurrent writes contending on the same SQLite file.

Fix: Reduce write concurrency, batch operations at the integration level, or retry the business operation when appropriate.

## Testing Notes

Tests that exercise the real `SqlTables` helper require Node.js 24 or newer because `node:sqlite` is unavailable in older
runtimes. For unit tests that do not need SQLite behavior, wrap `SqlTables` usage behind a small method or class property
and mock that boundary.

When testing real SQL Tables behavior, provide runtime-like environment variables and test metadata/SQLite files. Do not
scan production DynamoDB tables to discover metadata.
