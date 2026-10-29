# Publicação de automações TunnelHub com CI/CD

Fonte: https://docs.tunnelhub.io/cli/ci-cd-deployments
Última captura: 2026-06-30

Use esta referência ao configurar a publicação automática de uma integração criada com `@tunnelhub/sdk`. O resultado esperado é um workflow do GitHub Actions em `.github/workflows/deploy-tunnelhub.yml`. Execute `tunnelhub deploy-automation` somente dentro do workflow, sem executá-lo no ambiente local do agente.

## Objetivo

Gere ou atualize `.github/workflows/deploy-tunnelhub.yml` para que o GitHub Actions possa publicar uma automação TunnelHub com `@tunnelhub/cli`.

O fluxo deve usar credenciais CI/CD da conta com uma lista de ambientes permitidos. Essas credenciais pertencem à conta, mas só permitem publicar nos UUIDs de ambientes TunnelHub autorizados na criação da credencial.

## Pré-requisitos

Antes de gerar o workflow, confirme ou documente que o projeto possui:

- Uma automação já criada no TunnelHub.
- Um `tunnelhub.yml` configurado para a automação que será publicada.
- O caminho `package.artifact` em `tunnelhub.yml`; o workflow deve gerar o ZIP nesse caminho antes da publicação.
- O UUID de cada ambiente TunnelHub no qual o workflow pode publicar.
- Acesso a `Administration > Settings > CI/CD` no TunnelHub para criar credenciais CI/CD.
- Permissões no repositório GitHub para criar ambientes, segredos e variáveis.

## Criar a credencial CI/CD

No portal TunnelHub:

1. Abra `Administration > Settings`.
2. Abra a aba `CI/CD`.
3. Clique em `Criar credencial CI/CD`.
4. Informe um nome para a credencial.
5. Selecione os ambientes TunnelHub autorizados.
6. Conclua a criação.
7. Copie o `Client ID` e o `Client secret`.

Regras importantes:

- Não use `tunnelhub login` no CI.
- Não inclua credenciais no controle de versão.
- Armazene `Client ID` e `Client secret` como segredos do provedor de CI.
- Prefira credenciais com o menor privilégio necessário, idealmente uma por ambiente em fluxos sensíveis de produção.
- O `Client secret` é exibido apenas uma vez.
- Se um segredo for perdido ou exposto, regenere ou revogue a credencial em `Administration > Settings > CI/CD`.

## Segredos e variáveis do GitHub

Configuração recomendada no GitHub Actions:

- `TH_CLIENT_ID`: segredo do ambiente no GitHub.
- `TH_CLIENT_SECRET`: segredo do ambiente no GitHub.
- `TH_ENVIRONMENT_ID`: variável do ambiente no GitHub contendo o UUID do ambiente TunnelHub.
- `TH_API_HOST`: variável do repositório, variável do ambiente no GitHub ou valor fixo no workflow. Use `https://api.tunnelhub.io` para produção.

O ambiente do GitHub e o ambiente TunnelHub são conceitos diferentes:

- O ambiente do GitHub controla aprovações, regras de proteção, segredos e variáveis no GitHub.
- O ambiente TunnelHub é identificado pelo UUID passado em `--environment-id` durante a publicação.
- A variável `TH_ENVIRONMENT_ID` estabelece a ligação entre eles.

Exemplo de correspondência:

- O ambiente `production` do GitHub tem `TH_ENVIRONMENT_ID=<uuid-do-ambiente-de-producao>`.
- O ambiente `development` do GitHub tem `TH_ENVIRONMENT_ID=<uuid-do-ambiente-de-desenvolvimento>`.

## Regras de geração do workflow

Ao configurar publicação automática, gere ou atualize `.github/workflows/deploy-tunnelhub.yml`.

Alinhe a versão do Node.js no workflow com `engines`, a versão instalada do SDK e o runtime de
`tunnelhub.yml`. Os exemplos abaixo usam Node.js 24, compatível com o SDK 4.1.5 e os templates
atuais. Não copie Node.js 22 de workflows antigos para uma automação que exige Node.js 24.

Comportamento obrigatório:

- Use `workflow_dispatch` para permitir publicação manual.
- Use gatilhos por branch somente quando solicitados pelo usuário ou quando a correspondência de branches estiver clara no projeto.
- Instale as dependências com o gerenciador existente, identificando `pnpm-lock.yaml`, `yarn.lock` ou usando `npm ci` na ausência deles.
- Gere o ZIP no caminho configurado em `package.artifact` de `tunnelhub.yml` antes da publicação.
- Instale `@tunnelhub/cli` no workflow.
- Execute `tunnelhub deploy-automation` somente dentro do workflow.
- Passe `--environment-id "$TH_ENVIRONMENT_ID"`.
- Passe uma mensagem significativa em `--message`; esse argumento é obrigatório.

Contexto recomendado para a mensagem de publicação:

- Mensagem do commit.
- SHA abreviado do commit.
- E-mail do autor do commit ou usuário responsável pela execução no GitHub.
- Nome da branch.
- Execução do workflow ou contexto do PR/release, quando útil.

A revisão criada armazena:

- `createdBy` como `api-client:<clientId>`.
- `message` com o contexto humano e operacional fornecido em `--message`.

## Cuidados quando o workflow executa testes

Testes não são obrigatórios para gerar um workflow de publicação TunnelHub. Adicione uma etapa de testes somente quando o projeto já adotar essa convenção, o workflow já executar testes ou o usuário solicitar explicitamente.

Se o workflow executar testes, examine `package.json` e a estrutura de testes antes de manter ou adicionar essa etapa. Evite executar no CI testes de depuração, sem mocks ou com dados reais.

Verifique a existência de:

- Caminhos ou nomes de testes contendo `unmocked`.
- Testes criados para depuração manual.
- Testes que usam credenciais ou identificadores reais de ambientes.
- Testes que fazem chamadas reais HTTP, a bancos ou a serviços sem mocks.

Em projetos Jest que mantêm testes com chamadas reais em um caminho `unmocked`, prefira scripts como:

```json
"scripts": {
  "test": "jest --testPathIgnorePatterns=unmocked",
  "test:coverage": "jest --collect-coverage --silent --testPathIgnorePatterns=unmocked",
  "test:badges": "jest --coverage --silent --testPathIgnorePatterns=unmocked && make-coverage-badge"
}
```

Não reescreva configurações de teste não relacionadas apenas porque existe um caminho `unmocked`. Ajuste ou sinalize o problema somente quando esses testes seriam executados pelo workflow de publicação ou quando o usuário pedir para tornar os scripts de teste seguros.

Se não estiver claro se um teste faz chamadas reais, peça confirmação ou exclua os testes suspeitos do comando de CI, sem presumir silenciosamente que são seguros.

## Modelo de GitHub Actions: main para produção

Use este modelo quando `main` deve publicar no ambiente `production` do GitHub.

```yaml
name: Publicar automação TunnelHub

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy-production:
    if: github.ref_name == 'main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Obter código do repositório
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24

      - name: Habilitar Corepack
        run: corepack enable

      - name: Instalar dependências
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm install --frozen-lockfile
          elif [ -f yarn.lock ]; then
            yarn install --immutable
          else
            npm ci
          fi

      - name: Gerar artefato de publicação
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm build
          elif [ -f yarn.lock ]; then
            yarn build
          else
            npm run build
          fi

      - name: Instalar CLI do TunnelHub
        run: npm install --global @tunnelhub/cli

      - name: Publicar automação
        env:
          TH_API_HOST: https://api.tunnelhub.io
          TH_CLIENT_ID: ${{ secrets.TH_CLIENT_ID }}
          TH_CLIENT_SECRET: ${{ secrets.TH_CLIENT_SECRET }}
          TH_ENVIRONMENT_ID: ${{ vars.TH_ENVIRONMENT_ID }}
        run: |
          COMMIT_EMAIL="$(git log -1 --pretty=format:'%ae')"
          COMMIT_MESSAGE="$(git log -1 --pretty=format:'%B')"
          COMMIT_SHA="$(git rev-parse --short HEAD)"
          DEPLOY_MESSAGE="${COMMIT_MESSAGE}

          Publicação ${COMMIT_SHA} por ${COMMIT_EMAIL}"

          tunnelhub deploy-automation \
            --environment-id "$TH_ENVIRONMENT_ID" \
            --message "$DEPLOY_MESSAGE"
```

## Modelo de GitHub Actions: develop e main

Use este modelo quando `develop` deve publicar em `development` e `main` deve publicar em `production`.

```yaml
name: Publicar automação TunnelHub

on:
  push:
    branches:
      - develop
      - main
  workflow_dispatch:

jobs:
  deploy-development:
    if: github.ref_name == 'develop'
    runs-on: ubuntu-latest
    environment: development

    steps:
      - name: Obter código do repositório
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24

      - name: Habilitar Corepack
        run: corepack enable

      - name: Instalar dependências
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm install --frozen-lockfile
          elif [ -f yarn.lock ]; then
            yarn install --immutable
          else
            npm ci
          fi

      - name: Gerar artefato de publicação
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm build
          elif [ -f yarn.lock ]; then
            yarn build
          else
            npm run build
          fi

      - name: Instalar CLI do TunnelHub
        run: npm install --global @tunnelhub/cli

      - name: Publicar automação
        env:
          TH_API_HOST: https://api.tunnelhub.io
          TH_CLIENT_ID: ${{ secrets.TH_CLIENT_ID }}
          TH_CLIENT_SECRET: ${{ secrets.TH_CLIENT_SECRET }}
          TH_ENVIRONMENT_ID: ${{ vars.TH_ENVIRONMENT_ID }}
        run: |
          COMMIT_EMAIL="$(git log -1 --pretty=format:'%ae')"
          COMMIT_MESSAGE="$(git log -1 --pretty=format:'%B')"
          COMMIT_SHA="$(git rev-parse --short HEAD)"
          DEPLOY_MESSAGE="${COMMIT_MESSAGE}

          Publicação ${COMMIT_SHA} por ${COMMIT_EMAIL}"

          tunnelhub deploy-automation \
            --environment-id "$TH_ENVIRONMENT_ID" \
            --message "$DEPLOY_MESSAGE"

  deploy-production:
    if: github.ref_name == 'main'
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Obter código do repositório
        uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 24

      - name: Habilitar Corepack
        run: corepack enable

      - name: Instalar dependências
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm install --frozen-lockfile
          elif [ -f yarn.lock ]; then
            yarn install --immutable
          else
            npm ci
          fi

      - name: Gerar artefato de publicação
        run: |
          if [ -f pnpm-lock.yaml ]; then
            pnpm build
          elif [ -f yarn.lock ]; then
            yarn build
          else
            npm run build
          fi

      - name: Instalar CLI do TunnelHub
        run: npm install --global @tunnelhub/cli

      - name: Publicar automação
        env:
          TH_API_HOST: https://api.tunnelhub.io
          TH_CLIENT_ID: ${{ secrets.TH_CLIENT_ID }}
          TH_CLIENT_SECRET: ${{ secrets.TH_CLIENT_SECRET }}
          TH_ENVIRONMENT_ID: ${{ vars.TH_ENVIRONMENT_ID }}
        run: |
          COMMIT_EMAIL="$(git log -1 --pretty=format:'%ae')"
          COMMIT_MESSAGE="$(git log -1 --pretty=format:'%B')"
          COMMIT_SHA="$(git rev-parse --short HEAD)"
          DEPLOY_MESSAGE="${COMMIT_MESSAGE}

          Publicação ${COMMIT_SHA} por ${COMMIT_EMAIL}"

          tunnelhub deploy-automation \
            --environment-id "$TH_ENVIRONMENT_ID" \
            --message "$DEPLOY_MESSAGE"
```

## Rotação e revogação

Em `Administration > Settings > CI/CD`:

- Use `Regenerar secret` para gerar um novo segredo para uma credencial existente.
- Use `Revogar` para bloquear novas publicações com essa credencial.

Após regenerar um segredo, atualize imediatamente os segredos do ambiente no GitHub. Ao migrar para credenciais específicas por ambiente, revogue as antigas depois de validar o novo fluxo.
