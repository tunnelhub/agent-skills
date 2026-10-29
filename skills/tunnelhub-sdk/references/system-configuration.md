# Sistemas vinculados à automação

Use `this.systems` para acessar as conexões recebidas pelo fluxo. O SDK já fornece esses sistemas; não consulte novamente a plataforma para localizar uma conexão vinculada nem replique suas credenciais em parâmetros da automação.

Os contratos abaixo foram conferidos em `src/types/sdk.ts` e `src/platform/system.ts` do SDK 4.1.5. O formato de autenticação exigido por uma API externa continua sendo parte do contrato daquela integração.

## Seleção e validação

Selecione pelo `internalName` configurado, valide existência e `type`, e só então acesse os parâmetros específicos. Em sistemas HTTP, SOAP, SFTP ou banco, valide também o discriminador aplicável (`authType`, `databaseType` ou `oracleConnectionType`). Não use `as TunnelHubSystem` para esconder um sistema ausente ou incompatível.

Trecho para o construtor ou preparação da classe de fluxo existente, quando o destino contratado é HTTP com BASIC:

```typescript
const targetSystem = this.systems.find(system => system.internalName === 'TARGET_ERP');
if (!targetSystem || targetSystem.type !== 'HTTP') {
  throw new Error('O sistema TARGET_ERP deve estar vinculado e ser do tipo HTTP.');
}

const parameters = targetSystem.parameters;
if (parameters.authType !== 'BASIC') {
  throw new Error('O sistema TARGET_ERP deve usar autenticação BASIC.');
}
if (!parameters.user || !parameters.password) {
  throw new Error('Configure usuário e senha do sistema TARGET_ERP.');
}

const targetUrl = new URL(parameters.url);
const headers = new Headers({
  Authorization: `Basic ${Buffer.from(`${parameters.user}:${parameters.password}`).toString('base64')}`,
});
```

O trecho produz a URL e os cabeçalhos a serem passados ao cliente específico do destino; não cria uma fábrica universal de conexões. A validação deve refletir o protocolo contratado. Não implemente todos os tipos de autenticação apenas porque o SDK os representa.

Faça a preparação antes da primeira consulta que depende dela. `preProcessingCustomerRoutines()` ocorre **depois** da carga da origem e não serve para inicializar o cliente necessário a essa carga. Consulte [integration-flows.md](integration-flows.md).

## HTTP e autenticação

| `authType` | Campos definidos no SDK | Uso |
|---|---|---|
| `NONE` | `url` | Sem autenticação configurada no sistema. |
| `BASIC` | `url`, `user?`, `password?` | Valide os campos exigidos e construa o cabeçalho BASIC no cliente da API. |
| `QUERY_STRING` | `url`, `queryString?` | Interprete a string no formato contratado; use `URL`/`URLSearchParams` para preservar parâmetros e codificação. |
| `HEADERS` | `url`, `authHeader?` | A propriedade é uma string. Seu nome não determina o nome ou o formato do cabeçalho HTTP. |

O SDK não interpreta `authHeader`. O cadastro consultado também armazena esse campo como texto. Portanto, não há contrato universal que permita atribuir o conteúdo inteiro a `Authorization` ou executar `JSON.parse` automaticamente.

Defina o formato que a integração espera e rejeite entradas incompatíveis. Exemplos de contratos distintos são um valor para `Authorization`, um par `X-Api-Key=valor` ou um objeto JSON de cabeçalhos. Implemente somente o formato acordado, sem uma cadeia de tentativas para adivinhar os demais. Quando o contrato for JSON, valide que o resultado é um objeto de valores string antes de usá-lo como cabeçalhos.

Para `QUERY_STRING`, não concatene `?` indiscriminadamente a uma URL que pode já ter parâmetros. Para credenciais em URL ou cabeçalhos, confira o tratamento de registros em [http-interceptor.md](http-interceptor.md).

O tipo `HTTP` descreve a conexão; ele não implementa OAuth, renovação de tokens, paginação, retentativas ou validação de sucesso da API. Implemente esses comportamentos somente quando exigidos pelo destino e em seu cliente específico. `fetch` não rejeita por HTTP 4xx/5xx: valide a resposta e o resultado de negócio antes de confirmar um item.

## Parâmetros customizados do sistema

Cada sistema pode conter `parameters.custom` com pares `{name, value}`. Use `AutomationParameter` para lê-los, passando os parâmetros **do sistema**, quando a configuração pertencer à conexão.

Trecho após selecionar e validar `sourceSystem`:

```typescript
const companyCode = AutomationParameter.getRequiredParameter(sourceSystem.parameters, 'company_code');
if (!companyCode.trim()) {
  throw new Error('O código da empresa no sistema de origem não pode estar vazio.');
}
```

Importe `AutomationParameter` de `@tunnelhub/sdk`. Se a regra pertence apenas à automação, leia `this.parameters` em vez de `sourceSystem.parameters`. Veja as diferenças entre ausência, string vazia e conversão em [parameters-management.md](parameters-management.md).

Não recrie leitores genéricos de parâmetros com nomes alternativos, valores padrão ocultos ou múltiplas fontes. Preserve os nomes definidos para a integração.

## Demais tipos de sistema

Use os tipos do SDK para acessar os campos da conexão escolhida; não copie todos esses contratos para tipos locais.

| Tipo | Campos e discriminadores relevantes |
|---|---|
| `SOAP` | `wsdlUrl`; `authType: NONE \| BASIC \| HEADERS`, com `user`/`password` ou `authHeader` conforme o caso. |
| `DATABASE` | `databaseType`; MYSQL, MSSQL, REDSHIFT e POSTGRESQL usam campos como `host`, `port`, `user`, `password`, `database`; ORACLE acrescenta `oracleConnectionType`, `serviceName`/`sid`; MONGODB e JDBC possuem `uri`/`jdbcUrl`. |
| `FTP` | `host`, `port`, `user`, `password`, `encryption`, `timeout`, `maxReconnectionAttempts`, `reconnectDelay`, `automaticallyDisconnect`. |
| `SFTP` | Conexão semelhante a FTP, com `authType: PASSWORD \| SSH_KEY`; use `password` ou `sshPrivateKey`. |
| `LDAP` | `host`, `user`, `password`, `baseDn`. |
| `MAIL` | `host`, `user`, `password`, `timeout`. |
| `SAPRFC` | `host`, `user`, `password`, `systemNumber`, `mandt`, `trace`. |
| `SMB` | `host`, `user`, `password`, `homeDirectory`. |

A presença de uma configuração não cria automaticamente um cliente ou gerencia sua conexão. Use a biblioteca necessária ao protocolo, respeite seu contrato e libere recursos em `finally` quando a operação os abrir. Não transforme uma falha de conexão em lista vazia: em Delta isso pode produzir exclusões indevidas.

Em SOAP, autenticação HTTP do transporte e credenciais dentro do envelope são mecanismos distintos. Não substitua um pelo outro sem base no contrato do serviço.

## Consulta dinâmica de um sistema

Use `new System(event).getSystemByInternalName(internalName)` apenas quando o caso realmente exige buscar uma conexão que não veio no evento. O helper consulta o tenant e ambiente do evento e retorna `Promise<TunnelHubSystem | null>`. Valide o resultado como faria com `this.systems`.

```typescript
import {System} from '@tunnelhub/sdk';
import type {ProcessorPayload, TunnelHubSystem} from '@tunnelhub/sdk';

export async function loadAdditionalSystem(event: ProcessorPayload): Promise<TunnelHubSystem> {
  const system = await new System(event).getSystemByInternalName('ADDITIONAL_SYSTEM');
  if (!system) {
    throw new Error('O sistema ADDITIONAL_SYSTEM não foi encontrado.');
  }
  return system;
}
```

Na versão consultada, o helper faz uma consulta DynamoDB e pode chamar KMS para campos reconhecidos com prefixo `THENC:`. Ele não verifica `SDK.testMode`; simule essa fronteira em testes. A consulta usa uma única página, sem percorrer `LastEvaluatedKey`, portanto a ausência retornada não deve ser tratada como prova universal de que nenhum cadastro existe em ambientes com muitos sistemas.

Não implemente descriptografia própria na automação nem registre o conteúdo das credenciais. Se a conexão recebida não estiver utilizável, investigue o contrato do runtime e da versão do SDK. Não faça varredura da tabela de produção para descobrir sistemas.
