# Parâmetros da automação

Use esta referência ao ler configuração ou persistir um cursor de processamento. Os contratos abaixo foram conferidos no código do SDK 4.1.5; confirme a versão instalada antes de adaptar uma integração existente.

## Onde guardar cada informação

| Informação | Recurso esperado |
|---|---|
| URL, autenticação e credenciais de conexão | Sistema vinculado à automação; consulte [system-configuration.md](system-configuration.md) |
| Filtros, códigos e regras configuráveis do fluxo | Parâmetros da automação, em `custom` |
| De/para de valores | `ConversionTable`/`DataStore`; consulte [data-operations.md](data-operations.md) |
| Cursor entre execuções | Persistência compatível com o contrato da origem e com a versão do SDK, após validar leitura e escrita |

Não duplique credenciais em parâmetros de negócio quando o sistema já fornece a conexão. Não crie parâmetros para constantes internas ou comportamentos que o escopo não torna configuráveis.

## Leitura estática

Dentro de uma classe de fluxo, use os métodos estáticos com `this.parameters`. Essa leitura não consulta o banco; usa a configuração recebida na execução.

```typescript
import {AutomationParameter} from '@tunnelhub/sdk';
import type {AutomatedIntegrationParameters} from '@tunnelhub/sdk';

const parameters: AutomatedIntegrationParameters = {
  custom: [
    {name: 'company_code', value: 'EMPRESA-01'},
    {name: 'include_inactive', value: 'false'},
  ],
};

const companyCode = AutomationParameter.getRequiredParameter(parameters, 'company_code');
const includeInactive = AutomationParameter.getBooleanParameter(parameters, 'include_inactive', false);
const optionalFilter = AutomationParameter.getParameter(parameters, 'optional_filter');
```

Os nomes acima ilustram configuração de negócio, não parâmetros exigidos pelo SDK.

| Método | Comportamento verificado |
|---|---|
| `getParameter(parameters, name)` | Procura `parameters.custom` pelo nome. A assinatura declara `string \| null`, mas a implementação retorna `undefined` quando não encontra o parâmetro. |
| `getRequiredParameter(parameters, name)` | Lança erro quando o registro não existe. Não rejeita automaticamente `value: ''` nem valida seu conteúdo. |
| `getBooleanParameter(parameters, name, defaultValue = true)` | Valores `'1'` ou `'true'` sem distinção de maiúsculas retornam `true`; qualquer outra string não vazia retorna `false`. Ausência ou string vazia usa o padrão. Não remove espaços. |

Valide apenas o domínio necessário: código obrigatório não vazio, inteiro positivo, data ou enumeração admitida pela API. Não esconda configuração inválida com um valor padrão que altera silenciosamente o escopo. Para opcionais, `value ?? defaultValue` aceita tanto `null` quanto `undefined` e preserva uma string vazia quando ela for válida.

Exemplo completo de validação de um parâmetro numérico opcional:

```typescript
import {AutomationParameter} from '@tunnelhub/sdk';
import type {AutomatedIntegrationParameters} from '@tunnelhub/sdk';

export function readPageSize(parameters: AutomatedIntegrationParameters): number {
  const rawValue = AutomationParameter.getParameter(parameters, 'page_size');
  if (rawValue == null) {
    return 100;
  }

  const pageSize = Number(rawValue);
  if (!Number.isInteger(pageSize) || pageSize <= 0) {
    throw new Error('O parâmetro page_size deve ser um inteiro positivo.');
  }
  return pageSize;
}
```

O valor padrão e o limite máximo, se houver, devem vir do contrato da API usada. Este exemplo pressupõe que ela aceita páginas de 100 registros.

### Divergência de tipos do evento

No SDK 4.1.5, `ProcessorPayload.parameters` está declarado como `GenericParameter[]`, enquanto os fluxos mantêm `this.parameters: AutomatedIntegrationParameters` e os leitores procuram `{custom: GenericParameter[]}`. Os construtores atribuem `event.parameters` diretamente; não convertem o array para esse objeto.

Consequência: uma fixture com `parameters: [{name, value}]` pode satisfazer a declaração de `ProcessorPayload` e, ainda assim, não alimentar os leitores estáticos. Uma fixture com `{custom: [...]}` expressa o formato lido, mas conflita com aquela declaração.

Confira a declaração e o evento efetivo da versão do projeto. Não use `as ProcessorPayload`, `as any` ou `@ts-ignore` para fingir que a incompatibilidade está resolvida. Registre a divergência quando ela afetar a tarefa; uma eventual adaptação deve ficar na fronteira do evento, validando o formato realmente recebido, sem espalhar coerções pelo fluxo. Para testar regras de configuração isoladamente, use `AutomatedIntegrationParameters`, como nos exemplos acima.

## Persistência por instância

A instância recebe os identificadores da execução:

```typescript
import {AutomationParameter} from '@tunnelhub/sdk';
import type {ProcessorPayload} from '@tunnelhub/sdk';

export function createParameterManager(event: ProcessorPayload): AutomationParameter {
  return new AutomationParameter(event.tenantId, event.automationId, event.environmentId);
}
```

Contratos públicos:

- `getParameter(name, defaultValue?)`: retorna `Promise<string | null>`; o padrão é `null`.
- `saveParameter(name, value)`: recebe uma string e retorna `Promise<boolean>`; pode lançar erro. Não trate `false` como confirmação de gravação.

Esses métodos fazem operações reais de DynamoDB e não verificam `SDK.testMode`. Em testes unitários, simule essa fronteira explicitamente.

### Limitação verificada de leitura e escrita

Na implementação 4.1.5 consultada, o método de leitura busca a chave `#INTEGRATION#<automationId>`, enquanto `saveParameter` lê e grava `INTEGRATION#<automationId>`, sem o primeiro `#`. Portanto, não presuma que um valor salvo será recuperado pelo método de leitura nessa versão.

Antes de adotar esse par como checkpoint, confira o código da versão instalada e valide o ciclo de salvar e ler em um teste isolado. Se a divergência existir, registre o bloqueio e trate a correção no SDK ou utilize um recurso já adotado pelo projeto com contrato comprovado. Não espalhe acesso direto à tabela para contornar o problema, nem faça varreduras em produção para investigar.

`saveParameter` lê a configuração e depois substitui o objeto de parâmetros. Não oferece atualização condicional de cursor, bloqueio ou garantia contra gravações concorrentes. Se execuções sobrepostas forem possíveis, resolva a coordenação antes de usar esse mecanismo como estado compartilhado.

## Cursores e reprocessamento

Um checkpoint representa dados confirmados no destino. Ele não representa simplesmente o horário em que uma execução terminou.

- Defina o limite da consulta a partir do contrato da origem. Quando a API usa janela temporal, retenha o limite superior consultado; salvar a hora de término pode saltar alterações ocorridas durante o processamento.
- Complete a paginação e confirme os envios antes de avançar o checkpoint. Falha de página não é fim normal da listagem.
- Em um checkpoint global, uma falha de item impede o avanço, salvo se o projeto já tiver um mecanismo comprovado de retenção e reprocessamento dos itens pendentes.
- Uma falha ao persistir o checkpoint também é falha. Como os envios podem já ter ocorrido, a próxima execução precisa ser segura para repetição conforme o contrato do destino.
- Não adicione retentativas cegas a operações de criação. Use a chave de idempotência ou consulta de confirmação oferecida pelo destino quando necessária.

Os hooks reais são `preProcessingCustomerRoutines()` e `postProcessingCustomerRoutines()`. O primeiro ocorre depois da carga da origem; não serve para inicialização necessária àquela carga. O segundo pode ser chamado após falhas individuais registradas pelo fluxo. Não existe hook `afterIntegration()` nesses fluxos.

Trecho para uma classe que **já possui** um gerenciador de persistência validado e um `nextCursor` calculado pela carga da origem:

```typescript
protected async postProcessingCustomerRoutines(): Promise<void> {
  if (this.hasAnyErrors() || this.nextCursor === undefined) {
    return;
  }

  const saved = await this.parameterManager.saveParameter('source_cursor', this.nextCursor);
  if (!saved) {
    throw new Error('Não foi possível persistir o cursor da origem.');
  }
}
```

Esse trecho é parcial: não cria um mecanismo de checkpoint nem resolve a divergência de chaves descrita acima. Reutilize-o apenas quando o projeto cumprir essas condições. Consulte [integration-flows.md](integration-flows.md) para a ordem dos hooks e [testing-patterns.md](testing-patterns.md) para validar falhas parciais sem acessar sistemas reais.
