# Fluxos de integração

Use esta referência ao escolher um fluxo ou implementar seus métodos. Os contratos abaixo foram conferidos no
SDK 4.1.5; confirme as declarações e a implementação da versão instalada antes de usar um recurso diferente.

## Escolha do fluxo

| Necessidade | Classe | Métodos de envio |
| --- | --- | --- |
| Comparar a origem com uma imagem anterior, aplicando inclusões, alterações e exclusões previstas no escopo | `DeltaIntegrationFlow<T>` | `insertAction`, `updateAction`, `deleteAction` |
| Mesmo comportamento delta, com operação em lote suportada pelo destino | `BatchDeltaIntegrationFlow<T>` | `batchInsertAction`, `batchUpdateAction`, `batchDeleteAction` |
| Processar cada item recebido, sem comparação automática entre execuções | `NoDeltaIntegrationFlow<T>` | `sendData` |
| Mesmo comportamento sem delta, com operação em lote suportada pelo destino | `NoDeltaBatchIntegrationFlow<T>` | `sendDataInBatch` |

Quantidade de registros, isoladamente, não determina a classe. Uma API que aceita somente um registro por chamada
continua compatível com o fluxo individual em volumes grandes. Use batch quando o contrato e os limites da API
justificarem agrupar operações; não transforme o envio em lote em um `Promise.all` sem limite.

Delta opera da origem para o destino. Não oferece sincronização bidirecional nem resolução automática de conflitos.
NoDelta não fornece deduplicação entre execuções: repetição, confirmação de consumo e idempotência dependem do
contrato da integração. Nenhum dos quatro fluxos fornece garantia de execução exatamente uma vez.

## Contratos de construção e métodos

As assinaturas a seguir descrevem os membros da classe; não são arquivos completos de automação.

| Família | Chamada de `super` no construtor da automação |
| --- | --- |
| Delta e BatchDelta | `super(event, keyFields, deltaFields, context)` |
| NoDelta e NoDeltaBatch | `super(event, context)` |

`event` é `ProcessorPayload`; `context` é `LambdaContext` opcional. `keyFields` e `deltaFields` são arrays de
`keyof T` obrigatórios para Delta. O último argumento opcional `debug?: boolean` existe nos construtores,
mas o logger desses fluxos consulta `executionEvent.debug`; não suponha que passar somente esse argumento o habilite.

Todos os fluxos exigem:

- `loadSourceSystemData(payload?): Promise<T[]>`: carregar e normalizar os registros da origem.
- `defineMetadata(): Metadata[]`: definir as colunas do acompanhamento da execução.

Delta e BatchDelta também exigem `loadTargetSystemData(): Promise<T[]>`. A política de quando esse método é
chamado está descrita na seção sobre imagem delta abaixo.

### Métodos individuais

| Classe | Assinatura |
| --- | --- |
| Delta | `insertAction(item: T): Promise<IntegrationMessageReturn>` |
| Delta | `updateAction(oldItem: T, newItem: T): Promise<IntegrationMessageReturn>` |
| Delta | `deleteAction(item: T): Promise<IntegrationMessageReturn>` |
| NoDelta | `sendData(item: T, uuid: string): Promise<IntegrationMessageReturn>` |

Esses métodos são `protected`. NoDelta fornece em `uuid` o identificador do log do item; o argumento pode ser
omitido na implementação quando não for usado. Ele é criado a cada processamento e não é uma chave estável de
idempotência entre execuções.

`IntegrationMessageReturn` exige `message` e `data`. Retorne `data: null` quando não houver dado de retorno.
O tipo não possui `success` ou `status`. Um retorno resolvido é interpretado como sucesso pelo fluxo individual.

Para sinalizar falha, lance uma exceção. Não capture uma falha para retornar `{message: 'Falhou', data: null}`
ou `{success: false, ...}`: isso produz log de sucesso e pode incorporar o registro à imagem delta mesmo sem
gravação no destino. Capture apenas quando houver recuperação prevista ou contexto útil a acrescentar, relançando
o erro se a operação não foi concluída.

Exemplo completo de função chamada por uma ação, com transporte injetado para permitir teste isolado:

```typescript
import type {IntegrationMessageReturn} from '@tunnelhub/sdk';

type Employee = {employeeId: string; name: string};
type CreateEmployee = (employee: Employee) => Promise<{id: string}>;

export async function createEmployee(
  employee: Employee,
  createInTarget: CreateEmployee,
): Promise<IntegrationMessageReturn> {
  const created = await createInTarget(employee);
  return {message: 'Funcionário criado', data: created};
}
```

O cliente `createInTarget` deve rejeitar erros HTTP e rejeições de negócio do destino. `fetch` não rejeita
automaticamente respostas HTTP 4xx/5xx: confira `response.ok` e valide o corpo conforme o contrato da API.
Uma resposta HTTP 200 também pode carregar uma rejeição de negócio.

### Métodos em lote

| Classe | Assinatura |
| --- | --- |
| BatchDelta | `batchInsertAction(items: T[]): Promise<IntegrationMessageReturnBatch[]>` |
| BatchDelta | `batchUpdateAction(oldItems: T[], newItems: T[]): Promise<IntegrationMessageReturnBatch[]>` |
| BatchDelta | `batchDeleteAction(items: T[]): Promise<IntegrationMessageReturnBatch[]>` |
| NoDeltaBatch | `sendDataInBatch(items: T[]): Promise<IntegrationMessageReturnBatch[]>` |

O nome é `sendDataInBatch`, não `batchSendData`. Os fluxos batch já implementam os métodos individuais herdados;
não é necessário criar stubs de `sendData`, `insertAction`, `updateAction` ou `deleteAction` na automação batch.

`IntegrationMessageReturnBatch` acrescenta `status` a `message` e `data`. O tipo aceita `SUCCESS`, `FAIL` e
`NEUTRAL`; para gravações, use `SUCCESS` somente após confirmação e `FAIL` para falhas. `NEUTRAL` não é falha:
no BatchDelta, inclusões/alterações com esse status também avançam a imagem. Não o use para esconder rejeições.

Regras de correlação:

- Devolva exatamente um resultado por entrada, na mesma ordem. O SDK associa `results[i]` a `items[i]` e não
  confere a cardinalidade. Resultado faltante pode deixar um item sem log e sem estado correspondente.
- Se a API reordenar ou omitir resultados, correlacione pelo identificador documentado e reconstrua a ordem das
  entradas. Não presuma alinhamento por índice sem essa garantia do contrato.
- Em atualização, `oldItems[i]` e `newItems[i]` representam o mesmo registro. Use o identificador do destino
  em `oldItems[i]` quando ele não estiver presente na origem.
- Mantenha falhas parciais como `FAIL` individuais; não marque o lote inteiro como sucesso porque o HTTP foi 200.

Configure `packageSize` como inteiro positivo, conforme o limite do destino e o tamanho do payload.
O padrão do SDK é `1`; não copie valores como 100 ou 500 sem uma razão concreta.

No SDK conferido, uma exceção de `sendDataInBatch` marca todos os itens daquele lote como falha e permite
continuar os lotes seguintes. Uma exceção em `batchInsertAction`, `batchUpdateAction` ou `batchDeleteAction`
interrompe a execução delta. Quando o contrato permitir continuar após uma falha de lote, devolva um `FAIL`
por entrada. Não invente resultados individuais para uma resposta cuja correlação não é conhecida.

## Imagem delta, identidade e exclusão

O fluxo tenta ler a imagem persistida da automação e do ambiente. Quando ela contém registros, usa essa imagem
como `targetSystemData`; `loadTargetSystemData()` só é chamado quando não há imagem não vazia disponível.
Portanto, esse método não consulta necessariamente o destino em toda execução. Alterações manuais no destino
não são automaticamente detectadas pela comparação com a imagem anterior.

A origem deve representar o conjunto completo do escopo comparado. O SDK calcula:

- Inclusão: chave da origem ausente na imagem/destino.
- Atualização: chave presente nos dois lados, com diferença nos campos delta.
- Sem alteração: chave presente nos dois lados, sem diferença nos campos delta.
- Exclusão: chave da imagem/destino ausente na origem carregada.

Uma página incompleta, filtro temporário, evento isolado ou falha de consulta convertida em `[]` pode gerar
exclusões indevidas. Complete a paginação e propague falhas da extração. Para eventos ou consultas incrementais,
use NoDelta quando o requisito for processar apenas os registros recebidos; não trate ausência em um recorte
como exclusão de todo o universo anterior.

Defina a semântica de `deleteAction` pelo escopo: excluir, inativar ou não executar ação no destino são decisões
de negócio diferentes. A obrigatoriedade do método não autoriza excluir dados. Um retorno bem-sucedido de
`deleteAction`, mesmo sem chamar o destino, remove o registro da próxima imagem; se a intenção for preservar
o estado anterior, esse retorno não resolve o requisito. Não limpe filas internas para contornar uma escolha
de fluxo ou de escopo incorreta.

Escolha chaves estáveis e únicas no universo da automação. Não use campos mutáveis, como nome, como identidade.
Normalize os dois lados para o mesmo formato antes da comparação, incluindo datas, códigos e valores opcionais.

Na implementação conferida, chaves compostas e campos delta são comparados por concatenação dos valores,
sem separador. Não presuma comparação estrutural profunda de objetos ou ausência de colisões em componentes
variáveis. Quando relevante, use um campo normalizado e inequívoco para identidade/comparação, ou um hook de
comparação sustentado pelo contrato. Inclua nos campos delta apenas valores que realmente determinam atualização.

### Persistência e identificadores do destino

Depois de uma inclusão individual bem-sucedida, o SDK chama `fillGeneratedId` e adiciona o item à imagem.
Em atualização, chama `fillExternalIdAfterUpdate` e adiciona `newData`. Em falha de atualização ou exclusão,
preserva a versão anterior; uma inclusão que falhou não entra na imagem.

Retornar o ID do destino em `data` não o copia automaticamente para o item. Implemente os hooks quando esse
identificador for necessário em execuções futuras:

- `fillGeneratedId(item: T, data): void`: copiar o ID gerado para `item` após a inclusão.
- `fillExternalIdAfterUpdate(item: {oldData: T; newData: T}, data): void`: preservar o ID em `newData`,
  usando o retorno da atualização ou `oldData` conforme o contrato.

Esses hooks são `protected`, síncronos e vazios por padrão. No batch, só são chamados quando o resultado contém
`data` truthy. O SDK aceita `any` para `data`; na implementação da automação, prefira um tipo de resposta
conhecido ou `unknown` validado na fronteira.

A imagem é salva depois das filas e logs, antes de `postProcessingCustomerRoutines`. Gravações no destino e
salvamento da imagem não formam uma transação: se a execução parar entre essas etapas, uma repetição pode
reexecutar operações já aceitas. Aproveite a idempotência do destino quando disponível e não acrescente retries
de escrita sem definir como reconhecer uma operação já concluída.

## Ciclo de execução e hooks

NoDelta executa: carregar origem → escolher logger → `preProcessingCustomerRoutines` → atualizar metadata →
processar itens/lotes → salvar logs → `postProcessingCustomerRoutines` → atualizar estatísticas.

Delta executa: ler imagem → carregar origem → obter imagem/destino → calcular filas → escolher logger →
`preProcessingCustomerRoutines` → atualizar metadata → processar filas → salvar logs e imagem →
`postProcessingCustomerRoutines` → atualizar estatísticas.

Consequências:

- Inicialização necessária para consultar a origem deve ocorrer antes ou dentro de `loadSourceSystemData`.
  `preProcessingCustomerRoutines` é chamado depois da carga; em Delta, também depois do cálculo das filas.
- `postProcessingCustomerRoutines` pode executar mesmo com falhas individuais. Consulte `hasAnyErrors()`
  antes de avançar checkpoints ou confirmar consumo se o contrato exigir sucesso completo.
- A ordem delta padrão é inclusão → atualização → exclusão → sem alteração. `deleteFirst = true` muda para
  exclusão → inclusão → atualização → sem alteração; use somente quando houver dependência de negócio comprovada.
- Crie uma instância do fluxo por execução. As filas e o indicador de erro pertencem à instância.

Hooks adicionais de Delta, todos `protected`:

| Hook | Uso |
| --- | --- |
| `hasInvalidDelta(sourceSystemItem: T, targetSystemItem: T): boolean` | Comparação customizada quando a comparação padrão não atende ao contrato |
| `preSourceSystemSearchSort()` / `preTargetSystemSearchSort()` | Preparação dos índices usados na comparação |
| `searchItemInSourceSystem(item: T): T \| null` / `searchItemInTargetSystem(item: T): T \| null` | Busca customizada consistente com os índices e as chaves |
| `noDeltaAction(item: T): Promise<void>` | Trabalho opcional em registros sem alteração; não substituir envio obrigatório por esse hook |

`preProcessingCustomerRoutines()` e `postProcessingCustomerRoutines()` retornam `Promise<void>` e existem
nas duas famílias. Não sobrescreva `doIntegration`, métodos `process*Queue`, persistência ou logger para
implementar regras comuns: preserve a coordenação do SDK e use os pontos de extensão correspondentes.
Para configuração de logging, consulte [logging-strategy.md](logging-strategy.md).

## Entrada e encerramento

No handler de produção, use `AutomationExecution.executeAutomation(instance)`. Esse wrapper encaminha o payload,
resolve `payloadRef` quando aplicável, registra informações da execução e comunica falhas por item ao runtime.
Não substitua esse ciclo por um `try/catch` que apenas imprime a exceção e retorna sucesso.

O wrapper chama `doIntegration(executionEvent.payload)`, e esse argumento chega a `loadSourceSystemData(payload)`.
Valide o payload conforme o contrato da automação. `payload` não substitui os campos de identidade e contexto
de `ProcessorPayload`: `tenantId`, `environmentId`, `partitionSuffix`, `automationId`, `executionId`,
`expirationPeriod`, `systems` e `parameters`.

Há uma divergência no SDK conferido: `ProcessorPayload.parameters` é declarado como `GenericParameter[]`,
enquanto os fluxos e `AutomationParameter` consomem `AutomatedIntegrationParameters`, com `custom` para parâmetros
da automação. Confira o evento e a versão instalados; mantenha qualquer adaptação de tipo estritamente na fronteira,
sem espalhar casts nem inventar uma estrutura alternativa. Veja [parameters-management.md](parameters-management.md).

Em testes do fluxo, `doIntegration(payload)` pode ser chamado diretamente com todas as dependências externas
isoladas. Ele pode resolver com falhas por item: confira `hasAnyErrors()` além do resultado da chamada.
`SDK.testMode` não bloqueia todas as chamadas HTTP ou de serviços. Para isolamento e fixtures, consulte
[testing-patterns.md](testing-patterns.md).

## Metadata e verificação

Metadata deve descrever campos existentes no item normalizado e úteis ao acompanhamento. Tipos aceitos:
`TEXT`, `NUMBER`, `DATE`, `DATETIME` e `BOOLEAN`; `hideInTable` é opcional.
Não use os rótulos reservados `Action`, `Status` ou `Message`. Não inclua segredos nos itens ou mensagens de log.

Verifique conforme o comportamento implementado: propagação de falha individual; correlação e falha parcial de
lotes; preservação dos IDs na imagem; paginação e escopo de exclusão; checkpoint diante de falha; repetição de
operações quando pertinente. Use a checagem de tipos do projeto para capturar contratos incorretos, sem converter
exemplos em produção com `as any` ou `@ts-ignore` para silenciar incompatibilidades.
