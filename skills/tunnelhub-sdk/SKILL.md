---
name: tunnelhub-sdk
description: >-
  Cria, modifica e revisa automações com @tunnelhub/sdk seguindo os padrões de estrutura,
  modelo de integração, fluxos delta ou sem delta, clientes externos, conversões,
  parâmetros, monitoramento, testes e empacotamento TunnelHub. Use também para configurar
  CI/CD dessas automações com @tunnelhub/cli.
---

# TunnelHub SDK

Produza automações seguindo os padrões definidos nesta skill. Aplique essa base de estrutura,
modelo de dados e uso do SDK ao implementar o contrato de negócio de cada automação.

## Antes de implementar

Leia as instruções e os arquivos afetados do projeto, a versão instalada de `@tunnelhub/sdk`,
seus scripts e `tunnelhub.yml`. Preserve a organização existente em manutenções; não reorganize
uma automação inteira para aplicar esta skill a uma correção pequena.

Identifique no pedido: origem, destino, registros abrangidos, operações permitidas, identidade,
campos integrados, exclusões do escopo e comportamento de reprocessamento. Não invente um
mapeamento ou contrato de API para completar uma lacuna. Avance no que estiver definido e
pergunte somente quando a informação ausente mudar materialmente a entrega.

Leia [Padrões de implementação](references/implementation-patterns.md) ao criar uma automação
ou alterar seu modelo, mapeamento ou divisão de responsabilidades.

## Estrutura e responsabilidades

| Parte | Responsabilidade |
| --- | --- |
| `src/index.ts` | Receber evento/contexto, criar uma instância de `Integration` e executar `AutomationExecution.executeAutomation`. |
| `src/classes/integration.ts` | Estender o fluxo adequado, conectar os serviços e implementar carga, normalização e ações. |
| `src/types/integration.ts` | Definir `IntegrationModel`, o modelo normalizado usado na comparação e no monitoramento. |
| `src/types/<sistema>.ts` | Tipos e enumerações dos contratos externos efetivamente utilizados. |
| `src/services/<sistema>Service.ts` | Encapsular transporte, autenticação, paginação e validação da resposta daquele sistema. |
| `src/classes/conversions.ts` | Conversões de domínio reutilizadas ou que precisam de teste isolado; criar quando necessário. |
| `src/metadata.ts` | Definir as colunas operacionais, validadas com `satisfies Metadata<IntegrationModel>[]`. |

Essa é a organização padrão para uma automação nova. Preserve caminhos equivalentes já adotados,
como `src/core/integration.ts`, `src/integration.ts`, serviços `*Api.ts` e `util/conversions.ts`.
Crie somente os arquivos necessários. Clientes SOAP gerados ficam separados do código de negócio.
Não introduza frameworks internos, fábricas, repositórios genéricos ou camadas que só repassam chamadas.

## Modelo e transformação

- Use um `IntegrationModel` normalizado com campos de negócio explícitos, compartilhado pela carga
  da origem e do destino. Nos campos de identidade e delta, prefira valores escalares comparáveis.
- Mantenha separados o tipo recebido da origem, o modelo de integração e o payload exigido pelo destino.
  Não use o envelope HTTP/SOAP nem objetos `*WsVo` como o modelo de comparação.
- Normalize datas, códigos, documentos e opcionais na carga. Monte o payload do destino na ação de
  envio, com uma função ou método explícito quando necessário.
- Declare `keyFields` e `deltaFields` explicitamente e tipados. Chaves identificam o registro;
  delta contém os campos de negócio cuja mudança exige atualização. Não derive delta de
  `Object.keys`, não inclua respostas, timestamps de execução ou credenciais.
- Não sobrescreva `hasInvalidDelta` apenas para acomodar um modelo de payload aninhado.
  Primeiro mantenha os campos comparados no modelo normalizado. Use extensão do SDK somente para
  uma necessidade real do contrato que o padrão não atende, com validação correspondente.
- Metadata acompanha o modelo e oferece identificação, contexto e campos úteis à investigação.
  Não resuma toda a informação operacional em um campo contendo JSON.

## Fluxos e execução

| Requisito | Fluxo |
| --- | --- |
| Comparar o conjunto de origem com a imagem anterior | `DeltaIntegrationFlow` |
| Mesmo requisito com API de destino que recebe lotes | `BatchDeltaIntegrationFlow` |
| Processar cada registro recebido, sem comparação entre execuções | `NoDeltaIntegrationFlow` |
| Mesmo requisito com envio em lote | `NoDeltaBatchIntegrationFlow` |

Leia [Contratos dos fluxos](references/integration-flows.md) ao implementar ou alterar esses métodos.
Volume sozinho não determina o uso de batch. Delta não significa sincronização bidirecional.

Deixe iteração, filas, cálculo de delta, persistência da imagem, logs e estatísticas com o SDK.
Não reimplemente `doIntegration` ou `process*Queue`, nem crie um motor paralelo no handler.
Use `AutomationExecution.executeAutomation` em produção; `doIntegration` direto cabe em teste isolado.

Implemente a carga inicial do destino no mesmo modelo quando houver comparação delta com cadastros
existentes. Não substitua essa leitura por `[]` como atalho. Esse retorno só cabe se a primeira carga
sem leitura for parte do contrato, com identidade e repetição seguras. Uma extração incompleta ou
uma falha de consulta nunca deve se passar por uma lista vazia bem-sucedida.

## Configuração e recursos da plataforma

| Necessidade | Padrão |
| --- | --- |
| Conexão e credenciais | Sistemas vinculados em `this.systems`; valide nome interno, tipo e configuração necessária. |
| Comportamento configurável do fluxo | `AutomationParameter` com `this.parameters`; parâmetros apenas quando necessários ao escopo. |
| De/para mantido pela operação | `ConversionTable` com `DataStore`; não substitua por um grande parâmetro JSON de mapeamento. |
| Conversão fixa de formato ou enumeração documentada | Função explícita de conversão, sem torná-la configuração artificial. |
| Dados relacionais de apoio | `SqlTables`, quando houver essa necessidade e o runtime compatível. |
| Numeração sequencial | `Sequences`, quando prevista pelo negócio. |

Não implemente acesso direto às tabelas internas para reproduzir esses recursos. Não faça scan
na tabela `TunnelHub-prd`. Para cursores, confirme a persistência suportada pela versão instalada
e só avance sobre dados confirmados; veja [Parâmetros](references/parameters-management.md).

## Falhas e monitoramento

Ações individuais retornam `{message, data}` em sucesso e lançam exceção em falha.
`success: false` não pertence a `IntegrationMessageReturn` e não informa falha ao SDK.
Não capture um erro para devolver uma mensagem de falha como retorno bem-sucedido.

Em batch, retorne um `IntegrationMessageReturnBatch` por entrada, na mesma ordem, com `status`
correto; preserve falhas parciais. O método de NoDeltaBatch é `sendDataInBatch`.

O cliente deve conferir erros HTTP e rejeições de negócio, inclusive em respostas HTTP 200.
Não converta falha em `[]`, `{}` ou sucesso. Retentativas de escrita precisam respeitar a
idempotência do destino. Preserve o padrão de erros do cliente sem registrar credenciais,
envelopes sensíveis ou o evento completo.

O SDK já registra o resultado por item. Use mensagens operacionais claras e mantenha a estratégia
padrão de logging, salvo necessidade demonstrada. Interceptação HTTP não substitui os logs do fluxo
nem garante mascaramento de corpos; consulte a referência antes de habilitá-la.

## Verificação da entrega

Antes de concluir, revise o código contra os padrões desta skill: modelo separado dos payloads,
uso dos recursos nativos, operações contratadas implementadas e falhas visíveis ao SDK.
Um método obrigatório vazio não conta como implementação de uma operação prevista no escopo.

Execute a checagem de tipos e os testes pertinentes existentes. Acrescente regressão quando
houver risco em transformação, delta, paginação, falhas parciais ou persistência. Não crie testes
que substituem a própria regra por um mock. `SDK.testMode` não bloqueia todas as chamadas externas;
leia [Testes](references/testing-patterns.md) antes de executar o fluxo em teste.

Preserve o build existente; em projetos novos, use o empacotamento do SDK (`tunnelhub-sdk-esbuild`)
quando compatível. Confira runtime, entrypoint e ZIP indicado em `package.artifact`. SDK 4.1.5 e os
templates atuais usam Node.js 24; confirme a compatibilidade da versão efetivamente utilizada.
Build e testes com mocks não comprovam homologação nos sistemas do cliente.

Para configurar deploy, leia [CI/CD](references/ci-cd-deployments.md). Gere ou ajuste
`.github/workflows/deploy-tunnelhub.yml`; execute `tunnelhub deploy-automation` somente pelo
workflow, usando as credenciais CI/CD previstas. Implementar uma automação não implica executar
uma carga real, ativar agendamento ou fazer deploy sem que isso faça parte do pedido.

## Referências por necessidade

Não carregue todas as referências de uma vez. As declarações e a implementação do SDK instalado
prevalecem sobre um exemplo de outra versão; não silencie incompatibilidades com `as any`.

- [Padrões de implementação](references/implementation-patterns.md): organização, modelo, transformação e exemplo mínimo.
- [Fluxos](references/integration-flows.md): contratos, imagem delta, ações, batch e hooks.
- [Parâmetros](references/parameters-management.md): leitura, configuração e checkpoints.
- [Sistemas](references/system-configuration.md): tipos de conexão e autenticação.
- [Dados de apoio](references/data-operations.md): conversões, CRUD e sequências.
- [SQL Tables](references/sql-tables.md): contrato relacional e requisitos de runtime.
- [Logging](references/logging-strategy.md): escolha de estratégia e ajustes.
- [HTTP](references/http-interceptor.md): configuração e limites da interceptação.
- [Utilitários](references/utilities.md): concorrência, validação de metadata e flags do SDK.
- [Testes](references/testing-patterns.md): isolamento e verificações de comportamento.
- [CI/CD](references/ci-cd-deployments.md): credenciais, artefato e workflow de publicação.
