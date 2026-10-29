# De/para, tabelas de apoio e sequências

Use os recursos da plataforma somente quando o fluxo precisar deles. Um de/para mantido pela operação pertence a
`ConversionTable`; conversões estáveis de formato ou enumeração pertencem ao código de conversão da automação.
Não substitua tabelas de conversão por um parâmetro JSON abrangente nem acrescente caches, checkpoints ou tabelas
próprias sem uma necessidade do fluxo.

Os contratos abaixo foram conferidos no SDK 4.1.5. Os exemplos de uso são trechos para os métodos da integração,
com as propriedades indicadas já inicializadas no construtor.

## Inicialização

Uma instância de `ConversionTable` atende várias tabelas, identificadas pelo `externalCode`. Não crie uma propriedade
ou classe para cada tabela quando o mesmo helper atende às consultas.

Trecho da classe e do construtor, após `super(...)`:

```typescript
import {ConversionTable, DataStore} from '@tunnelhub/sdk';

// Propriedades da classe
private readonly dataStore: DataStore;
private readonly conversionTables: ConversionTable;

// Inicialização no construtor
this.dataStore = new DataStore(this.executionEvent);
this.conversionTables = new ConversionTable(this.dataStore);
```

O evento determina o tenant e o ambiente. Esses helpers não criam a definição da tabela de conversão; ela precisa
estar cadastrada com o `externalCode` esperado.

## Consultar de/para

| Método | Retorno e comportamento |
| --- | --- |
| `dataStore.getConversionTableItems(externalCode)` | `Promise<DataStoreItemInternal[]>`; consulta os itens e retorna `[]` quando a tabela não existe. |
| `conversionTables.getValuesFromConversionTable(tableName)` | Retorna os mesmos itens e mantém o resultado em cache na instância. `tableName` é o `externalCode`. |
| `conversionTables.getValueFromConversionTable(tableName, fromValue?, toValue?, notRequired?)` | Resolve uma direção do de/para; retorna o valor correspondente ou lança erro conforme as regras abaixo. |

```typescript
const departments = await this.conversionTables.getValuesFromConversionTable('DEPARTAMENTOS');
const targetCode = await this.conversionTables.getValueFromConversionTable('DEPARTAMENTOS', 'RH-01');
const sourceCode = await this.conversionTables.getValueFromConversionTable('DEPARTAMENTOS', undefined, 'ERP-01');
const optionalCode = await this.conversionTables.getValueFromConversionTable(
  'DEPARTAMENTOS',
  'RH-02',
  undefined,
  true,
);
```

Regras de `getValueFromConversionTable`:

- Informe exatamente uma direção: `fromValue` para obter `toValue`, ou `toValue` para obter `fromValue`.
- A comparação é exata; não há normalização automática de espaços, caixa ou zeros à esquerda.
- Valores vazios são tratados como ausentes na validação dos argumentos.
- Com `notRequired = false` (padrão), argumentos incompatíveis geram `database.invalidConversionTableParams`;
  correspondência ausente gera `database.valueNotFoundInTable`.
- Com `notRequired = true`, esses dois casos retornam `undefined`. Falhas de acesso ao banco continuam sendo erros.
- A busca retorna a primeira correspondência. O helper não garante unicidade de `fromValue` ou `toValue`, nem
  detecta ambiguidade na consulta reversa.

Use o modo obrigatório quando a ausência impedir o envio correto. Se a regra permitir ausência, trate `undefined`
explicitamente conforme essa regra. Não transforme de/para obrigatório ausente em sucesso com
`{message: 'Não encontrado', data: null}`: uma ação individual resolvida continua sendo sucesso para o SDK.

### Cache e leitura após escrita

O cache de `ConversionTable` é por instância e por tabela, sem expiração ou método público de invalidação. Uma lista
vazia também fica em cache. `createConversionTableItem`, `updateConversionTableItem` e `deleteConversionTableItem`
não atualizam uma instância de `ConversionTable` já carregada.

Após uma alteração, se o fluxo precisar reler, use `DataStore.getConversionTableItems()` ou uma nova instância de
`ConversionTable`. Isso evita reutilizar a cópia em memória; não garante leitura fortemente consistente, pois as
consultas atuais do `DataStore` não configuram `ConsistentRead`.

Não altere a lista retornada por `getValuesFromConversionTable` supondo que ela é uma cópia: a instância reutiliza
essa lista em suas próximas buscas. Não adicione outro cache sobre esse comportamento sem necessidade concreta.

## Criar, alterar e excluir itens

As operações abaixo alteram itens de uma tabela existente. `externalCode` identifica a tabela; `itemUuid` identifica
o registro, não o valor de origem. Use o UUID obtido na leitura ou na criação.

Contratos de entrada exportados pelo SDK:

```typescript
type CreateConversionTableItemInput = {
  externalCode: string;
  fromValue: string;
  toValue: string;
};

type UpdateConversionTableItemInput = {
  externalCode: string;
  itemUuid: string;
  fromValue?: string;
  toValue?: string;
};

type DeleteConversionTableItemInput = {
  externalCode: string;
  itemUuid: string;
};
```

Use os tipos exportados de `@tunnelhub/sdk`; as definições acima documentam o contrato, não são tipos a duplicar na
integração.

| Método | Retorno e condições |
| --- | --- |
| `createConversionTableItem(input)` | `Promise<DataStoreItemInternal>`; gera UUID e preenche `createdAt`/`createdBy`, sendo o autor `SDK`. |
| `updateConversionTableItem(input)` | `Promise<DataStoreItemInternal>`; exige `fromValue` ou `toValue` diferente de `undefined` e preenche `updatedAt`/`updatedBy`. |
| `deleteConversionTableItem(input)` | `Promise<boolean>`; retorna `true` após excluir. Item ausente lança erro. |

Trechos independentes; execute somente a operação que faz parte do fluxo:

```typescript
const created = await this.dataStore.createConversionTableItem({
  externalCode: 'DEPARTAMENTOS',
  fromValue: 'RH-01',
  toValue: 'ERP-01',
});
```

```typescript
const updated = await this.dataStore.updateConversionTableItem({
  externalCode: 'DEPARTAMENTOS',
  itemUuid: created.uuid,
  toValue: 'ERP-02',
});
```

```typescript
await this.dataStore.deleteConversionTableItem({
  externalCode: 'DEPARTAMENTOS',
  itemUuid: created.uuid,
});
```

As três operações lançam `database.conversionTableNotFound` se a tabela não existir. Atualização sem campos lança
`database.invalidConversionTableItemUpdateParams`. Alteração e exclusão verificam se o item pertence à tabela
resolvida e lançam `database.conversionTableItemNotFound` se a condição falhar.

A criação gera um novo UUID a cada chamada; ela não é um upsert por `fromValue` e não impede valores de origem
repetidos. Não repita uma criação como se fosse idempotente. O SDK não oferece transação conjunta entre essas
operações e o envio ao sistema de destino: trate falhas parciais conforme a regra de negócio, sem simular sucesso.

## Sequências

Use `Sequences` somente quando houver necessidade de um contador persistente da plataforma. A sequência precisa
existir no tenant e no ambiente do evento; o helper não a cria automaticamente.

```typescript
import {Sequences} from '@tunnelhub/sdk';

// Dentro de um método da integração
const sequences = new Sequences(this.executionEvent);
const nextValue = await sequences.getSequenceNextValue('PEDIDOS');
```

- `getSequenceDetails(sequenceName): Promise<SequenceInternal>` consulta a definição e o valor atual.
- `getSequenceNextValue(sequenceName): Promise<number>` verifica a sequência e incrementa `currentValue` em 1 por
  atualização atômica no DynamoDB; devolve o novo valor e registra `updatedAt`/`updatedBy`.
- Não calcule o próximo valor com `getSequenceDetails() + 1`: isso perde a atomicidade.
- A leitura atual de detalhes traduz também falhas de acesso em `database.sequenceNotFound`; essa mensagem não
  comprova, isoladamente, que a sequência esteja ausente.

O incremento não é uma transação com a chamada ao destino. Uma falha ou repetição pode consumir números, por isso
não prometa uma sequência sem lacunas nem gere outro número para o mesmo registro sem avaliar a regra de reenvio.
Não use um novo número como chave de delta a cada execução: a identidade do mesmo registro precisa permanecer estável.

## SQL Tables, sistemas e testes

Para dados estruturados de apoio com colunas tipadas, filtros e manutenção pela plataforma, use `SqlTables` quando
essa persistência fizer parte do fluxo. Requisitos de runtime, API e validação estão em [sql-tables.md](sql-tables.md).
Tabelas SQL não substituem o de/para simples atendido por `ConversionTable`.

As configurações dos sistemas atribuídos já estão em `this.systems`. Valide e use esse conteúdo conforme
[system-configuration.md](system-configuration.md), sem uma nova consulta por item.

`SDK.testMode` não impede o acesso AWS de `DataStore`, `ConversionTable` ou `Sequences`. Faça mock da fronteira de
persistência nos testes conforme [testing-patterns.md](testing-patterns.md). Nunca faça scan de `TunnelHub-prd`;
use os helpers e consultas direcionadas compatíveis com o modelo de dados quando a investigação exigir acesso.
