# SQL Tables / Tabelas de Apoio

Esta referência explica como usar `SqlTables` de `@tunnelhub/sdk` para ler e alterar dados relacionais estruturados de
apoio durante execuções de automações no ambiente gerenciado do TunnelHub.

## Visão geral

SQL Tables, também chamadas de Tabelas de Apoio, são pequenas tabelas relacionais gerenciadas pelo TunnelHub e
armazenadas como arquivos SQLite no EFS. A plataforma gerencia os metadados das tabelas, as alterações de esquema,
a importação e exportação de CSV, o controle de acesso e o transporte entre ambientes. As automações usam o helper
do SDK apenas para operar registros durante a execução.

Use Tabelas de Apoio quando os dados tiverem estrutura própria, colunas tipadas, chave primária, filtros, paginação ou
precisarem ficar visíveis e disponíveis para manutenção no produto.

Continue usando `DataStore` e `ConversionTable` quando o requisito for um de/para simples de valores.

## Requisitos do ambiente de execução

As Tabelas de Apoio funcionam apenas em um ambiente gerenciado com suporte a SQLite no EFS habilitado.

Configuração obrigatória em `tunnelhub.yml`:

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

Requisitos:

- Use `runtime: nodejs24.x`, pois o helper depende de `node:sqlite`
- Use `runInVpc: true`
- Use `runtimeEngine: LAMBDA` ou `ECS_FARGATE`
- Habilite `configuration.sqlTables.enabled: true`
- Execute no ambiente gerenciado do TunnelHub com `TH_SQL_TABLES_ENABLED=true`
- Garanta que a plataforma forneça a montagem EFS das Tabelas de Apoio
- Garanta que o evento inclua `tenantId` e `environmentId`

Não use `SqlTables` em scripts locais, ambientes não gerenciados ou versões anteriores do Node.js, a menos que o
teste substitua explicitamente o helper por um mock.

## Modelo de dados

Cada Tabela de Apoio tem metadados na plataforma:

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

Tipos de coluna aceitos:

- `TEXT`
- `INTEGER`
- `REAL`
- `NUMERIC`
- `BOOLEAN`
- `DATETIME`

As colunas podem aceitar valores nulos, ter valores padrão, ser únicas ou ter incremento automático, conforme o
esquema da plataforma. Além da chave primária de negócio, o TunnelHub mantém um `rowId` técnico usado por
`updateRow` e `deleteRow`.

## Construtor

```typescript
import {SqlTables} from '@tunnelhub/sdk';

const sqlTables = new SqlTables(event);
```

Dentro de um fluxo de integração, prefira `this.executionEvent`:

```typescript
class CustomerCacheIntegration extends NoDeltaIntegrationFlow<Customer> {
  private readonly sqlTables: SqlTables;

  constructor(event: ProcessorPayload, context?: LambdaContext) {
    super(event, context);
    this.sqlTables = new SqlTables(this.executionEvent);
  }
}
```

O construtor falha imediatamente quando o suporte do ambiente de execução não está configurado, incluindo ausência
de `tenantId`, `environmentId` ou caminho de montagem, `TH_SQL_TABLES_ENABLED !== 'true'` ou versão do Node.js inferior a 24.

## insertRow

```typescript
public async insertRow<T extends Record<string, unknown> = Record<string, unknown>>(
  externalCode: string,
  values: T,
): Promise<{rowId: string; values: T}>
```

Insere um registro e retorna o `rowId` técnico gerado e os valores normalizados lidos de volta do SQLite.

Exemplo:

```typescript
const inserted = await sqlTables.insertRow('customer-cache', {
  document: '12345678900',
  name: 'Cliente de exemplo',
  active: true,
  lastSeenAt: '2026-06-17T12:00:00Z',
});
```

Comportamento:

- Resolve os metadados da tabela por `externalCode`
- Exige que o `operationalStatus` da tabela seja `READY`
- Rejeita colunas desconhecidas e `rowId` no payload
- Exige valores não nulos nas colunas da chave primária e nas que não aceitam nulos, salvo quando têm valor padrão ou `autoIncrement`
- Aplica valores padrão quando configurados no esquema
- Normaliza os valores pelo tipo da coluna antes da inserção
- Converte falhas de restrições do SQLite em erros de Tabelas de Apoio do SDK

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

Lista registros com paginação, filtros opcionais e ordenação opcional.

Exemplo:

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
  SDK.log(`Cliente ${row.values.document} tem rowId ${row.rowId}`);
}
```

Comportamento:

- O valor padrão de `current` é `1`
- O valor padrão de `pageSize` é `20`
- Rejeita valores de paginação não inteiros, iguais a zero ou negativos
- Constrói os filtros como condições `IN` para cada campo
- Rejeita filtros para campos ausentes no esquema
- Normaliza os valores dos filtros conforme as regras do tipo da coluna
- Usa a ordenação explícita quando fornecida
- Rejeita ordenações por campos ausentes no esquema ou valores de ordem diferentes de `ascend`/`descend`
- A ordenação padrão usa as colunas da chave primária seguidas de `rowId ASC`

Sempre informe um `pageSize` adequado; evite presumir que uma Tabela de Apoio possa ser carregada inteira em uma chamada.

## updateRow

```typescript
public async updateRow<T extends Record<string, unknown> = Record<string, unknown>>(
  externalCode: string,
  rowId: string,
  patch: Partial<T>,
): Promise<{rowId: string; values: T}>
```

Altera parcialmente um registro existente pelo `rowId` técnico.

Exemplo:

```typescript
const updated = await sqlTables.updateRow('customer-cache', inserted.rowId, {
  name: 'Cliente atualizado',
  active: false,
});
```

Comportamento:

- Exige que o `operationalStatus` da tabela seja `READY`
- Exige que o `rowId` exista
- Rejeita colunas desconhecidas e `rowId` no patch
- Rejeita alterações em colunas que fazem parte da chave primária da Tabela de Apoio
- Normaliza os valores do patch pelo tipo da coluna
- Retorna todos os valores do registro após a alteração

Use o `rowId` obtido em `insertRow` ou `queryRows`. Não tente atualizar diretamente pela chave primária de negócio
com este helper.

## deleteRow

```typescript
public async deleteRow(externalCode: string, rowId: string): Promise<boolean>
```

Exclui um registro existente pelo `rowId` técnico.

Exemplo:

```typescript
await sqlTables.deleteRow('customer-cache', inserted.rowId);
```

Comportamento:

- Exige que o `operationalStatus` da tabela seja `READY`
- Exige que o `rowId` exista
- Retorna `true` após uma exclusão bem-sucedida

## Regras de normalização de tipos

Use valores compatíveis com o esquema da Tabela de Apoio. O SDK normaliza as entradas aceitas e rejeita payloads inválidos.

Regras:

- `TEXT`: converte para string
- `BOOLEAN`: aceita `true`, `false`, `'true'`, `'false'`, `1`, `0`, `'1'`, `'0'`
- `INTEGER`: aceita inteiros finitos seguros ou strings numéricas que representem inteiros seguros
- `REAL` e `NUMERIC`: aceitam números finitos ou strings numéricas
- `DATETIME`: aceita valores interpretáveis por `Date` que contenham informação de fuso horário e os armazena como string ISO
- `null`, `undefined` e valores vazios não textuais são aceitos apenas em colunas que aceitam nulos e não fazem parte da chave primária

Exemplos importantes de `DATETIME`:

```typescript
// Válido: inclui fuso horário
await sqlTables.insertRow('events', {eventId: '1', occurredAt: '2026-06-17T12:00:00Z'});

// Inválido: sem sufixo de fuso horário
await sqlTables.insertRow('events', {eventId: '2', occurredAt: '2026-06-17T12:00:00'});
```

## Regras de código externo

O helper normaliza `externalCode`, removendo espaços nas extremidades e convertendo para minúsculas. Os valores
válidos correspondem a:

```text
^[a-z][a-z0-9-]{0,63}$
```

Use códigos externos estáveis e relacionados ao domínio, como `customer-cache`, `processed-orders` ou `employee-state`.

## Padrão de integração

Exemplo de uso das Tabelas de Apoio como cache operacional:

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
      return {message: 'Cache do cliente atualizado', data: customer};
    }

    await this.sqlTables.insertRow('customer-cache', customer);
    return {message: 'Cliente inserido no cache', data: customer};
  }

  protected defineMetadata(): Array<Metadata> {
    return [
      {fieldName: 'document', fieldLabel: 'Documento', fieldType: 'TEXT'},
      {fieldName: 'name', fieldLabel: 'Nome', fieldType: 'TEXT'},
      {fieldName: 'active', fieldLabel: 'Ativo', fieldType: 'BOOLEAN'},
    ];
  }
}
```

## Solução de problemas

### Ambiente de execução não configurado

Causa: ausência de Node.js 24, `TH_SQL_TABLES_ENABLED`, `tenantId`, `environmentId` ou montagem EFS das Tabelas de Apoio.

Correção: confira `tunnelhub.yml`, a validação de implantação, o mecanismo de execução da automação, as configurações
de VPC e se as Tabelas de Apoio estão habilitadas para a automação.

### Tabela não encontrada

Causa: `externalCode` incorreto, tabela não associada à automação ou ao pacote, ou metadados não transportados para o
ambiente de destino.

Correção: confirme que a tabela existe no mesmo tenant e ambiente e use o `externalCode` da plataforma, não o nome de exibição.

### Tabela não está pronta

Causa: a tabela ainda está em provisionamento, em exclusão ou em estado de erro.

Correção: confira o status operacional da Tabela de Apoio na plataforma antes de alterar registros.

### Payload de registro inválido

Causa: coluna desconhecida, coluna obrigatória ausente, tipo inválido, `DATETIME` inválido ou tentativa de escrever `rowId`.

Correção: compare o payload com `schemaDefinition`; inclua o fuso horário nos valores de data e hora.

### Alteração de chave primária não permitida

Causa: o patch de `updateRow` inclui uma coluna da chave primária.

Correção: exclua e recrie o registro se a chave de negócio precisar mudar, ou modele os campos mutáveis fora da chave primária.

### Violação de restrição

Causa: chave primária ou índice único duplicado, violação de coluna que não aceita nulos ou erro de restrição do SQLite.

Correção: consulte primeiro pela chave de negócio ao implementar um comportamento semelhante a upsert e trate
as duplicidades explicitamente.

### Banco bloqueado ou ocupado

Causa: escritas concorrentes disputando acesso ao mesmo arquivo SQLite.

Correção: reduza a concorrência de escrita, agrupe as operações em lotes na integração ou repita a operação de negócio
quando apropriado.

## Observações sobre testes

Testes que exercitam o helper `SqlTables` real exigem Node.js 24 ou superior, pois `node:sqlite` não está disponível em
versões anteriores. Em testes unitários que não precisam validar o comportamento do SQLite, encapsule o uso de
`SqlTables` em um método pequeno ou uma propriedade de classe e faça mock dessa fronteira.

Ao testar o comportamento real das Tabelas de Apoio, forneça variáveis de ambiente equivalentes às de execução e
metadados e arquivos SQLite de teste. Não faça scan de tabelas DynamoDB de produção para descobrir metadados.
