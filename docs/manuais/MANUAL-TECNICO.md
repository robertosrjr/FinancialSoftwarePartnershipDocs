# Manual Técnico — Subir, usar e testar a aplicação

**Versão**: 1.1 · **Data**: 2026-10-11 · **Produto**: Agente de Bolso
**Para quem**: engenheiros e QA que vão rodar o produto na própria máquina, chamar a API e executar os testes. Os comandos abaixo foram executados e conferidos em 2026-10-03 (Windows com Git Bash, JDK 25).
**Documentos irmãos**: [Jornada Técnica](../jornadas/JORNADA-TECNICA.md) (como funciona por dentro) · [Cenários de teste](../testes/README.md) · [application/README.md](../../application/README.md)

## 1. Em um minuto

```bash
cd application
./mvnw -q install -DskipTests                       # compila e gera o jar (cerca de 1 minuto)
java -jar app/target/app-0.1.0-SNAPSHOT.jar --spring.profiles.active=local
```

A API fica em `http://localhost:8080`. Falta só um **token** para chamá-la (seção 4). Os dados ficam **em memória** e somem a cada reinício.

## 2. Pré-requisitos

| Item | Detalhe |
|---|---|
| **JDK 25** | Nada mais. O Maven Wrapper (`mvnw`) baixa o Maven 3.9.11. Confira com `java -version` |
| **Porta 8080** livre | Para mudar: `--server.port=8081` |
| **Banco e containers** | **Não são necessários** nesta fase (Fase A, persistência em memória) |
| **curl** e **python** | Só para os exemplos deste manual. Qualquer cliente HTTP serve |

No Windows, use `mvnw.cmd` no lugar de `./mvnw` no PowerShell ou no CMD. No PowerShell, o `curl` é um apelido de outro comando: use `curl.exe`.

## 3. Compilar e testar

| Objetivo | Comando |
|---|---|
| Compilar sem testes e gerar o jar | `./mvnw -q install -DskipTests` |
| Testes e todos os gates (cobertura, SpotBugs, ArchUnit, Modulith) | `./mvnw verify` (cerca de 10 a 12 minutos) |
| Só os testes de um módulo | `./mvnw -pl conciliacao -am test` |
| Uma classe de teste | `./mvnw -pl conciliacao -am test -Dtest=DeduplicadorTest -Dsurefire.failIfNoSpecifiedTests=false` |
| Varredura de dependências (OWASP Dependency-Check, base NVD) | `./mvnw -Psca install -DnvdApiKey=<sua chave>` (roda também no CI; ver a seção abaixo) |

O `verify` roda, nesta ordem: regras do enforcer, testes de unidade, de propriedade (jqwik), de arquitetura e de integração, gate de cobertura (mínimo de 80% em domínio e aplicação) e SpotBugs com FindSecBugs. Um teste do módulo `app` imprime `Application run failed` de propósito: ele prova que o perfil `prod` se recusa a subir com adaptadores em memória.

Situação em 2026-10-05: BUILD SUCCESS (`shared-kernel` 58 testes, `plataforma` 163, `fluxocaixa` 222, `conciliacao` 643 (1 ignorado), `app` 2060).

### Varredura de dependências e a National Vulnerability Database

A varredura (SCA) usa o OWASP Dependency-Check, que compara as dependências do projeto com as vulnerabilidades publicadas na [National Vulnerability Database (NVD)](https://nvd.nist.gov/), a base do governo dos Estados Unidos (NIST) que numera as CVEs e dá a cada uma a nota CVSS. O build **falha** se alguma dependência tiver vulnerabilidade com CVSS 7 ou mais.

- **Chave da API**: o download da base exige uma chave gratuita da NVD. Peça em [nvd.nist.gov/developers/request-an-api-key](https://nvd.nist.gov/developers/request-an-api-key), confirme pelo e-mail e guarde a chave que a NVD enviar.
- **No CI** (GitHub Actions): a chave é o secret `NVD_API_KEY` do repositório (Settings, Secrets and variables, Actions); o job `sca` do `ci.yml` o lê.
- **Local**: `./mvnw -Psca -DskipTests install -DnvdApiKey=<sua chave>`. Sem a chave, a análise falha com `NvdApiException: Invalid API Key`. A primeira execução baixa a base inteira e demora; as seguintes usam o cache local.
- **Nunca** coloque a chave em arquivo do repositório, em log ou em conversa; use o secret ou a linha de comando.
- O relatório sai em `application/target/dependency-check-report.*` (e como artefato `dependency-check` no CI).

### CodeQL em repositório privado

O CodeQL roda no GitHub Actions (`.github/workflows/codeql.yml`), mas em repositório privado sem o plano GitHub Code Security não existe o Code scanning para receber o resultado. Por isso a análise roda com `upload: never`: o job escreve a contagem de achados por regra no **resumo do job** e guarda o arquivo SARIF como artefato `codeql-sarif` (baixe e abra com a extensão SARIF Viewer do VS Code). Não há alerta automático; a análise estática que bloqueia o build é o SpotBugs com FindSecBugs do `verify`.

## 4. Obter um token de desenvolvimento

A API exige um token JWT do emissor. Em desenvolvimento, quem emite é um gerador local; as chaves ficam **fora do repositório**, em `~/.agentedebolso/dev-keys` (criadas na primeira execução).

```bash
cd application
TOKEN=$(./mvnw -q -pl app exec:java -Dexec.classpathScope=test \
  -Dexec.mainClass=br.com.robertosrjr.agentedebolso.dev.GeradorDeTokenDeDesenvolvimento \
  -Dexec.args="--tenant=11111111-1111-1111-1111-111111111111 --usuario=ana --papeis=ADMIN --mfa=true" 2>/dev/null | tail -1)
echo $TOKEN
```

| Argumento | Para que serve |
|---|---|
| `--tenant` | A empresa (UUID). Cada UUID diferente é uma empresa isolada |
| `--usuario` | Identificador do usuário |
| `--papeis` | `ADMIN`, `OPERADOR`, `CONSULTOR_BPO` ou `AUDITOR` (separe por vírgula para mais de um) |
| `--mfa` | `true` simula quem confirmou o segundo fator; `false` testa o bloqueio das ações críticas |

O token vale **1 hora**. Se a API responder 401, gere outro.

**Dica para testar papéis**: gere um token para cada papel, por exemplo `--papeis=OPERADOR` e `--papeis=AUDITOR`, e repita a mesma chamada para ver as permissões.

## 5. Chamar a API: um roteiro completo

Defina uma vez:

```bash
H="Authorization: Bearer $TOKEN"
B=http://localhost:8080/api/v1
K() { echo "Idempotency-Key: $(python -c 'import uuid;print(uuid.uuid4())')"; }   # uma chave nova a cada chamada de escrita
```

**Toda escrita exige o cabeçalho `Idempotency-Key`** (`POST`, `PUT` e `PATCH`). Sem ele, a resposta é 400 `idempotency_key_obrigatoria`. A função `K` acima gera uma chave nova (um UUID) por chamada. Detalhes na [seção 5.8](#58-reenvio-seguro-idempotency-key).

### 5.1 Sem token, a API recusa
```bash
curl -s -o /dev/null -w "%{http_code}\n" $B/plataforma/segregacao/modo       # 401
curl -s -H "$H" $B/plataforma/segregacao/modo
# {"modo":"REVISOR_UNICO","origem":"AUTOMATICA", ...}
```

### 5.2 Cadastrar uma conta financeira e a conta bancária (Admin com MFA)
Desde a U2b, toda conta bancária da conciliação nasce ligada a uma conta financeira `BANCARIA` ativa do fluxo de caixa (uma só por conta financeira).
```bash
curl -s -H "$H" -H "$(K)" -H "Content-Type: application/json" -X POST $B/fluxo-caixa/contas-financeiras   -d '{"nome":"Itau PJ","tipo":"BANCARIA","saldoInicial":"0.00","dataBase":"2026-08-01"}'
# {"id":"<CONTA_FINANCEIRA>", ...}
curl -s -H "$H" -H "$(K)" -H "Content-Type: application/json" -X POST $B/conciliacao/contas   -d '{"contaFinanceiraId":"<CONTA_FINANCEIRA>","nome":"Itau PJ","banco":"0341","agencia":"0001","conta":"99999-9","nomeDoBanco":"Itau"}'
# {"id":"...","contaFinanceiraId":"...","nome":"Itau PJ","banco":"341","numeroMascarado":"****9999","saldoConferido":null, ...}
```
O número completo **não** volta nem é guardado. Repetir o cadastro dá 409.

### 5.3 Importar um extrato OFX
Há arquivos de exemplo (sintéticos) em `application/conciliacao/src/test/resources/ofx/sinteticos/`.
```bash
R=$(curl -s -H "$H" -H "$(K)" -F "arquivo=@conciliacao/src/test/resources/ofx/sinteticos/banco-a-1x-folhas-abertas.ofx" \
  $B/conciliacao/importacoes)
ID=$(echo $R | python -c "import sys,json;print(json.load(sys.stdin)['id'])")
echo $ID        # a importação nasce RECEBIDA (HTTP 202) e é processada em segundo plano
```
Para informar a conta explicitamente (quando o arquivo não a identifica): `-F "contaId=<id da conta>"`.

### 5.4 Acompanhar e ver o resumo
```bash
curl -s -H "$H" "$B/conciliacao/importacoes/$ID?aguardar=10" | python -m json.tool
```
`aguardar` espera até 30 segundos por um estado final. Resultado esperado para o arquivo de exemplo: `status` `CONCLUIDA`, 4 transações novas, `situacaoDoSaldo` `SEM_CONFERENCIA` com motivo `PRIMEIRA_IMPORTACAO` e `progresso.pendentes` 4.

**Reimportar o mesmo arquivo** responde **200** (não 202) com uma importação `REPETIDA` e nenhuma transação nova.

### 5.5 Categorias e lançamentos
```bash
curl -s -H "$H" $B/fluxo-caixa/categorias          # 20 categorias, criadas na primeira chamada
CAT=<id de uma categoria de entrada>
curl -s -H "$H" -H "$(K)" -H "Content-Type: application/json" -X POST $B/fluxo-caixa/lancamentos \
  -d "{\"direcao\":\"ENTRADA\",\"valor\":\"2000.00\",\"categoriaId\":\"$CAT\",\"dataPrevista\":\"2026-10-10\",\"descricao\":\"Mensalidade contrato\"}"
# {"id":"...","valor":"2000.00","situacao":"PREVISTO", ...}
```
O **valor é sempre texto** (`"2000.00"`); um número JSON (`2000.00`) responde **400**.

Outras chamadas: `GET /fluxo-caixa/lancamentos` (filtros e paginação), `PATCH /lancamentos/{id}` (exige `versao`), `POST /lancamentos/{id}/realizacao`, `POST /lancamentos/{id}/cancelamento` (Admin com MFA, `motivo` de 10+ caracteres).

### 5.6 Auditoria e segregação
```bash
curl -s -H "$H" $B/plataforma/auditoria/verificacao     # Admin ou Auditor com MFA
# {"integra":true,"registrosVerificados":6, ...}
curl -s -H "$H" "$B/plataforma/segregacao/relatorio?inicio=2026-10-01&fim=2026-10-31"
```

### 5.7 Documentação interativa e saúde
| O que | Endereço |
|---|---|
| **Swagger UI** (só fora de produção; desligado no perfil `prod`) | `http://localhost:8080/swagger-ui.html` (redireciona para `/swagger-ui/index.html`) |
| OpenAPI em JSON | `http://localhost:8080/v3/api-docs` |
| Saúde | `http://localhost:8080/actuator/health` |
| Métricas Prometheus | `http://localhost:8080/actuator/prometheus` (aberto no perfil `local`) |

No Swagger UI, clique em **Authorize**, cole o token (só o texto do token, sem a palavra `Bearer`) e use **Try it out** em qualquer endpoint. O contrato OpenAPI e a tela são públicos, mas as chamadas exigem o token. Para o upload do extrato, o Swagger oferece o seletor de arquivo.

### 5.8 Reenvio seguro: `Idempotency-Key`
Para testar o reenvio, use a **mesma chave** duas vezes:
```bash
CH="Idempotency-Key: minha-chave-de-teste-001"
curl -s -i -H "$H" -H "$CH" -H "Content-Type: application/json" -X POST $B/fluxo-caixa/lancamentos -d "<o mesmo corpo>"
curl -s -i -H "$H" -H "$CH" -H "Content-Type: application/json" -X POST $B/fluxo-caixa/lancamentos -d "<o mesmo corpo>"
```
A segunda resposta é **idêntica à primeira** (mesmo status, corpo e `Location`), traz o cabeçalho `Idempotent-Replayed: true` e **não cria outro lançamento**.

| Situação | Resposta |
|---|---|
| Sem o cabeçalho | 400 `idempotency_key_obrigatoria` |
| Chave fora do formato (8 a 128 caracteres entre letras, dígitos e `. _ : -`) | 400 `idempotency_key_invalida` |
| Mesma chave com **outro corpo, outro endpoint ou outra consulta** | 409 `idempotency_key_reutilizada` |
| Mesma chave enquanto a primeira ainda executa | Espera até 30 s e devolve a mesma resposta; passando disso, 409 `requisicao_em_andamento` com `Retry-After` |
| Resposta que **não** foi de sucesso (400, 403, 422, 5xx...) | A chave fica livre: o reenvio executa de novo |
| Mesma chave de **outro usuário ou outra empresa** | É uma chave nova |
| Chave com mais de 24 horas | É uma chave nova |
| Armazém cheio ou indisponível | 503 `capacidade_excedida` ou `idempotencia_indisponivel` (a escrita **não** acontece) |

Reformatar o JSON (espaços, ordem das chaves) não conta como corpo diferente. No upload do extrato, o nome do arquivo e o separador do `multipart` não contam; o conteúdo conta. Detalhes em [Regras de negócio, §5](../sdd/REGRAS-DE-NEGOCIO.md) e na [Jornada técnica](../jornadas/JORNADA-TECNICA.md).

No Swagger UI, o cabeçalho aparece como parâmetro obrigatório de toda escrita; gere um UUID e cole.

### 5.9 Mapa de endpoints
Rotas do ciclo 2 (U4 a U9), além das da primeira entrega. Toda escrita pede `Idempotency-Key`, inclusive os `POST .../totais`, que só leem (o filtro decide pelo método HTTP). A documentação interativa (seção 5.7) traz os corpos.

| Prefixo | Rotas novas | Papéis |
|---|---|---|
| `/api/v1/fluxo-caixa/centros-de-custo`, `/formas-de-pagamento` | `GET`, `POST`, `PATCH /{id}`, `POST /{id}/inativacao`, `/reativacao` | Escrita: Admin. Leitura: os quatro |
| `/api/v1/fluxo-caixa/contas-financeiras` | `POST /{id}/correcao-de-titularidade` | Admin com MFA |
| `/api/v1/fluxo-caixa/series` | `POST` (com `simular` para a prévia), `POST /{id}/edicoes`, `/extensoes`, `GET` e `POST /{id}/cancelamento`, `GET /{id}`, `/historico` | Escrita: Admin; cancelar com MFA. Leitura: os quatro |
| `/api/v1/fluxo-caixa/lancamentos` | `GET ?q=` (busca), `POST /totais` (`ids` ou `filtro` + `q`) | Os quatro |
| `/api/v1/conciliacao/transacoes` | `GET ?q=&conta=&de=&ate=&situacao=&motivo=`, `POST /totais`, `GET /{id}/versoes`, `POST /{id}/conferencia-de-revisao` | Busca e totais: os quatro |
| `/api/v1/conciliacao/conexoes` | `POST`, `GET /{id}/contas-na-origem`, `POST /{id}/confirmacao`, `/reconexao`, `/desconexao`, `/atualizacoes`, `GET`, `GET /{id}/sincronizacoes` | Admin com MFA; "atualizar agora": Admin e Operador; leitura: os quatro |
| `/api/v1/conciliacao/revisoes-da-origem`, `/periodos-reabertos-pela-origem` | `GET` | Os quatro |
| `/api/v1/conciliacao/importacoes/linhas` | `POST` (entrada por linhas) | Admin e Operador |
| `/api/v1/conciliacao/duplicidades` | `GET`, `GET /{id}`, `POST /{id}/manutencao`, `/descarte` | Decidir: Admin com MFA |
| `/api/v1/importacao/lotes` | `POST` (multipart), `/{id}/mapeamento`, `/previa`, `/linhas`, `/revisoes`, `/confirmacao`, `/cancelamento`, `/arquivo` | Admin e Operador; arquivo: Admin ou Auditor com MFA |
| `/api/v1/importacao/lotes/{id}` | `GET` e `POST /desfazimento`, `POST /desfazimento-e-reimportacao` | Admin com MFA |
| `/api/v1/importacao/lotes-de-lancamentos` | `POST`, `/{id}/mapeamento`, `/revisoes`, `/confirmacao`, `/reprocessamento-de-recusados`, `GET` e `POST /{id}/desfazimento` | Preparar: Admin e Operador; confirmar: Admin; desfazer: Admin com MFA |
| `/api/v1/importacao/layouts` | `GET`, `POST /{id}/arquivamento`, `GET /{id}/correspondencias`, `POST .../desfazimento` | Admin e Operador; leitura: os quatro |

Exemplo de busca e totais:

```bash
curl -s "http://localhost:8080/api/v1/fluxo-caixa/lancamentos?q=aluguel" -H "Authorization: Bearer $TOKEN"
curl -s -X POST http://localhost:8080/api/v1/fluxo-caixa/lancamentos/totais -H "Authorization: Bearer $TOKEN" \
  -H "Idempotency-Key: $(uuidgen)" -H "Content-Type: application/json" -d '{"filtro":{"direcao":"SAIDA"},"q":"aluguel"}'
```

Papéis, MFA e caminhos da primeira entrega estão em [Jornada Técnica: Plataforma](../jornadas/JORNADA-TECNICA.md#5-mapa-de-endpoints-da-plataforma-implementado), [Fluxo de Caixa](../jornadas/JORNADA-TECNICA-FLUXO-DE-CAIXA.md#3-mapa-de-endpoints-implementado) e [Conciliação](../jornadas/JORNADA-TECNICA-CONCILIACAO.md#4-mapa-de-endpoints).

## 6. Configuração

Tudo é sobrescrevível por variável de ambiente ou argumento (`--chave=valor`).

| Variável | Uso | Padrão em `local` |
|---|---|---|
| `AGENTEDEBOLSO_DEV_KEYS` | Diretório das chaves JWT de desenvolvimento | `~/.agentedebolso/dev-keys` |
| `AGENTEDEBOLSO_CHAVE_HMAC` | Chave da cadeia de auditoria (Base64, 32 bytes ou mais) | Gerada a cada execução |
| `AGENTEDEBOLSO_CHAVE_PSEUDONIMIZACAO` | Chave de pseudonimização (IP, tenant em logs e índices cegos), Base64, 32 bytes ou mais | Gerada a cada execução |
| `AGENTEDEBOLSO_JWT_ISSUER_URI` | Emissor de tokens (obrigatório em `prod`) | Não usado |
| `agentedebolso.conciliacao.processamentos-simultaneos` | Importações processadas ao mesmo tempo | 8 |
| `agentedebolso.seguranca.limitador.maximo-de-baldes` | Teto de baldes do limitador de taxa na memória; cheio, chave nova recebe 429 (sem expulsar ninguém). Baldes parados há 10 minutos expiram | 100000 |

**Atenção**: como as chaves de auditoria e de pseudonimização são geradas a cada execução, **reiniciar apaga também os dados** (memória), então não há inconsistência. Se algum dia as chaves forem fixadas sem persistência, os índices de conta passam a depender delas.

| Perfil | Quando | Comportamento |
|---|---|---|
| `local` | Desenvolvimento | Chave pública de desenvolvimento, logs em `DEBUG` com campos estruturados, Prometheus aberto, Swagger ligado. **Nunca processe extrato real neste perfil** |
| `test` | Testes | Usado pelo build |
| `prod` | Produção | **Recusa-se a iniciar** enquanto houver adaptadores em memória (`GuardaDeProducao`). Logs JSON (ECS), Actuator em porta separada, Swagger desligado |

Limites do upload: 10 MB por arquivo e 5.000 transações; 20 importações por minuto por empresa. Planilhas: 10 MB, 5.000 linhas, 50 colunas, 1.000 caracteres por célula.

Limites da busca (U9), por minuto: 60 buscas com texto por usuário e 300 por empresa; 10 buscas por CPF ou CNPJ por usuário; 60 totalizações por usuário. Além disso, cada rota tem o teto por empresa do `@LimitarTaxa`.

## 7. Problemas comuns

| Sintoma | Causa provável | O que fazer |
|---|---|---|
| `release version 25 not supported` ou erro de versão | JDK abaixo de 25 | Instale o JDK 25 e confira `java -version` e `JAVA_HOME` |
| `Port 8080 was already in use` | Outro processo na porta | Suba com `--server.port=8081`, ou encerre o outro processo |
| **401** em toda chamada | Token ausente, expirado (1 hora) ou assinado por outra chave | Gere outro token; confira se a chave em `AGENTEDEBOLSO_DEV_KEYS` é a que a aplicação lê |
| **403** `mfa_required` | Ação crítica com token sem MFA | Gere o token com `--mfa=true` |
| **403** em escrita | Papel sem permissão (Operador, Auditor ou Consultor BPO) | Use um token `ADMIN` |
| **404** em recurso que existe | O recurso é de **outra empresa** (outro `--tenant`) | Use o mesmo tenant do token que criou o recurso |
| **400** `entrada_invalida` ao criar lançamento | Valor enviado como número JSON | Envie o valor entre aspas: `"350.00"` |
| **415** na importação | O arquivo não é OFX | Use OFX 1.x ou 2.x |
| **422** `conta_nao_cadastrada` | Extrato de conta que a empresa não cadastrou | Cadastre a conta primeiro (seção 5.2) |
| Importação parada em `RECEBIDA` ou `PROCESSANDO` | Falha no processamento; expira em 10 minutos | Consulte o `falha`; Admin com MFA pode `POST /importacoes/{id}/reprocessamento` |
| Falha ao subir reclamando do arquivo `jwt-publica.pem` | Chaves de desenvolvimento ainda não geradas | Rode o gerador de token da seção 4 uma vez |
| Tudo sumiu depois de reiniciar | Esperado: Fase A guarda tudo em memória | Aguarde a Fase B (PostgreSQL) |
| `Application run failed` no meio do `verify` | Teste que prova que `prod` não sobe com adaptadores em memória | Esperado; veja se o `BUILD` terminou em `SUCCESS` |

Os logs mostram um `correlationId` por requisição; a resposta de erro traz o mesmo `correlationId`, o que liga o erro ao log. Erros seguem o formato ProblemDetail (RFC 9457), com um `codigo` estável.

## 8. Estrutura do repositório

```
FinancialSoftwarePartnership/
├── README.md                 visão de negócio
├── application/              projeto Maven (shared-kernel, plataforma, fluxocaixa, conciliacao, app)
├── docs/                     manuais, regras, decisões, ADRs, cenários de teste
├── docs/sdd/rules/       regras oficiais v2 e Manifesto de Confiança e Precisão
└── aidlc-docs/               requisitos, histórias, desenho, estado e registro de decisões
```

| Para | Veja |
|---|---|
| Entender o que o produto faz | [Manual do usuário](MANUAL-DO-USUARIO.md) |
| Ver as regras | [Regras de negócio](../sdd/REGRAS-DE-NEGOCIO.md) |
| Ver as decisões | [Decisões do projeto](DECISOES-DO-PROJETO.md) |
| Entender o código por dentro | [Jornada técnica](../jornadas/JORNADA-TECNICA.md) |
| Saber o que é testado | [Cenários de teste](../testes/README.md) |

## 9. Convenções para quem vai mexer no código

- **Português** em código, API e mensagens, com um dicionário de termos para os nomes das regras.
- **Dinheiro** é `Dinheiro` (centavos, positivo) com `Direcao`; saldo é `ValorComSinal`. Nunca `double` ou `float`.
- **Arquitetura**: `domain` sem Spring nem JPA; o fluxo de caixa não conhece a conciliação; a conciliação só enxerga `fluxocaixa.api`; nenhuma porta de repositório com exclusão física. O build quebra se violar.
- **Sem PII** em logs, métricas, eventos e erros; há teste que prova.
- **Toda escrita** gera um registro de auditoria na mesma operação; sem ele, a ação falha.
- **Regras e testes andam juntos**: uma regra nova ganha cenário em [testes/](../testes/README.md) e, se muda o comportamento, atualiza [REGRAS-DE-NEGOCIO.md](../sdd/REGRAS-DE-NEGOCIO.md).
- **Fluxo de trabalho**: o projeto segue o AI-DLC, com aprovação a cada etapa e registro em `aidlc-docs/audit.md`.

## Pipeline: segredos, Actions fixadas e proteção da `main`

- **Detecção de segredos**: o job `segredos` do `ci.yml` roda o gitleaks sobre o histórico do PR e do push. A configuração fica em `.gitleaks.toml` (regras padrão mais uma liberação para um exemplo fictício de `Idempotency-Key` na documentação). Para rodar local: baixe o binário do [gitleaks](https://github.com/gitleaks/gitleaks/releases) e execute `gitleaks git . --config .gitleaks.toml --redact`.
- **Actions fixadas por hash**: todo `uses:` dos workflows aponta para o hash do commit, com a versão em comentário. O Dependabot propõe a atualização do hash. Ao revisar esse PR, confira que o hash novo pertence ao repositório oficial da Action.
- **Proteção da `main`** (configuração do GitHub, que o repositório não controla): em *Settings, Branches, Add branch protection rule* para `main`, marque *Require a pull request before merging* e *Require status checks to pass before merging* e selecione os checks `build`, `sca`, `segredos` e `analisar` (CodeQL). Com um só mantenedor, mantenha as aprovações em 1 e registre isso como exceção temporária (ITGC CM-2).
