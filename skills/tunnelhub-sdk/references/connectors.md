# Descobrir e usar conectores TunnelHub

Procure primeiro nas dependências do projeto. Para descobrir outros candidatos, pesquise no registro público do npm e filtre os resultados pelo prefixo do escopo oficial:

```sh
npm search --registry=https://registry.npmjs.org --searchlimit=100 --json '@tunnelhub/connector-' |
  node -e "const fs = require('node:fs'); for (const p of JSON.parse(fs.readFileSync(0, 'utf8'))) if (p.name.startsWith('@tunnelhub/connector-')) console.log(p.name)"
```

A busca retorna candidatos por relevância. Se souber o nome exato, consulte-o diretamente com `npm view`. O pacote `@tunnelhub/connector-core` reúne componentes compartilhados. Para integrar um fornecedor, escolha o pacote cuja documentação cubra o sistema e a operação.

1. Identifique a versão resolvida no lockfile ou, em um projeto novo, consulte `npm view <pacote> version engines` e confira a compatibilidade com o runtime da automação.
2. Consulte `npm view <pacote>@<versão> exports dependencies`. Confira no README, nas declarações de tipos e no manifest distribuídos nessa versão o método, a configuração, o retorno e o nível de validação da operação. `homologated` indica prova real no cenário documentado. Uma operação `exposed` ainda precisa de validação para o cenário de uso.
3. Carregue as credenciais pelo [sistema configurado no TunnelHub](system-configuration.md) e passe ao conector os campos que ele documenta. Verifique no pacote as orientações para a [primeira configuração do interceptor](http-interceptor.md) e os recursos de teste disponíveis.

Se a operação necessária não estiver coberta, use o cliente existente do projeto ou implemente apenas a chamada que falta.
