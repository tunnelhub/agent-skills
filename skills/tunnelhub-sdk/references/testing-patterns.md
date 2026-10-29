# Testes de automações

Teste o comportamento da automação com o SDK real e substitua as dependências externas necessárias. Reutilize o
executor e os padrões de teste do projeto. Não adicione Jest, Faker ou outra infraestrutura apenas para acompanhar
uma alteração que pode ser verificada com os recursos existentes.

## Escolher o que verificar

Comece pelo risco da mudança e execute os testes diretamente relacionados. Uma mudança de regra, validação,
persistência ou tratamento de falha pode justificar regressão nova; uma alteração editorial não exige teste novo.

| Comportamento alterado | Verificação útil |
| --- | --- |
| Mapeamento | Entrada representativa produz o payload esperado, incluindo campos opcionais e valores inválidos relevantes. |
| Paginação | Todas as páginas esperadas são consumidas; falha intermediária não vira extração completa. |
| Envio individual | Falha do cliente chega ao fluxo como exceção e `hasAnyErrors()` indica erro após o processamento. |
| Envio em lote | Sucessos e falhas ficam associados aos itens corretos, na ordem exigida pelo fluxo. |
| Delta | Casos pertinentes de inclusão, alteração, ausência de mudança e exclusão respeitam o escopo contratado. |
| Checkpoint | Não avança sobre registros que falharam ou páginas que não foram processadas. |
| Configuração obrigatória | Ausência ou valor inválido interrompe antes de escrever no destino. |

Selecione apenas os casos pertinentes à alteração. Não refaça na automação toda a suíte de testes interna do SDK.

## `SDK.testMode` não isola todas as dependências

O modo de teste evita algumas operações internas de logs, estatísticas e imagens de delta. Ele **não** transforma
todos os clientes em mocks e não bloqueia a rede.

Na implementação consultada:

- Clientes HTTP/SOAP, bancos e arquivos usados pela automação continuam executando normalmente.
- `DataStore`, `Sequences` e os métodos de instância de `AutomationParameter` podem acessar DynamoDB.
- `AutomationExecution.executeAutomation()` consulta um serviço externo de IP; também pode resolver `payloadRef`
  via S3 e enviar callbacks ao Step Functions conforme o evento.
- `SqlTables` depende de metadados, `node:sqlite` e do caminho configurado para as tabelas.

Inspecione os caminhos efetivamente chamados pela versão instalada e faça mock dessas fronteiras antes de executar
o teste. Não use credenciais, eventos capturados ou documentos reais como fixtures. Nunca faça scan da tabela
`TunnelHub-prd` para preparar testes ou descobrir configuração.

Restaure o estado global após o teste. Este exemplo é completo para a configuração de uma suíte Jest que já existe:

```typescript
import {SDK} from '@tunnelhub/sdk';

let previousTestMode: boolean;

beforeEach(() => {
  previousTestMode = SDK.testMode;
  SDK.testMode = true;
});

afterEach(() => {
  SDK.testMode = previousTestMode;
  jest.restoreAllMocks();
});
```

Restaure também `SDK.verbose`, variáveis de ambiente e substituições globais se o teste as alterar. O modo de teste
deve estar ativo antes de criar ou executar o fluxo que precisa dele.

## Fixtures compatíveis com o contrato instalado

Inclua os campos obrigatórios, sem usar `as ProcessorPayload` para ocultar um evento incompleto. Este evento mínimo
é compatível com a declaração de `ProcessorPayload` do SDK 4.1.5 e serve para casos sem parâmetros customizados:

```typescript
import type {ProcessorPayload} from '@tunnelhub/sdk';

const event: ProcessorPayload = {
  tenantId: 'test-tenant',
  environmentId: 'test-environment',
  partitionSuffix: 'test-partition',
  automationId: 'test-automation',
  executionId: 'test-execution',
  expirationPeriod: 30,
  systems: [],
  parameters: [],
};
```

Preencha `systems` com fixtures sintéticas dos sistemas exigidos pela automação. Para testar parâmetros customizados,
confira também o contrato em execução: nessa versão, `ProcessorPayload.parameters` é declarado como
`GenericParameter[]`, enquanto os helpers estáticos de `AutomationParameter` leem `AutomatedIntegrationParameters.custom`.
Essa divergência do SDK não deve ser escondida com casts espalhados ou com um formato de configuração inventado.
Use o tipo do helper ao testá-lo isoladamente; se o teste do fluxo precisar cruzar essa fronteira, documente e limite
a adaptação ao ponto necessário, conforme a versão realmente instalada.

## Manter o comportamento testado real

- Importe a classe de integração de produção e simule as respostas dos clientes que ela usa.
- Para testar o mapeamento, não substitua o método que mapeia. Para testar falhas de envio, não substitua a ação do
  fluxo por uma implementação diferente criada no teste.
- Verifique payloads enviados, chamadas que devem ou não ocorrer e resultados de erro. Apenas contar logs ou afirmar
  que uma execução vazia terminou não demonstra que a regra foi implementada.
- Se precisar acessar um método protegido, prefira exercitá-lo pelo fluxo público. Uma subclasse de teste pode expor
  uma chamada existente quando necessário, mas não deve reimplementar a regra sob teste.
- Um teste do fluxo pode chamar `doIntegration(payload)` diretamente para verificar a classe. O handler de produção
  deve continuar usando `AutomationExecution.executeAutomation()`; teste esse encaminhamento separadamente quando
  ele for parte da mudança.

Em ações individuais, uma resposta resolvida `{message: 'Falhou', data: null}` não sinaliza falha ao SDK. Configure
o mock do cliente para rejeitar e confira o erro observado no fluxo. Em lote, use os status previstos pelo contrato
real; consulte [integration-flows.md](integration-flows.md) para nomes de métodos e regras de correspondência.

Para HTTP, simule tanto a resposta de transporte quanto os erros de negócio relevantes. `fetch` pode resolver com
HTTP 400/500; o teste deve exercitar a checagem feita pelo cliente real. Se testar esse cliente, faça mock do transporte;
se testar a orquestração da integração, faça mock do cliente e mantenha a orquestração real.

## SQL Tables e persistência

Em testes de regras de negócio, faça mock do helper ou da fronteira de persistência já existente. Não crie uma camada
genérica de repositórios só para permitir o mock. Quando a própria operação SQL for o comportamento testado, use
Node.js compatível com `node:sqlite`, metadados sintéticos e um diretório temporário isolado. Configure apenas as
variáveis necessárias ao caso e remova os arquivos temporários ao terminar.

Para acesso AWS real do SDK, use os mocks já adotados pelo projeto, como `aws-sdk-client-mock`, ou substitua o helper
na fronteira da automação. O `SDK.testMode` sozinho não substitui esse isolamento.

## Conclusão da validação

Execute a checagem de tipos e os testes pertinentes disponíveis no projeto; confira também o build quando afetado.
Antes de executar uma suíte ou colocá-la no CI, revise testes de debug e de integração para identificar chamadas
reais. Relate o que foi validado e as limitações. Testes com mocks não confirmam conectividade, credenciais ou
homologação funcional nos sistemas do cliente.
