# System Configuration

This reference explains how to work with external system configurations in TunnelHub SDK.

## Overview

TunnelHub supports integration with various external systems (databases, APIs, file servers, etc.). Each system type has
specific configuration parameters and authentication methods.

## Accessing System Configuration

Systems are already available in the `this.systems` array in all integration flow classes (DeltaIntegrationFlow,
BatchDeltaIntegrationFlow, NoDeltaIntegrationFlow, etc.).

### Constructor Validation Pattern

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private sourceSystem: TunnelHubSystem;
  private targetSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    this.sourceSystem = this.systems.find(s => s.internalName === 'SOURCE_API') as TunnelHubSystem;
    this.targetSystem = this.systems.find(s => s.internalName === 'TARGET_ERP') as TunnelHubSystem;

    if (!this.sourceSystem) {
      throw new Error('Required system SOURCE_API not found');
    }

    if (!this.targetSystem) {
      throw new Error('Required system TARGET_ERP not found');
    }

    if (this.sourceSystem.type !== 'HTTP') {
      throw new Error('SOURCE_API must be HTTP type');
    }

    if (this.targetSystem.type !== 'DATABASE') {
      throw new Error('TARGET_ERP must be DATABASE type');
    }
  }
}
```

### Using System in Methods

```typescript
class MyIntegration extends DeltaIntegrationFlow<MyType> {
  private apiSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const apiSystem = this.systems.find(s => s.internalName === 'MY_HTTP_API');

    if (!apiSystem) {
      throw new Error('System MY_HTTP_API not found');
    }

    this.apiSystem = apiSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const url = this.apiSystem.parameters.url;

    if (this.apiSystem.type === 'HTTP') {
      const headers = this.getHeaders(this.apiSystem);
      const response = await fetch(url, {headers});
      return await response.json();
    }

    throw new Error('Unsupported system type');
  }

  private getHeaders(system: TunnelHubSystem): Record<string, string> {
    const headers: Record<string, string> = {'Content-Type': 'application/json'};

    if (system.type === 'HTTP') {
      switch (system.parameters.authType) {
        case 'BASIC':
          const auth = btoa(`${system.parameters.user}:${system.parameters.password}`);
          headers['Authorization'] = `Basic ${auth}`;
          break;
        case 'HEADERS':
          headers['Authorization'] = system.parameters.authHeader;
          break;
      }
    }

    return headers;
  }
}
```

## System Types

### HTTP System

HTTP systems connect to RESTful APIs with various authentication methods.

**Type:** `HTTP`

**Parameters:**

```typescript
{
  url: string;
  authType: 'NONE' | 'BASIC' | 'QUERY_STRING' | 'HEADERS';
  user?: string;        // For BASIC auth
  password?: string;     // For BASIC auth
  queryString?: string;   // For QUERY_STRING auth
  authHeader?: string;   // For HEADERS auth
  custom?: GenericParameter[];
}
```

**Authentication Types:**

#### NONE (No Authentication)

```typescript
const httpSystem = this.systems.find(s => s.internalName === 'public_api');

if (httpSystem && httpSystem.type === 'HTTP') {
  const response = await fetch(httpSystem.parameters.url);
  return await response.json();
}
```

#### BASIC (HTTP Basic Authentication)

```typescript
const httpSystem = this.systems.find(s => s.internalName === 'basic_auth_api');

if (httpSystem && httpSystem.type === 'HTTP' && httpSystem.parameters.authType === 'BASIC') {
  const auth = btoa(`${httpSystem.parameters.user}:${httpSystem.parameters.password}`);

  const response = await fetch(httpSystem.parameters.url, {
    headers: {
      Authorization: `Basic ${auth}`,
    },
  });

  return await response.json();
}
```

#### QUERY_STRING (Authentication via Query String)

```typescript
const httpSystem = this.systems.find(s => s.internalName === 'query_auth_api');

if (httpSystem && httpSystem.type === 'HTTP' && httpSystem.parameters.authType === 'QUERY_STRING') {
  const url = `${httpSystem.parameters.url}?${httpSystem.parameters.queryString}`;

  const response = await fetch(url);
  return await response.json();
}
```

#### HEADERS (Authentication via Headers)

```typescript
const httpSystem = this.systems.find(s => s.internalName === 'header_auth_api');

if (httpSystem && httpSystem.type === 'HTTP' && httpSystem.parameters.authType === 'HEADERS') {
  const response = await fetch(httpSystem.parameters.url, {
    headers: {
      Authorization: httpSystem.parameters.authHeader,
    },
  });

  return await response.json();
}
```

### SOAP System

SOAP systems connect to SOAP web services.

**Type:** `SOAP`

**Parameters:**

```typescript
{
  wsdlUrl: string;
  authType: 'NONE' | 'BASIC' | 'HEADERS' | 'WS_SECURITY';
  user?: string;        // For BASIC or WS_SECURITY auth
  password?: string;     // For BASIC or WS_SECURITY auth
  authHeader?: string;   // For HEADERS auth
  timestamp?: boolean;   // For WS_SECURITY auth
}
```

**Example:**

```typescript
const soapSystem = this.systems.find(s => s.internalName === 'soap_api');

if (soapSystem && soapSystem.type === 'SOAP') {
  const {wsdlUrl, authType} = soapSystem.parameters;

  const soapClient = await this.createSoapClient(wsdlUrl);

  const authHeaders = this.getSoapAuthHeaders(soapSystem);

  const results = await soapClient.GetCustomers(authHeaders);

  const data = results.map(customer => ({
    customerId: customer.CustomerID,
    name: customer.CustomerName,
  }));

  return data;
}
```

### DATABASE System

Database systems connect to relational and NoSQL databases.

**Type:** `DATABASE`

**Parameters vary by database type:**

#### MySQL

```typescript
{
  databaseType: 'MYSQL';
  host?: string;
  port?: string;
  user?: string;
  password?: string;
  database?: string;
  charset?: string;
  collation?: string;
}
```

#### PostgreSQL / MSSQL / Redshift

```typescript
{
  databaseType: 'POSTGRESQL' | 'MSSQL' | 'REDSHIFT';
  host?: string;
  port?: string;
  user?: string;
  password?: string;
  database?: string;
  charset?: string;
}
```

#### Oracle

```typescript
{
  databaseType: 'ORACLE';
  host?: string;
  port?: string;
  user?: string;
  password?: string;
  database?: string;
  charset?: string;
  oracleConnectionType?: 'SID' | 'SERVICE';
  serviceName?: string;  // For SERVICE connection
  sid?: string;          // For SID connection
}
```

#### MongoDB / JDBC

```typescript
{
  databaseType: 'MONGODB' | 'JDBC';
  uri?: string;          // MongoDB connection string
  jdbcUrl?: string;      // JDBC connection string
}
```

**Example:**

```typescript
class DatabaseIntegration extends DeltaIntegrationFlow<MyType> {
  private dbSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const dbSystem = this.systems.find(s => s.internalName === 'DATABASE_SERVER');

    if (!dbSystem || dbSystem.type !== 'DATABASE') {
      throw new Error('Database system not found');
    }

    this.dbSystem = dbSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const params = this.dbSystem.parameters;

    switch (params.databaseType) {
      case 'MYSQL':
      case 'POSTGRESQL':
      case 'MSSQL':
      case 'REDSHIFT':
        return await this.queryRelationalDatabase(params);
      case 'MONGODB':
        return await this.queryMongoDB(params);
      case 'ORACLE':
        return await this.queryOracle(params);
      default:
        throw new Error(`Unsupported database type: ${params.databaseType}`);
    }
  }

  private async queryRelationalDatabase(params: any): Promise<MyType[]> {
    const {host, port, user, password, database, databaseType} = params;

    const connection = await this.createConnection({
      host,
      port,
      user,
      password,
      database,
      type: databaseType,
    });

    const query = 'SELECT * FROM items';
    const results = await connection.query(query);

    await connection.close();

    return results;
  }

  // ... other methods
}
```

### FTP System

FTP systems connect to FTP servers for file transfers.

**Type:** `FTP`

**Parameters:**

```typescript
{
  host: string;
  port: number;
  user: string;
  password: string;
  encryption: 'IMPLICIT_FTPS' | 'EXPLICIT_FTPS' | 'PLAIN_FTP';
  timeout: number;
  maxReconnectionAttempts: number;
  reconnectDelay: number;
  automaticallyDisconnect: boolean;
}
```

**Example:**

```typescript
class FtpIntegration extends DeltaIntegrationFlow<MyType> {
  private ftpSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const ftpSystem = this.systems.find(s => s.internalName === 'FTP_SERVER');

    if (!ftpSystem || ftpSystem.type !== 'FTP') {
      throw new Error('FTP system not found');
    }

    this.ftpSystem = ftpSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const {host, port, user, password, encryption} = this.ftpSystem.parameters;

    const client = await this.createFtpClient({
      host,
      port,
      user,
      password,
      encryption,
    });

    const files = await client.list('/data/');
    const data = files.map(file => this.parseFile(file));

    await client.close();

    return data;
  }

  // ... other methods
}
```

### SFTP System

SFTP systems connect to SFTP servers for secure file transfers.

**Type:** `SFTP`

**Parameters:**

```typescript
{
  host: string;
  port: number;
  user: string;
  timeout: number;
  maxReconnectionAttempts: number;
  reconnectDelay: number;
  automaticallyDisconnect: boolean;
  authType: 'PASSWORD' | 'SSH_KEY';
  password?: string;        // For PASSWORD auth
  sshPrivateKey?: string;    // For SSH_KEY auth
}
```

**Example:**

```typescript
class SftpIntegration extends DeltaIntegrationFlow<MyType> {
  private sftpSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const sftpSystem = this.systems.find(s => s.internalName === 'SFTP_SERVER');

    if (!sftpSystem || sftpSystem.type !== 'SFTP') {
      throw new Error('SFTP system not found');
    }

    this.sftpSystem = sftpSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const {host, port, user, authType} = this.sftpSystem.parameters;

    let client;

    if (authType === 'PASSWORD') {
      client = await this.createSftpClient({
        host,
        port,
        user,
        password: this.sftpSystem.parameters.password,
      });
    } else {
      client = await this.createSftpClient({
        host,
        port,
        user,
        privateKey: this.sftpSystem.parameters.sshPrivateKey,
      });
    }

    const files = await client.list('/data/');
    const data = files.map(file => this.parseFile(file));

    await client.close();

    return data;
  }

  // ... other methods
}
```

### LDAP System

LDAP systems connect to LDAP directory services.

**Type:** `LDAP`

**Parameters:**

```typescript
{
  host: string;
  user: string;
  password: string;
  baseDn: string;
}
```

**Example:**

```typescript
class LdapIntegration extends DeltaIntegrationFlow<MyType> {
  private ldapSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const ldapSystem = this.systems.find(s => s.internalName === 'LDAP_SERVER');

    if (!ldapSystem || ldapSystem.type !== 'LDAP') {
      throw new Error('LDAP system not found');
    }

    this.ldapSystem = ldapSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const {host, user, password, baseDn} = this.ldapSystem.parameters;

    const client = await this.createLdapClient({
      url: `ldap://${host}`,
      bindDN: user,
      bindCredentials: password,
    });

    const results = await client.search(baseDn, {
      filter: '(objectClass=person)',
      attributes: ['cn', 'mail', 'telephoneNumber'],
    });

    const data = results.map(entry => ({
      userId: entry.object.dn,
      name: entry.object.cn,
      email: entry.object.mail,
    }));

    await client.unbind();

    return data;
  }

  // ... other methods
}
```

### MAIL System

Mail systems connect to SMTP/IMAP servers for email operations.

**Type:** `MAIL`

**Parameters:**

```typescript
{
  host: string;
  user: string;
  password: string;
  timeout: number;
}
```

**Example:**

```typescript
class MailIntegration extends DeltaIntegrationFlow<MyType> {
  private mailSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const mailSystem = this.systems.find(s => s.internalName === 'SMTP_SERVER');

    if (!mailSystem || mailSystem.type !== 'MAIL') {
      throw new Error('Mail system not found');
    }

    this.mailSystem = mailSystem;
  }

  protected async sendData(item: MyType): Promise<IntegrationMessageReturn> {
    const {host, user, password, timeout} = this.mailSystem.parameters;

    const transporter = this.createTransporter({
      host,
      port: 587,
      secure: false,
      auth: {
        user,
        pass: password,
      },
      timeout,
    });

    await transporter.sendMail({
      from: user,
      to: item.email,
      subject: 'Notification',
      text: item.message,
    });

    return {message: 'Email sent'};
  }

  // ... other methods
}
```

### SAPRFC System

SAPRFC systems connect to SAP systems via RFC (Remote Function Call).

**Type:** `SAPRFC`

**Parameters:**

```typescript
{
  host: string;
  user: string;
  password: string;
  systemNumber: string;
  mandt: string;
  trace: boolean;
}
```

**Example:**

```typescript
class SapIntegration extends DeltaIntegrationFlow<MyType> {
  private sapSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const sapSystem = this.systems.find(s => s.internalName === 'SAP_SYSTEM');

    if (!sapSystem || sapSystem.type !== 'SAPRFC') {
      throw new Error('SAP system not found');
    }

    this.sapSystem = sapSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const {host, user, password, systemNumber, mandt} = this.sapSystem.parameters;

    const client = await this.createSapClient({
      ashost: host,
      sysnr: systemNumber,
      client: mandt,
      user,
      passwd: password,
      trace: this.sapSystem.parameters.trace,
    });

    const results = await client.call('Z_GET_CUSTOMERS');

    const data = results.map(customer => ({
      customerId: customer.KUNNR,
      name: customer.NAME1,
    }));

    await client.close();

    return data;
  }

  // ... other methods
}
```

### SMB System

SMB systems connect to Windows/Samba file shares.

**Type:** `SMB`

**Parameters:**

```typescript
{
  host: string;
  user: string;
  password: string;
  homeDirectory: string;
}
```

**Example:**

```typescript
class SmbIntegration extends DeltaIntegrationFlow<MyType> {
  private smbSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['id'], ['name'], context);

    const smbSystem = this.systems.find(s => s.internalName === 'FILE_SHARE');

    if (!smbSystem || smbSystem.type !== 'SMB') {
      throw new Error('SMB system not found');
    }

    this.smbSystem = smbSystem;
  }

  protected async loadSourceSystemData(): Promise<MyType[]> {
    const {host, user, password, homeDirectory} = this.smbSystem.parameters;

    const client = await this.createSmbClient({
      host,
      username: user,
      password,
      shareName: homeDirectory,
    });

    const files = await client.listFiles('/');
    const data = files.map(file => this.parseFile(file));

    await client.close();

    return data;
  }

  // ... other methods
}
```

## Custom Parameters

All system types support custom parameters:

```typescript
{
  custom?: GenericParameter[];  // Additional custom parameters
}
```

**Accessing Custom Parameters:**

```typescript
const system = this.systems.find(s => s.internalName === 'my_system');

if (system) {
  const customParam1 = system.parameters.custom?.find(p => p.name === 'param1');
  const customParam2 = system.parameters.custom?.find(p => p.name === 'param2');

  console.log('Custom param 1:', customParam1?.value);
  console.log('Custom param 2:', customParam2?.value);
}
```

## Complete Example: Multi-System Integration

```typescript
import {DeltaIntegrationFlow, ProcessorPayload, LambdaContext, Metadata, TunnelHubSystem} from '@tunnelhub/sdk';

type User = {
  userId: string;
  name: string;
  email: string;
};

class MultiSystemIntegration extends DeltaIntegrationFlow<User> {
  private sourceSystem: TunnelHubSystem;
  private targetSystem: TunnelHubSystem;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, ['userId'], ['name', 'email'], context);

    this.sourceSystem = this.systems.find(s => s.internalName === 'SOURCE_CRM') as TunnelHubSystem;
    this.targetSystem = this.systems.find(s => s.internalName === 'TARGET_ERP') as TunnelHubSystem;

    if (!this.sourceSystem) {
      throw new Error('Source CRM system not found');
    }

    if (!this.targetSystem) {
      throw new Error('Target ERP system not found');
    }

    if (this.sourceSystem.type !== 'HTTP') {
      throw new Error('Source system must be HTTP type');
    }

    if (this.targetSystem.type !== 'DATABASE') {
      throw new Error('Target system must be DATABASE type');
    }
  }

  protected async loadSourceSystemData(): Promise<User[]> {
    const headers = this.getHttpHeaders(this.sourceSystem);
    let url = this.sourceSystem.parameters.url;

    if (this.sourceSystem.parameters.authType === 'QUERY_STRING') {
      url += `?${this.sourceSystem.parameters.queryString}`;
    }

    const response = await fetch(url, {headers});

    if (!response.ok) {
      throw new Error(`Failed to load users: ${response.statusText}`);
    }

    return await response.json();
  }

  protected async loadTargetSystemData(): Promise<User[]> {
    const params = this.targetSystem.parameters;

    if (params.databaseType !== 'POSTGRESQL') {
      throw new Error('Target system must be PostgreSQL');
    }

    const client = await this.createPgClient({
      host: params.host,
      port: params.port,
      user: params.user,
      password: params.password,
      database: params.database,
    });

    const results = await client.query('SELECT * FROM users');

    await client.end();

    return results.rows;
  }

  protected async insertAction(item: User): Promise<IntegrationMessageReturn> {
    const client = await this.createPgClient({
      host: this.targetSystem.parameters.host,
      port: this.targetSystem.parameters.port,
      user: this.targetSystem.parameters.user,
      password: this.targetSystem.parameters.password,
      database: this.targetSystem.parameters.database,
    });

    await client.query('INSERT INTO users (external_id, name, email) VALUES ($1, $2, $3)', [
      item.userId,
      item.name,
      item.email,
    ]);

    await client.end();

    return {message: 'User inserted'};
  }

  private getHttpHeaders(system: TunnelHubSystem): Record<string, string> {
    const headers: Record<string, string> = {
      'Content-Type': 'application/json',
    };

    if (system.type === 'HTTP') {
      switch (system.parameters.authType) {
        case 'BASIC':
          const auth = btoa(`${system.parameters.user}:${system.parameters.password}`);
          headers['Authorization'] = `Basic ${auth}`;
          break;
        case 'HEADERS':
          headers['Authorization'] = system.parameters.authHeader;
          break;
      }
    }

    return headers;
  }

  protected async updateAction(oldItem: User, newItem: User): Promise<IntegrationMessageReturn> {
    /* ... */
  }
  protected async deleteAction(item: User): Promise<IntegrationMessageReturn> {
    /* ... */
  }
  protected defineMetadata(): Array<Metadata> {
    /* ... */
  }
}

export const handler = async (event: ProcessorPayload, context: LambdaContext) => {
  const integration = new MultiSystemIntegration(event, context);
  await integration.doIntegration();

  if (integration.hasAnyErrors()) {
    throw new Error('Integration completed with errors');
  }

  return {statusCode: 200, body: 'Integration completed successfully'};
};
```

## Best Practices

### 1. Validate System Type in Constructor

```typescript
constructor(event: ProcessorPayload, context?: LambdaContext) {
  super(event, ['id'], ['name'], context);

  const apiSystem = this.systems.find(s => s.internalName === 'MY_API');

  if (!apiSystem) {
    throw new Error('System MY_API not found');
  }

  if (apiSystem.type !== 'HTTP') {
    throw new Error(`Expected HTTP system, got ${apiSystem.type}`);
  }

  this.apiSystem = apiSystem;
}
```

### 2. Handle Missing Required Parameters

```typescript
const {host, port, user, password} = this.dbSystem.parameters;

if (!host || !port || !user || !password) {
  throw new Error('Missing required database parameters');
}
```

### 3. Validate Authentication Configuration

```typescript
if (this.httpSystem.type === 'HTTP') {
  switch (this.httpSystem.parameters.authType) {
    case 'BASIC':
      if (!this.httpSystem.parameters.user || !this.httpSystem.parameters.password) {
        throw new Error('user and password are required for BASIC auth');
      }
      break;
    case 'HEADERS':
      if (!this.httpSystem.parameters.authHeader) {
        throw new Error('authHeader is required for HEADERS auth');
      }
      break;
  }
}
```

### 4. Log System Details for Debugging

```typescript
const system = this.systems.find(s => s.internalName === 'my_system');

if (system) {
  SDK.log(`System type: ${system.type}`);
  SDK.log(`System name: ${system.name}`);
  SDK.log(`System internal name: ${system.internalName}`);
}
```
