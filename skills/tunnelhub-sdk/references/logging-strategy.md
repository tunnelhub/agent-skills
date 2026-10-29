# Estratégia de registros da execução

Os fluxos já escolhem entre persistência `realtime` por DynamoDB e envio `batch` por Firehose. Preserve essa escolha automática em novas automações. Ajuste somente para resolver uma necessidade observada, com evidência e validação proporcionais; não acrescente configurações de desempenho por antecipação.

Este documento descreve a implementação do SDK 4.1.5. Os valores são limiares da decisão no código, não garantias de latência, durabilidade ou custo.

## Separar processamento de persistência

- O tipo de fluxo define como enviar dados ao destino: por item ou em lotes.
- A estratégia de logging define como persistir os registros da execução.
- Um fluxo individual também pode usar logging `batch`; um fluxo batch também pode usar logging `realtime`.
- O fluxo cria seu `LogManager`, salva logs, atualiza metadados e estatísticas. Não replique essas tarefas na automação.

A aceitação de registros pelo Firehose não confirma que já estejam disponíveis na consulta da plataforma. Não prometa um tempo fixo de disponibilização sem evidência da infraestrutura em uso.

## Decisão padrão

A decisão ocorre após a carga e preparação das filas, antes de `preProcessingCustomerRoutines()`. Ela é calculada uma vez nessa etapa; não se ajusta continuamente à duração medida durante o envio.

### Fluxos Delta

Para `DeltaIntegrationFlow` e `BatchDeltaIntegrationFlow`, o total é a soma das filas de inserção, atualização, exclusão e itens sem mudança. A proporção sem mudança é `noDeltaQueue.length / total`; para total zero, o método retorna zero.

Com as propriedades padrão, a decisão é:

| Condição | Estratégia |
|---|---|
| Total maior que 1.000 | `batch` |
| Total até 100 | `realtime` |
| Total entre 101 e 500 e proporção sem mudança de pelo menos 80% | `realtime` |
| Total entre 101 e 500 e `isKnownFastIntegration()` retorna `true` | `realtime` |
| Demais casos | `batch` |

`isKnownFastIntegration()` retorna `false` por padrão. Sobrescrevê-lo com `true` não força `realtime` acima de 500 registros nem ignora a verificação de `maxRealtimeItems`.

### Fluxos sem Delta

Para `NoDeltaIntegrationFlow` e `NoDeltaBatchIntegrationFlow`, o total é `dataQueue.length`.

| Condição, com as propriedades padrão | Estratégia |
|---|---|
| Total até 500 | `realtime` |
| Total maior que 500 | `batch` |

A implementação primeiro verifica `maxRealtimeItems`, depois `realtimeLoggingThreshold`, e por último a condição intermediária até 500 itens. Em NoDelta, `isKnownFastIntegration()` não amplia esse limite intermediário: a condição final também exige total até 500.

## Propriedades existentes

| Propriedade protegida | Padrão | Efeito verificado |
|---|---:|---|
| `realtimeLoggingThreshold` | 100 | Até esse total, retorna `realtime`, desde que a checagem anterior de `maxRealtimeItems` não tenha retornado `batch`. |
| `maxRealtimeItems` | 1.000 | Acima desse total, retorna `batch` antes das outras condições. |
| `highNoDeltaRatioThreshold` | 0,8 | Apenas Delta: proporção usada na condição intermediária, que também exige total até 500. |
| `maxRealtimeDuration` | 30 | Declarada nos fluxos, mas não é consultada por `defineLoggingStrategy()` nessa versão. Alterá-la não muda a decisão. |

O limite intermediário de 500 está no corpo do método, não em uma propriedade configurável. Aumentar apenas `maxRealtimeItems` não torna automaticamente volumes maiores elegíveis a `realtime`. Reduzir apenas `realtimeLoggingThreshold` também não remove as condições intermediárias.

Não use `executionEvent.metadata` para essa decisão: esse campo não faz parte de `ProcessorPayload`. Não invente parâmetros da plataforma para propriedades internas do SDK.

## Quando houver necessidade de ajuste

Primeiro confirme o volume, a composição das filas e o caminho escolhido na versão instalada. Diferencie uma falha de envio ao destino de uma falha de persistência de logs ou de disponibilidade posterior na plataforma.

Se a evidência justificar um ajuste, altere apenas a propriedade necessária na classe existente e registre seu motivo. A configuração precisa estar pronta antes de `defineLoggingStrategy()`; os hooks de pré e pós-processamento ocorrem tarde demais para alterar a decisão daquela execução. Não existem hooks `beforeIntegration()` ou `afterIntegration()` nesses fluxos.

Evite sobrescrever `defineLoggingStrategy()` ou os métodos de fila para forçar um resultado. Isso elimina proteções e exige assumir a manutenção do algoritmo. Uma necessidade que não caiba no contrato existente deve ser tratada explicitamente, sem copiar o motor do SDK para a automação.

Para validar uma alteração real de limiar, teste os valores imediatamente abaixo, no limite e acima dele. Em Delta, cubra também a proporção sem mudança envolvida. Esses testes podem verificar a estratégia escolhida sem escrever em DynamoDB ou Firehose. Não introduza testes só para repetir os padrões inalterados do SDK.

## Diagnóstico

`SDK.verbose = true` habilita as mensagens de escolha emitidas por `SDK.log`, incluindo tamanho da fila e, em Delta, proporção sem mudança. Ative-o apenas quando necessário ao diagnóstico.

O `LogManager` recebe `debug` de `executionEvent.debug`. As mensagens de debug internas desse gerenciador também passam por `SDK.log`, portanto dependem de `SDK.verbose`. Já mensagens que o SDK envia diretamente ao console não dependem dessa flag.

O fluxo atualiza `loggingStrategy` no registro da execução quando seleciona `batch`. Essa persistência faz parte do SDK; não deve ser reproduzida com comandos próprios na automação.

`LogManager.saveBatch()` pula a gravação quando `SDK.testMode` é verdadeiro, mas isso não torna toda a execução isolada. A construção dos componentes e outros caminhos podem continuar ativos; simule as fronteiras exigidas pelo teste. Consulte [testing-patterns.md](testing-patterns.md).

Ao investigar falha de Firehose, confira a configuração do ambiente gerenciado, incluindo `TH_LOGS_FIREHOSE_STREAM_NAME`, e as permissões aplicáveis. Não fixe nomes de recursos de outro ambiente no código nem crie infraestrutura paralela apenas para contornar a falha.

## Conteúdo dos registros

Deixe o fluxo produzir os registros por item a partir dos retornos e exceções de suas ações. Um `console.log` ou `SDK.log` não substitui esse contrato, e um texto de erro devolvido como sucesso continuará sendo registrado como sucesso. Consulte [integration-flows.md](integration-flows.md).

Escolha campos úteis no modelo e nos metadados sem carregar credenciais, tokens ou documentos desnecessários. `hideInTable` altera a exibição; não é remoção do dado persistido nem controle de acesso. A proteção dos registros HTTP é independente da estratégia descrita aqui; consulte [http-interceptor.md](http-interceptor.md).
