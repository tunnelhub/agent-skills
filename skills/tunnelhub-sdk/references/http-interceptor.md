# Interceptor de requisições HTTP

`setupInterceptor` instala registros de requisições e respostas para `fetch`. Use quando esse diagnóstico for necessário e compatível com os dados trafegados. Ele não é um cliente HTTP, não implementa autenticação ou retentativas e não substitui os resultados por item do fluxo.

Contratos conferidos em `src/util/interceptor/{interceptorSetup,fetchInterceptor,logger}.ts` do SDK 4.1.5.

## Configuração

```typescript
import {setupInterceptor} from '@tunnelhub/sdk';

setupInterceptor({
  enableGlobalFetch: true,
  enableUndiciFetch: false,
  ignoreUrls: [
    'https://auth.example.invalid/token',
    /^https:\/\/files\.example\.invalid\/documents(?:\/|\?|$)/,
  ],
});
```

Os endereços são ilustrativos. Defina exclusões a partir dos endpoints reais e do conteúdo que eles recebem ou devolvem, sem colocar valores de tokens nas regras. Faça a configuração uma vez na inicialização do módulo de entrada, antes das chamadas que pretende interceptar.

| Opção | Comportamento |
|---|---|
| `enableGlobalFetch` | Padrão `true`; substitui `globalThis.fetch`. |
| `enableUndiciFetch` | Padrão `false`; modifica o `fetch` da instância de `undici` resolvida pelo SDK. |
| `ignoreUrls` | Array de strings ou expressões regulares; acrescentado às exclusões padrão na primeira configuração. |

`setupInterceptor()` mantém uma instância única no módulo. Chamadas posteriores retornam essa mesma instância; não atualizam as opções nem adicionam novas exclusões. A assinatura pode incluir `null`, mas isso não significa que uma segunda chamada retorne `null` por já estar configurada.

O escopo é o processo/módulo, não uma única execução da automação. Não faça reconfiguração por item ou por tenant.

### Exclusões

O SDK exclui por padrão `https://api.ipify.org` e, quando definida, a URL em `ECS_CONTAINER_METADATA_URI`.

Uma exclusão em string corresponde a:

- URL exatamente igual à string;
- URL que começa com a string seguida de `/`;
- URL que começa com a string seguida de `?`.

Não é uma comparação por substring livre. Por exemplo, excluir `https://api.example.invalid/token` não exclui `/tokens`. Considere também barra final, query e outros caminhos ao definir a regra.

Expressões regulares usam `.test(url)`. Prefira padrões ancorados e sem flags `g` ou `y`, para evitar que o estado de `lastIndex` altere correspondências sucessivas.

A URL ignorada é enviada diretamente pelo fetch original, sem os interceptores normais de requisição e resposta. Exceções síncronas nesse caminho ainda podem entrar no tratamento de erros e registrar mensagem/stack; exclusão não é uma política geral de remoção de informações sensíveis dos logs.

## O que é registrado

| Momento | Dados produzidos pelo logger atual |
|---|---|
| Requisição | Data/hora, método, URL, cabeçalhos presentes em `config.headers` e corpo formatado. |
| Resposta | Data/hora, código e texto do status, cabeçalhos e corpo lido de um clone da resposta. |
| Erro | Data/hora, mensagem e stack. |

A representação dos cabeçalhos de requisição depende do valor recebido (`Headers`, objeto etc.). Não assuma que o registro reconstrói fielmente todos os detalhes do transporte. Não há medição de duração implementada nesse logger.

### Não há mascaramento automático

Na implementação consultada, o logger não remove `Authorization`, cookies, chaves de API, query strings, documentos ou outros campos sensíveis. Ele escreve diretamente no console e não consulta `SDK.verbose` nem `SDK.testMode`.

Por isso:

- Não habilite o interceptor supondo que credenciais ou corpos serão mascarados.
- Exclua endpoints cujo conteúdo não possa ser registrado; quando todo o cliente trafega dados inadequados para logs, mantenha a interceptação desse cliente desabilitada e registre somente contexto mínimo permitido.
- Não use um interceptor adicionado depois de `setupInterceptor()` como suposta proteção: os callbacks do logger são registrados primeiro, antes dos callbacks que você acrescentar.
- Uma proteção externa eventualmente existente na plataforma não comprova que o dado deixou de ser emitido pelo SDK.

Não modifique cabeçalhos ou corpos reais para esconder segredos no registro: isso pode quebrar a chamada. Uma solução de redaction deve atuar sobre a representação do log e precisa ser implementada/validada no componente que o produz, quando fizer parte da tarefa.

## Limites de cobertura e efeitos

O interceptor cobre os caminhos de `fetch` efetivamente substituídos. Ele não cobre automaticamente Axios, clientes SOAP, drivers de banco, SDKs de terceiros ou referências a funções capturadas antes da instalação.

Se o projeto usa `undici.fetch`, ativar `enableUndiciFetch` afeta a instância resolvida pelo SDK. Dependências duplicadas, referências anteriores ou o empacotamento podem impedir que outra cópia seja interceptada. Não fixe uma versão antiga só para seguir um exemplo; confira a resolução das dependências do projeto. Se o código importa `undici` diretamente, mantenha a dependência direta apropriada em vez de depender de uma instalação transitiva.

O logger lê o corpo completo da resposta clonada. Isso pode aumentar consumo de memória e alterar a latência percebida, especialmente com arquivos grandes ou respostas em stream. O caminho de formatação de corpos de requisição também manipula streams. Não presuma transparência para uploads, streaming ou combinações complexas de `Request` e opções de `fetch`; exclua esses caminhos ou valide o comportamento exigido antes de interceptá-los.

Erros HTTP 4xx/5xx normalmente produzem uma `Response`, não uma rejeição de `fetch`. O interceptor registra a resposta e não a transforma automaticamente em falha de negócio. O cliente deve verificar `response.ok` e o contrato de resposta da API; as ações do fluxo devem propagar falhas conforme [integration-flows.md](integration-flows.md).

## Métodos da instância

A instância retornada expõe `useRequest`, `useResponse` e `useError`; cada um registra um callback e devolve uma função para removê-lo. Use somente se a integração tiver uma necessidade real além do logger existente. Não crie uma cadeia genérica de middleware para chamadas simples.

| Método | Contrato do callback |
|---|---|
| `useRequest` | Recebe URL e configuração; retorna `{url, config}` de forma síncrona ou assíncrona. |
| `useResponse` | Recebe a resposta; retorna a resposta de forma síncrona ou assíncrona. |
| `useError` | Recebe um erro; pode retornar outro erro. O caminho final continua rejeitando, não retorna uma resposta de sucesso. |

`disable()` restaura as funções originais capturadas na construção. Como o singleton mantém seu estado, chamar `setupInterceptor()` novamente depois disso não o reativa automaticamente. `enable(options)` existe na instância, mas é uma operação diferente de `setupInterceptor`: usa as exclusões fornecidas diretamente, sem acrescentar os defaults do wrapper. Evite esse ciclo no código normal de uma automação.

## Verificação local

Quando a tarefa alterar a interceptação, use chamadas simuladas e confirme o caminho realmente utilizado pelo cliente. Cubra as exclusões relevantes, propagação de falhas e preservação da resposta. Não envie credenciais reais para testar os registros.

Os efeitos globais precisam ser isolados e restaurados pelos testes; `SDK.testMode = true` não desabilita esse interceptor. Veja [testing-patterns.md](testing-patterns.md). Para acompanhar sucesso/falha por item, use os retornos do fluxo; para persistência desses registros, veja [logging-strategy.md](logging-strategy.md).
