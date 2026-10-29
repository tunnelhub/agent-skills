# Padrões de implementação

Aplique estes padrões diretamente. Eles definem a base de uma automação TunnelHub. O contrato
do cliente define campos, regras e operações dentro dessa base. Em manutenção, preserve
estruturas equivalentes e limite a mudança ao pedido.

## Uma classe de fluxo, serviços por sistema e conversões explícitas

A classe `Integration` estende uma das quatro classes de fluxo do SDK. Ela coordena as cargas,
normaliza os registros e implementa as ações. O handler instancia a classe uma vez por execução
e chama `AutomationExecution.executeAutomation`; não contém regras de mapeamento ou loops de envio.

Um serviço representa o acesso a um sistema externo. Nele ficam autenticação, transporte,
paginação, timeout, tratamento de resposta e operações com nomes do domínio daquele sistema.
Reutilize a conexão ou o cliente quando o protocolo permitir, respeitando seu ciclo de vida.
O serviço não calcula delta, não grava logs de itens do fluxo e não conhece tabelas internas do TunnelHub.

As conversões traduzem formatos e conceitos de negócio: datas, códigos, enumerações e composição
de identificadores. Use funções ou uma classe simples conforme a organização do projeto. Uma
função local curta é suficiente quando usada em um único ponto; não crie um módulo para cada campo.
Não construa um mecanismo genérico de mapeamento, registro de estratégias ou interpretador de JSON
quando atribuições explícitas e funções de conversão resolvem o contrato.

O payload completo de envio pode ser montado em um método da integração ou em um mapper específico.
A existência de um mapper não é o problema: sua saída e o ponto em que é chamado devem preservar
a separação entre o modelo comparado e o contrato externo.

## Três representações distintas

```text
Resposta da origem → IntegrationModel → Payload do destino
Resposta do destino → IntegrationModel
                           ↓
                  Delta e monitoramento
```

1. **Tipo de origem:** descreve os campos realmente recebidos da API, consulta ou arquivo.
2. **IntegrationModel:** descreve o registro normalizado que a automação integra. Tem nomes de
   negócio legíveis, identidade explícita e valores consistentes entre as duas cargas.
3. **Tipo de destino:** descreve o corpo da operação HTTP, SOAP ou outra gravação. É construído
   na etapa de envio. Autenticação pertence à conexão, não ao registro integrado.

Use campos planos para os valores comparados pelo SDK. Um objeto aninhado pode existir como
informação auxiliar quando houver necessidade, mas não deve substituir todos os campos delta
por um envelope de requisição. A comparação padrão do SDK conferido concatena valores; ela não
faz comparação profunda de objetos. Consulte [integration-flows.md](integration-flows.md) para
identidade composta e limites dessa comparação.

Declare somente campos do escopo. Não copie um catálogo inteiro de outro cliente, nem carregue
salário, dados bancários ou credenciais porque uma API os disponibiliza. Datas e opcionais devem
ter uma representação interna consistente; converta para o formato do destino ao montar o envio.

### Exemplo de normalização e montagem

Este exemplo completo demonstra as representações de um cadastro simples. Os nomes dos campos
externos são ilustrativos; substitua-os pelo contrato documentado da automação.

```typescript
import type {Metadata} from '@tunnelhub/sdk';

type SourceRecord = {code: string; description: string; enabled: boolean};
type TargetRecord = {
  id: string;
  externalCode: string;
  name: string;
  state: 'ACTIVE' | 'INACTIVE';
};
type TargetInput = Omit<TargetRecord, 'id'>;

export type IntegrationModel = {
  externalCode: string;
  name: string;
  active: boolean;
  targetId?: string;
};

export const keyFields: (keyof IntegrationModel)[] = ['externalCode'];
export const deltaFields: (keyof IntegrationModel)[] = ['name', 'active'];

export const metadata = [
  {fieldName: 'externalCode', fieldLabel: 'Código de origem', fieldType: 'TEXT'},
  {fieldName: 'name', fieldLabel: 'Nome', fieldType: 'TEXT'},
  {fieldName: 'active', fieldLabel: 'Ativo', fieldType: 'BOOLEAN'},
  {fieldName: 'targetId', fieldLabel: 'Código no destino', fieldType: 'TEXT'},
] satisfies Metadata<IntegrationModel>[];

export function fromSource(record: SourceRecord): IntegrationModel {
  return {
    externalCode: record.code,
    name: record.description,
    active: record.enabled,
  };
}

export function fromTarget(record: TargetRecord): IntegrationModel {
  return {
    externalCode: record.externalCode,
    name: record.name,
    active: record.state === 'ACTIVE',
    targetId: record.id,
  };
}

export function toTarget(item: IntegrationModel): TargetInput {
  return {
    externalCode: item.externalCode,
    name: item.name,
    state: item.active ? 'ACTIVE' : 'INACTIVE',
  };
}
```

Os clientes validam os retornos externos antes dessas funções. A identidade `externalCode` precisa
ser única no universo da automação. `targetId` serve para endereçar operações e não dispara atualização.
Se a criação gerar esse ID, use `fillGeneratedId`; preserve-o em atualização com
`fillExternalIdAfterUpdate`, conforme o contrato do fluxo. Não presuma que retornar o ID em `data`
o incorpora automaticamente à imagem delta.

Valide as colunas com `satisfies Metadata<IntegrationModel>[]` e devolva esse array em
`defineMetadata(): Metadata[]`. No SDK 4.1.5, anotar a variável diretamente como
`Metadata<IntegrationModel>[]` pode incluir `undefined` no tipo de `fieldName` quando o modelo tem
propriedades opcionais, tornando-a incompatível com o retorno da classe base. `satisfies` valida
os nomes sem ampliar os valores efetivamente declarados; não use um cast para ocultar esse erro.

## Cargas, ações e operações de negócio

- `loadSourceSystemData`: obter todo o conjunto definido para a execução, completar a paginação,
  normalizar e devolver `IntegrationModel[]`. Não enviar ao destino dentro dessa carga.
- `loadTargetSystemData`: quando o fluxo delta exigir carga inicial, consultar o destino e
  normalizar para o mesmo modelo. Não devolver `[]` apenas para dispensar a implementação.
- `insertAction` / `sendData`: montar o payload, chamar o serviço e retornar o resultado confirmado.
- `updateAction`: considerar estado anterior e novo. Identificadores e operações intermediárias
  exigidas pelo destino precisam ser respeitados antes de atualizar o cadastro.
- `deleteAction`: executar a semântica contratada de exclusão, inativação ou ausência de ação.
  Ausência na extração não comprova, por si só, uma exclusão de negócio.

Uma atualização que exige transferência, cancelamento ou outra operação específica não pode ser
reduzida silenciosamente a um upsert. Ao mesmo tempo, não acrescente essas operações a integrações
que não as contrataram. Método obrigatório implementado com sucesso fictício não completa o escopo.

No fluxo delta, a carga do destino é usada quando não há imagem persistida não vazia; não suponha
que o destino será relido em todas as execuções. Não altere o motor para compensar um modelo inadequado.

## De/para e parâmetros

Use recursos distintos conforme a informação:

- Correspondências mantidas pela operação, como códigos de empresas entre sistemas: tabela de
  conversão da plataforma, acessada com `ConversionTable` e `DataStore`.
- Enumeração fixa e documentada de uma API: função de conversão explícita. Por exemplo, `ACTIVE`
  para `true`; não exija um de/para configurável para uma equivalência já definida pelo contrato.
- Opção de negócio configurável, como incluir inativos: parâmetro da automação.
- URL, credenciais e configuração de conexão: sistema vinculado.
- Dados estruturados que exigem colunas, filtros e manutenção tabular: `SqlTables`, se necessário.

Não crie um único parâmetro JSON reunindo todos os mapas, regras e credenciais. Também não crie uma
tabela para toda constante. Quando um de/para obrigatório estiver ausente, sinalize a falha com
contexto operacional e não escolha arbitrariamente um código. Valores desconhecidos de enumeração
seguem o contrato: erro ou omissão explicitamente permitida, sem defaults inventados.

## Código entregue

- Mantenha o estilo do projeto, nomes claros e tipos de domínio. Evite `any` e coerções usadas
  apenas para esconder uma incompatibilidade. Não replique os tipos do SDK na automação.
- Reaproveite os helpers do SDK. Não reimplemente parâmetros, de/para, sequências, filas, delta,
  logs ou estatísticas com acesso direto à infraestrutura.
- Use mensagens em português quando esse for o idioma do projeto. Comente decisões não óbvias,
  sem gerar explicações linha a linha ou documentação repetida em vários arquivos.
- Implemente cada operação prevista no contrato e mantenha visíveis as pendências reais. Não
  chame uma entrega de homologada por ter compilado, passado em mocks ou sido publicada.
- Valide transformações e falhas sem mockar a regra em teste. Revise modelo, campos delta,
  metadata e payload juntos quando um campo integrado mudar.

Não amplie a tarefa para modernizar todo o código existente. Um desvio necessário deste padrão
precisa ter motivo técnico ou de negócio explícito; preferência pessoal do agente não basta.
