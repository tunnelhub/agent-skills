# Utilitários do SDK

Use os utilitários abaixo quando houver uma necessidade concreta no fluxo. Contratos conferidos em `src/util/promises.ts`, `src/util/validations.ts` e `src/index.ts` do SDK 4.1.5; confira a versão instalada ao modificar um projeto existente.

## Promessas independentes

`promiseAllSettled(promises)` recebe um array de promessas e retorna resultados na ordem da entrada. Cada resultado tem `status: 'fulfilled'` e `value`, ou `status: 'rejected'` e `reason`. Rejeições das promessas recebidas viram resultados; não fazem o conjunto rejeitar automaticamente.

Use quando precisar aguardar **todas** as operações independentes e avaliar cada falha. Não use como forma de ignorar erros. O helper não limita concorrência, não faz retentativas e não cancela operações iniciadas. Se `Promise.allSettled` já atende ao projeto, não precisa trocá-lo apenas para usar um helper do SDK.

Exemplo completo para leituras independentes que devem todas ter sucesso:

```typescript
import {promiseAllSettled} from '@tunnelhub/sdk';

export async function readAll<T>(readers: Array<() => Promise<T>>): Promise<T[]> {
  const results = await promiseAllSettled(readers.map(read => read()));
  const values: T[] = [];
  const errors: unknown[] = [];

  for (const result of results) {
    if (result.status === 'fulfilled') {
      values.push(result.value);
    } else {
      errors.push(result.reason);
    }
  }

  if (errors.length > 0) {
    throw new AggregateError(errors, 'Uma ou mais leituras falharam.');
  }
  return values;
}
```

Não chame `insertAction`, `updateAction`, `deleteAction` ou `sendData` manualmente para paralelizar filas que o fluxo já controla. Isso contorna registros por item, indicadores de falha e atualização de delta. Se o destino suporta lotes, escolha o fluxo batch e implemente seu contrato de resultados; consulte [integration-flows.md](integration-flows.md).

## Concorrência limitada

`promiseWithConcurrency({array, mapFunction, options})` usa `Readable.from(array).map(...)` e retorna um stream. O trabalho precisa ser consumido, por exemplo com `for await`. A entrada continua sendo um array em memória; esse helper não implementa paginação da origem.

- `mapFunction(item, options?)` é assíncrona e pode receber `options.signal`.
- `options.concurrency` limita o trabalho simultâneo. Escolha-o pelo contrato e pelos limites do serviço, sem um valor universal.
- `options.signal` permite cancelamento do stream. Propague o sinal para a operação externa quando ela o aceitar; um cancelamento não desfaz uma gravação já confirmada.
- Uma rejeição do mapper faz o consumo falhar; não há conversão automática para resultados do tipo `allSettled`.
- Limitar concorrência não equivale a limitar requisições por segundo. Se houver uma cota temporal, trate-a conforme o serviço.

Exemplo completo de enriquecimento por leituras, preservando a gestão de envios do fluxo:

```typescript
import {promiseWithConcurrency} from '@tunnelhub/sdk';

export async function enrichItems<T, R>(
  items: T[],
  readDetails: (item: T, signal?: AbortSignal) => Promise<R>,
  concurrency: number,
): Promise<R[]> {
  const results: R[] = [];
  const stream = promiseWithConcurrency({
    array: items,
    mapFunction: (item, options) => readDetails(item, options?.signal),
    options: {concurrency},
  });

  for await (const result of stream) {
    results.push(result);
  }
  return results;
}
```

Esse helper só é útil se o fluxo realmente precisar desse enriquecimento. Não crie uma camada genérica de concorrência para uma integração sequencial simples. Se a concorrência vier de entrada externa, valide que seja um inteiro positivo antes de usá-la.

## Validação de metadados

`validateMetadata(metadata)` exige um `fieldLabel` presente e rejeita os valores exatos `Action`, `Status` e `Message`. `validateMetadataArray(metadataArray)` exige um array e aplica a mesma validação, identificando o índice inválido no erro.

Essas funções **não** validam dados da origem, tipos dos valores, existência de `fieldName` no objeto, unicidade de colunas ou o contrato de uma API.

```typescript
import {validateMetadataArray} from '@tunnelhub/sdk';
import type {Metadata} from '@tunnelhub/sdk';

type IntegrationModel = {externalId: string; active: boolean};

const metadata: Metadata<IntegrationModel>[] = [
  {fieldName: 'externalId', fieldLabel: 'Identificador externo', fieldType: 'TEXT'},
  {fieldName: 'active', fieldLabel: 'Ativo', fieldType: 'BOOLEAN'},
];

validateMetadataArray(metadata);
```

Use os metadados do projeto e mantenha a validação no ponto apropriado de sua preparação. Não introduza um hook `beforeIntegration()` ou `afterIntegration()` para isso: esses hooks não existem nos fluxos. Veja a ordem de execução em [integration-flows.md](integration-flows.md).

## Configuração global do SDK

| Recurso | Efeito real |
|---|---|
| `SDK.verbose` | Booleano global, inicialmente `false`; habilita mensagens emitidas por `SDK.log`. Não silencia `console.log` usado diretamente pelo SDK ou pela automação. |
| `SDK.log(message)` | Aceita uma string e a envia ao console somente quando `SDK.verbose` é verdadeiro. Não cria resultado de processamento por item. |
| `SDK.testMode` | Booleano global, inicialmente `false`; alguns componentes o consultam para pular persistência. Não bloqueia rede ou sistemas externos de forma geral. |
| `SDK.setLocale(locale)` | Método assíncrono para selecionar o idioma das mensagens internas; confira os idiomas suportados na versão instalada. |

Não use `SDK.testMode` como modo de simulação de negócio ou garantia de teste sem efeitos externos. Os métodos da integração continuam sendo executados. Leia [testing-patterns.md](testing-patterns.md) para simular as fronteiras corretas e restaurar flags globais após cada teste.

Mensagens devem trazer contexto útil sem despejar eventos, credenciais ou respostas integrais. Use `message` e os metadados dos resultados do fluxo para o acompanhamento operacional; `SDK.log` é diagnóstico complementar. Para requisições HTTP, consulte [http-interceptor.md](http-interceptor.md).
