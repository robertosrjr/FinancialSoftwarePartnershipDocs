# Segurança e Privacidade — visão consolidada

**Versão**: 1.1 · **Data**: 2026-10-11 · **Produto**: Agente de Bolso
**Para quem**: quem avalia a segurança do produto (liderança técnica, auditoria, jurídico, futuros clientes e um eventual pentester) e quem desenvolve nele.
**O que é este documento**: um **mapa**. Resume a postura de segurança e de privacidade, diz **o que está feito, o que está só planejado e o que falta**, e aponta para o documento onde cada assunto está detalhado. Não repete o conteúdo dos ADRs nem dos requisitos.
**O que não é**: parecer jurídico, relatório de pentest nem certificação. Nada aqui foi validado por terceiro independente.

---

## 1. Resumo

O produto está na **Fase A**: domínio e API completos, **adaptadores em memória, sem dado real de cliente**. A segurança **da aplicação** está bem coberta e testada. A segurança **da infraestrutura** (TLS, cifra em repouso, chaves em cofre, alarmes, WAF) ainda não existe, porque depende das Fases B (PostgreSQL) e C (AWS). Por isso a aplicação **recusa subir em produção com adaptadores em memória** (`GuardaDeProducao`, com teste).

**O que está sólido (implementado e testado)**
- Autorização negada por padrão, por papel, em todo endpoint, com testes exaustivos (1.632 casos nos dois oráculos de autorização).
- Isolamento entre empresas (tenants), confirmação com segundo fator em operações críticas e segregação de funções.
- Trilha de auditoria à prova de adulteração e ledger que só cresce.
- Dados pessoais minimizados: o número da conta bancária nunca é guardado; documentos saem mascarados; logs e métricas sem dado pessoal.
- Varredura de dependências e análise estática no CI, já executadas e com achados corrigidos.

**O que o ciclo 2 acrescentou (U4 a U9, implementado e testado)**
- **Contas pessoais (PF)**: visíveis por padrão só ao Admin; somas de saldo e projeção só PJ, com a parte PF declarada; transferência PJ↔PF recusada. Na U9 foi achada e corrigida uma brecha anterior: quem não via contas PF podia filtrar a listagem de lançamentos pelo id de uma delas.
- **Arquivos não confiáveis (XLSX e CSV)**: tipo pelo conteúdo; fórmula nunca avaliada; recusa de macro, DOCTYPE e entidade externa, bomba de descompressão, entradas duplicadas no zip, zip dentro de zip e caminho perigoso; valor lido do texto cru, sem ponto flutuante; testes adversariais gerados no próprio teste.
- **Busca (U9)**: limites por usuário e por empresa contra enumeração de clientes e fornecedores; busca por CPF ou CNPJ com limite menor e auditada sem gravar o documento; documento comparado por índice cego; CNPJ alfanumérico validado pelo dígito verificador (IN RFB 2.229/2024).
- **Limitador de taxa**: baldes com expiração por inatividade e teto de memória (chave nova recebe 429 quando o teto enche), para não virar vetor de esgotamento de memória.
- **Conexão bancária**: somente leitura; adaptador de teste não sobe em produção; revisões e cancelamentos da origem nunca sobrescrevem o registro anterior.

**As cinco lacunas que mais importam antes do primeiro cliente**

| # | Lacuna | Por quê importa |
|---|---|---|
| 1 | **Nenhum teste de intrusão (pentest) foi feito** | Tudo foi verificado por quem construiu. Falta olhar independente. Ver seção 8. |
| 2 | **Prazos de retenção e eliminação de dados (ADR 011) ainda são proposta** | LGPD arts. 15, 16 e 18. Sem eliminação implementada até a Fase B. |
| 3 | **Sem plano de resposta a incidentes** | LGPD art. 48 exige comunicar incidente com prazo. Não há procedimento escrito. |
| 4 | **Sem RIPD (relatório de impacto) e sem Encarregado (DPO) nomeado** | LGPD arts. 38 e 41. |
| 5 | **TLS, cifra em repouso, cofre de chaves e rotação** | Dependem da Fase C. As chaves hoje são locais e de desenvolvimento. |

A lista completa, com prioridade, está na seção 10.

---

## 2. Como ler o estado de cada controle

| Rótulo | Significa |
|---|---|
| **Feito e testado** | Existe no código e há teste automatizado que falha se o controle quebrar. |
| **Feito, sem teste próprio** | Existe no código, mas nenhum teste cobre especificamente. |
| **Planejado** | Decidido em ADR ou requisito, para a Fase B ou C. Ainda não existe. |
| **Não feito** | Não há decisão nem código. |

---

## 3. OWASP Top 10 (edição 2021)

A edição de referência é a de **2021**. O projeto ainda não adotou formalmente uma versão; se adotar a mais nova, esta tabela deve ser refeita.

| Risco | O que o produto faz | Estado | Detalhe |
|---|---|---|---|
| **A01 Controle de acesso quebrado** | Tudo negado por padrão; papel por endpoint com `@PreAuthorize` (ArchUnit exige em todo endpoint); empresa isolada (outra empresa responde 404); segregação de funções | Feito e testado | [Testes de segurança](../../aidlc-docs/construction/build-and-test/security-test-instructions.md), [ADR 004](../adr/004-maker-checker-revisor-unico.md) |
| **A02 Falhas criptográficas** | AES-256-GCM no conteúdo da auditoria (chave por empresa); HMAC-SHA256 na cadeia e nos índices cegos. **TLS e cifra em repouso: Fase C.** Chaves locais na Fase A | Parcial: aplicação feita e testada; infraestrutura planejada | [ADR 002](../adr/002-auditoria-cadeia-hmac-crypto-shredding.md), [ADR 005](../adr/005-indice-cego-numero-de-conta-nao-armazenado.md) |
| **A03 Injeção** | Sem banco hoje, então sem SQL. Entrada validada em todo endpoint; campo desconhecido rejeitado; leitor de OFX com entidades externas e DTD desligados e limites de tamanho | Feito e testado na Fase A. **Reabrir na Fase B**: consultas parametrizadas e teste de injeção contra o banco | [NFR da U1](../../aidlc-docs/construction/u1-fundacao/nfr-requirements/nfr-requirements.md) |
| **A04 Projeto inseguro** | Decisões registradas em ADRs; regras de negócio com limites explícitos (valores em centavos, ledger imutável, ambiguidade nunca vira automático) | Feito. **Falta um modelo de ameaças formal** (por exemplo STRIDE) | [ADRs](../adr/README.md), [Manifesto](../sdd/rules/MANIFESTO-CONFIANCA-E-PRECISAO.md) |
| **A05 Configuração insegura** | Actuator público só com `health` e `info`; Prometheus só pela porta de gestão em produção; sem stack trace nem mensagem de erro na resposta; documentação interativa da API desligada em produção; produção exige emissor do token ou não inicia | Feito e testado. Cabeçalhos de segurança em toda resposta, inclusive de erro: `nosniff`, `no-store`, `X-Frame-Options: DENY`, política de conteúdo restrita, `Referrer-Policy`, `Permissions-Policy` e HSTS de um ano em conexão segura (2026-10-05; a documentação interativa local fica fora da política restrita) | [Manual Técnico](MANUAL-TECNICO.md) |
| **A06 Componentes vulneráveis** | OWASP Dependency-Check (consulta a [National Vulnerability Database](https://nvd.nist.gov/); falha com CVSS ≥ 7), Dependabot semanal, SBOM CycloneDX, versões fixas | Feito e executado: achou o Tomcat 11.0.24 (9 CVEs), corrigido para 11.0.26 | [Seção "Varredura de dependências"](MANUAL-TECNICO.md) |
| **A07 Falhas de autenticação** | A aplicação **não guarda senha**: valida o token (JWT RS256) de um provedor de identidade; segundo fator exigido por `amr` e `auth_time` (15 minutos); nenhum endpoint emite token | Feito e testado. **O provedor de identidade de produção ainda não foi escolhido** (Fase C). **Não encontrei validação do público do token (`aud`); confirmar** | [NFR da U1](../../aidlc-docs/construction/u1-fundacao/nfr-requirements/nfr-requirements.md) |
| **A08 Integridade de software e dados** | Cadeia HMAC por empresa, verificada em job e sob demanda; ledger só insere; sem desserialização polimórfica; idempotência de toda escrita | Feito e testado. GitHub Actions fixadas por hash do commit desde 2026-10-05 | [ADR 002](../adr/002-auditoria-cadeia-hmac-crypto-shredding.md), [ADR 008](../adr/008-idempotencia-de-escrita.md), [ADR 013](../adr/013-vinculo-ledger-insert-only-e-identificador-deterministico.md) |
| **A09 Falhas de log e monitoramento** | Logs estruturados sem dado pessoal (`LogSanitizer`: cartão, CNPJ, CPF, telefone, e-mail, token, valores); métricas de acesso negado; alertas de cadeia quebrada como evento. **Alarmes reais: Fase C** | Feito e testado, inclusive o formato ECS do perfil `prod` (2026-10-05; `trace.id` e `span.id` corrigidos) | [Seção 6](#6-registros-logs-métricas-e-auditoria) |
| **A10 SSRF** | A aplicação **não faz chamadas a endereços fornecidos pelo usuário** (não há cliente HTTP no código) | Feito (por ausência). **Reavaliar quando entrar a conexão bancária via agregador** | [Spec 002](../sdd/specs/002-conexao-bancaria/spec.md) |

---

## 4. OWASP API Security Top 10 (edição 2023)

O produto é uma API, então esta lista vale tanto quanto a anterior.

| Risco | O que o produto faz | Estado |
|---|---|---|
| **API1 Autorização por objeto quebrada** | Todo acesso por identificador passa pela empresa do token; objeto de outra empresa responde 404, igual a "não existe" | Feito e testado |
| **API2 Autenticação quebrada** | Ver A07 | Parcial |
| **API3 Autorização por propriedade quebrada** | Entrada estrita (campo desconhecido é rejeitado, valor monetário só em texto); respostas montadas por DTO explícito | Feito e testado |
| **API4 Consumo irrestrito de recursos** | Limite de taxa por empresa e por operação (ex.: 120 alocações por minuto); upload de até 10 MB; limites do leitor de OFX; corpo máximo na idempotência. **O limitador vive na memória de cada instância**: com mais de uma instância (Fase C), precisa ser distribuído | Feito e testado. Limite distribuído: planejado |
| **API5 Autorização por função quebrada** | Papel por endpoint, deny-by-default; oráculos com 1.056 e 624 casos | Feito e testado |
| **API6 Acesso irrestrito a fluxos sensíveis** | Segundo fator nas operações críticas; segregação de funções; reenvio seguro com `Idempotency-Key`; regra de ouro da conciliação (dois candidatos nunca viram automático) | Feito e testado |
| **API7 SSRF** | Ver A10 | Feito (por ausência) |
| **API8 Configuração incorreta** | Ver A05 | Parcial |
| **API9 Inventário impróprio** | Contrato OpenAPI conferido por teste (lista os endpoints e suas respostas); uma única versão (`/api/v1`) | Feito e testado |
| **API10 Consumo inseguro de APIs** | Não há consumo de API externa hoje. **Entra com o agregador de Open Finance**: tratar a resposta como entrada não confiável | Não aplicável hoje; reavaliar |

---

## 5. Privacidade e LGPD

**Aviso**: esta seção organiza o que o produto faz em relação à LGPD (Lei 13.709/2018). **Não substitui a análise do jurídico**, que é pendência declarada.

### 5.1 Que dados pessoais e sigilosos o produto trata

| Dado | Onde aparece | Como é protegido | Estado |
|---|---|---|---|
| **Número de agência e conta bancária** | Cadastro da conta; arquivo do extrato | **Nunca guardado.** Só dois índices cegos (HMAC) e os 4 últimos dígitos. Vazamento do banco não expõe número utilizável | Feito e testado ([ADR 005](../adr/005-indice-cego-numero-de-conta-nao-armazenado.md)) |
| **CPF/CNPJ da contraparte** | Descrição do extrato; lançamentos | Índice cego para comparar e **máscara** (`***.***.***-25`) em toda leitura, inclusive **dentro da descrição** | Feito e testado |
| **Descrição da transação** (texto livre, pode ter nome) | Extrato; leitura de pendências | Aparece só a quem tem papel de leitura; fora de logs, métricas e auditoria; **sem prazo de retenção nem anonimização ainda** | Parcial: ver 5.3 |
| **Arquivo OFX original** | Armazenamento de extratos | Só Admin e Auditor baixam; **sem prazo de expiração ainda** | Parcial: ver 5.3 |
| **Conteúdo da trilha de auditoria** (valores antes e depois, motivo) | Trilha | Cifrado por empresa; eliminável por destruição da chave (crypto-shredding) | Feito e testado; destruição de chave: Fase C |
| **Identificação de quem acessa** (IP, usuário) | Log de acesso | IP guardado só como hash; empresa identificada por pseudônimo nos logs | Feito e testado |
| **Telemetria** (logs, métricas, rastros) | Observabilidade | Sem dado pessoal; sem rótulo de empresa em métrica | Feito e testado |

### 5.2 Princípios e artigos da LGPD

| Tema | O que o produto faz | Estado |
|---|---|---|
| **Minimização e finalidade** (art. 6º, I, III) | Só guarda o necessário; o número da conta nem entra | Feito |
| **Segurança** (art. 46) | Controles das seções 3, 4 e 6 | Parcial: aplicação feita; infraestrutura na Fase C |
| **Eliminação e retenção** (arts. 15, 16) | Proposta de prazos e de anonimização em [ADR 011](../adr/011-retencao-e-eliminacao-de-dados-pessoais.md): 5 anos para registro contábil; anonimizar a descrição ao fim do prazo sem apagar a transação | **Planejado** (Fase B); **prazos aguardam o jurídico e o contador** |
| **Direitos do titular** (art. 18) | Procedimento descrito no ADR 011: localizar transações pelo documento (índice cego), avaliar base legal, anonimizar se não houver | **Planejado**; sem canal de atendimento |
| **Comunicação de incidente** (art. 48) | — | **Não feito** |
| **Relatório de impacto** (art. 38) | — | **Não feito** |
| **Encarregado (DPO)** (art. 41) | — | **Não feito** (pendência do ADR 011) |
| **Operador e transferência internacional** (arts. 33 a 36) | Depende da região e dos serviços da AWS e, depois, do agregador de Open Finance | **Não decidido** (Fase C) |
| **Consentimento do Open Finance** | Texto, finalidades e prazo de guarda após desconexão | **Não feito**; pendência do jurídico ([spec 002](../sdd/specs/002-conexao-bancaria/spec.md)) |

### 5.3 O que isso significa na prática hoje
Como a Fase A não guarda nada entre execuções e não tem dado real de cliente, **não há exposição de dado pessoal real**. Isso muda no momento em que existir persistência. **Antes de qualquer cliente usar o produto**, as lacunas 2, 3 e 4 da seção 1 precisam estar resolvidas.

---

## 6. Registros (logs), métricas e auditoria

| Assunto | O que existe | Onde |
|---|---|---|
| **Logs** | Formato ECS em JSON no `stdout` em produção; sem dado pessoal; exceções passam pelo `LogSanitizer`, que mascara cartão, CNPJ, CPF, telefone, e-mail, token e valores, e nunca grava a mensagem crua | Testes `LogSanitizerTest`, `LogSanitizerPropriedadesTest`, `LogDeExcecaoSanitizadoTest` |
| **Métricas** | Acesso negado por motivo; verificação da cadeia; sem rótulo de empresa | [NFR da U1](../../aidlc-docs/construction/u1-fundacao/nfr-requirements/nfr-requirements.md) |
| **Auditoria** | Cadeia HMAC por empresa, com verificação incremental e completa; conteúdo cifrado | [ADR 002](../adr/002-auditoria-cadeia-hmac-crypto-shredding.md) |
| **Quem leu o quê** | Leitura de Auditor e de Consultor BPO gera registro de acesso | [Testes de segurança](../../aidlc-docs/construction/build-and-test/security-test-instructions.md) |
| **Alarmes** | Eventos e métricas existem; **alarmes reais só na Fase C** | Planejado |

---

## 7. Cadeia de suprimentos e pipeline

| Controle | Estado | Observação |
|---|---|---|
| Versões fixas e SBOM CycloneDX | Feito | |
| OWASP Dependency-Check no CI (`sca`, CVSS ≥ 7 falha) | Feito e executado | Chave da National Vulnerability Database guardada como segredo do GitHub |
| Supressão de falso positivo com justificativa e validade | Feito | CVE-2026-53914 (Kotlin), válida até 2027-01-31, em `application/config/dependency-check-suppressions.xml` |
| SpotBugs com FindSecBugs no `verify` | Feito | |
| CodeQL no CI | Feito, **com limite** | Repositório privado sem Code scanning: a análise roda e o resultado fica como artefato, **sem alerta automático**. Alguém precisa abrir o SARIF. Ver [Manual Técnico](MANUAL-TECNICO.md) |
| Dependabot (Maven e Actions) | Feito | |
| Cobertura mínima de 80% em domínio e aplicação | Feito | |
| Proteção da `main` e revisão de PR | **Não verificado** | Os requisitos pedem; o que está configurado no GitHub eu não consegui conferir daqui. Ação do dono do repositório (ver [Manual Técnico](MANUAL-TECNICO.md)) |
| Detecção de segredos (gitleaks no CI, job `segredos`) | Feito (2026-10-05) | Varre o histórico do PR e do push; regras padrão e uma liberação estreita para um falso positivo, em `.gitleaks.toml`. Histórico e árvore atual sem achados |
| Ações do GitHub fixadas por hash do commit | Feito (2026-10-05) | 15 usos em `ci.yml` e `codeql.yml`, com a versão no comentário; o Dependabot atualiza o hash |

---

## 8. Teste de intrusão (pentest)

**Estado: não realizado.** O que existe são testes automatizados escritos pela própria equipe (seção 3 e [testes de segurança](../../aidlc-docs/construction/build-and-test/security-test-instructions.md)). Eles provam que os controles funcionam como projetados, mas **não substituem um olhar independente que tenta quebrar o produto**.

### 8.1 Proposta (para decisão)

| Item | Proposta |
|---|---|
| **Quando** | **Depois da Fase B** (com banco real, para cobrir injeção e persistência) e **antes do primeiro cliente**. Repetir a cada mudança grande e, no mínimo, uma vez por ano |
| **Onde** | Ambiente de homologação com dados fictícios. Nunca em produção nem com dado real |
| **Quem** | Empresa ou profissional independente, que não escreveu o código |
| **Escopo mínimo** | API REST completa; autenticação e emissão de token do provedor de identidade; isolamento entre empresas; segregação de funções e segundo fator; envio de arquivo OFX; limites de taxa e de tamanho; endpoints de gestão |
| **Foco no negócio** | Tentar ler ou alterar dados de outra empresa (BOLA/IDOR); contornar segundo fator; confirmar o próprio item no modo estrito; duplicar ou alterar o ledger; forçar divergência de saldo; abusar de reenvio com `Idempotency-Key` |
| **Técnicas** | Teste manual dirigido pelo OWASP API Security Top 10; varredura dinâmica (DAST) automatizada; fuzzing de entrada e de arquivo OFX; revisão de configuração da nuvem na Fase C |
| **Regras do teste** | Escopo, janela e contatos acordados por escrito antes de começar |
| **Resultado esperado** | Relatório com severidade (CVSS) e prova de cada achado; plano de correção; **reteste** dos achados altos e críticos |
| **Critério para seguir** | Nenhum achado crítico ou alto aberto |

### 8.2 O que dá para fazer antes, sem custo
- Rodar uma varredura dinâmica automatizada (por exemplo, OWASP ZAP em modo básico) contra a aplicação local, como etapa do CI. **Não está implementado**; é uma proposta.
- Escrever o modelo de ameaças (seção 10, item 6). Ele orienta o escopo do pentest.

---

## 9. Continuidade e resposta a incidentes

| Assunto | Estado |
|---|---|
| Metas de recuperação | Definidas para a Fase C: perda de dados até 15 min, retorno em até 4 h, disponibilidade 99,5% ao mês. Ainda não há o que medir |
| Backup e restauração | Planejado (Fase C) |
| **Plano de resposta a incidentes** | **Não feito.** Deve dizer quem decide, como conter, como avisar o titular e a ANPD no prazo da LGPD, e como registrar |
| Procedimento de deploy de emergência | Previsto para o Infrastructure Design |
| Destruição de chave de uma empresa (crypto-shredding) | Procedimento com dupla aprovação, a definir na Fase C |
| Rotação das chaves de pseudonimização | Exige reindexar as contas; procedimento a definir ([ADR 005](../adr/005-indice-cego-numero-de-conta-nao-armazenado.md)) |

---

## 10. Lacunas e plano

Ordem sugerida. A coluna **Quando** usa as fases do projeto, não datas.

| # | Lacuna | Risco | Quando | Dono sugerido |
|---|---|---|---|---|
| 1 | Pentest independente | Alto | Depois da Fase B, antes do 1º cliente | Liderança técnica |
| 2 | Prazos de retenção e eliminação (ADR 011): validar e implementar | Alto (LGPD) | Validar já; implementar na Fase B | Jurídico e contador; depois o time |
| 3 | Plano de resposta a incidentes (LGPD art. 48) | Alto | Antes do 1º cliente | Liderança técnica e jurídico |
| 4 | RIPD e Encarregado (DPO) | Alto (LGPD) | Antes do 1º cliente | Negócio e jurídico |
| 5 | TLS, cifra em repouso, cofre de chaves, rotação | Alto | Fase C | Time |
| 6 | **Modelo de ameaças formal** (STRIDE ou equivalente) | Médio | Junto do NFR Design da Fase B | Time |
| 7 | Reabrir injeção e consulta parametrizada com o banco; teste de carga e de isolamento no PostgreSQL | Alto | Fase B | Time |
| 8 | Limite de taxa distribuído (hoje na memória de cada instância, com teto e expiração desde a U9) | Médio | Fase C | Time |
| 9 | ~~Fixar GitHub Actions por hash; ativar detecção de segredos~~ feito em 2026-10-05. **Falta conferir a proteção da `main`** (ação do dono do repositório) | Médio | Já | Dono do repositório |
| 10 | Validar o público do token (`aud`) e escolher o provedor de identidade | Médio | Fase C | Time |
| 11 | ~~Configurar e testar cabeçalhos de segurança HTTP~~ feito em 2026-10-05 | Baixo a médio | Feito | Time |
| 12 | ~~Teste do formato de log ECS no perfil `prod`~~ feito em 2026-10-05 | Baixo | Feito | Time |
| 13 | Alarmes (cadeia quebrada, picos de 401/403, falha de job) | Médio | Fase C | Time |
| 14 | Texto de consentimento e prazo de guarda do Open Finance | Alto, se a conexão bancária entrar | Antes da spec 002 | Jurídico |
| 15 | Canal e procedimento de atendimento ao titular (art. 18) | Médio | Fase B | Negócio e time |
| 16 | Eliminação de dados (ADR 011) precisa recalcular também o índice de busca da U9 | Médio (LGPD) | Fase B | Time |

---

## 11. Onde aprofundar

| Assunto | Documento |
|---|---|
| Decisões de segurança | [ADR 002](../adr/002-auditoria-cadeia-hmac-crypto-shredding.md) (auditoria), [004](../adr/004-maker-checker-revisor-unico.md) (segregação), [005](../adr/005-indice-cego-numero-de-conta-nao-armazenado.md) (conta bancária), [008](../adr/008-idempotencia-de-escrita.md) (idempotência), [011](../adr/011-retencao-e-eliminacao-de-dados-pessoais.md) (retenção), [013](../adr/013-vinculo-ledger-insert-only-e-identificador-deterministico.md) (ledger) |
| Requisitos de segurança (SECURITY-01 a 15) | [NFR da U1](../../aidlc-docs/construction/u1-fundacao/nfr-requirements/nfr-requirements.md) |
| Como cada controle é testado | [Instruções de teste de segurança](../../aidlc-docs/construction/build-and-test/security-test-instructions.md) |
| Cenários de teste da plataforma | [CENARIOS-PLATAFORMA.md](../testes/CENARIOS-PLATAFORMA.md) |
| Rodar a varredura de dependências e o CodeQL | [Manual Técnico](MANUAL-TECNICO.md) |
| Regras de negócio que protegem o dinheiro | [Manifesto de Confiança e Precisão](../sdd/rules/MANIFESTO-CONFIANCA-E-PRECISAO.md), [Regras de Negócio](../sdd/REGRAS-DE-NEGOCIO.md) |
| Conexão bancária (Open Finance) | [Spec 002](../sdd/specs/002-conexao-bancaria/spec.md), [RULES-CX](../sdd/rules/REGRAS-CONEXAO-BANCARIA.md) |
| Princípios do projeto | [Constituição](../sdd/constitution.md) |

**Para o usuário final** (papéis, segundo fator, privacidade em linguagem simples): [Manual do Usuário](MANUAL-DO-USUARIO.md).
