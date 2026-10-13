# Arquitetura de serviços (TOGAF)

Este documento descreve o **Agente de Bolso** com os conceitos do TOGAF Standard, 10ª edição: a visão de realização entre camadas (da oferta vendida até a tecnologia), a arquitetura de negócio (capacidades, fluxo de valor, atores, serviços e princípios), os serviços de aplicação e de tecnologia, a análise de lacunas e, em seção separada, os pacotes de trabalho. A modelagem segue a skill [`service-modeling-skill`](../../.claude/skills/service-modeling-skill/SKILL.md) ([referência completa](../../.claude/skills/service-modeling-skill/references/service-modeling-skill.md)).

> **Revisão de 10/10/2026:** reestruturado após duas críticas severas e imparciais do caderno "Master TOGAF" (NotebookLM). O que foi aceito e o que foi recusado está na seção 14.

## 1. Visão de realização entre camadas

O TOGAF organiza a arquitetura em quatro domínios: **Negócio, Dados, Aplicação e Tecnologia**. A tabela abaixo não é uma lista desses domínios: é uma **visão de realização**, que mostra como uma oferta vendida é realizada, de cima para baixo, por serviços de cada domínio. "Oferta" é o termo de mercado; no TOGAF, é o **Produto**, uma entidade da arquitetura de negócio.

| Conceito | Domínio TOGAF | O que representa | No Agente de Bolso |
|---|---|---|---|
| **Oferta de Serviços** (Produto) | Negócio | O pacote comercializável dos serviços | Starter, Pro e BPO / Consultoria (seção 2) |
| **Serviço de Negócio** | Negócio | A funcionalidade de negócio que gera valor, independente de tela e de canal | Controlar o fluxo de caixa, conciliar o extrato bancário, apurar a DRE gerencial... (seção 6) |
| **Serviço de Aplicação** | Aplicação | A função do software que executa o serviço | Registrar lançamento, conciliar transação, importar extrato... (seção 8) |
| **Serviço de Tecnologia** | Tecnologia | A infraestrutura que sustenta a aplicação | Runtime Java 25 e Spring Boot 4; memória na Fase A, PostgreSQL na Fase B (seção 9) |

Leitura: a oferta empacota serviços de negócio; cada serviço de negócio é realizado por serviços de aplicação; cada serviço de aplicação roda sobre serviços de tecnologia. Um serviço de negócio aparece em várias ofertas, e um endpoint, uma tela ou um módulo nunca é um serviço de negócio.

## 2. Ofertas (Produto)

Definidas em [`PRODUCT-SPEC-v4.md` §5](../geral/PRODUCT-SPEC-v4.md).

| Oferta | Público | Serviços de negócio incluídos | Canais |
|---|---|---|---|
| **Starter** | MEIs e autônomos de serviços | Controlar o fluxo de caixa | WhatsApp |
| **Pro** | PMEs de serviços (Simples / Presumido) | Tudo da Starter; receber dados financeiros; conciliar o extrato bancário; apurar a DRE gerencial; acompanhar a adimplência; gerar inteligência financeira | WhatsApp, painel web, app |
| **BPO / Consultoria** | Escritórios de BPO e consultores | Tudo da Pro; gestão financeira delegada (BPO) | Portal BPO, WhatsApp dos clientes |

Os serviços de suporte da seção 7 (identidade e acesso, auditoria) estão presentes em todas as ofertas, mas não são vendidos como fim.

## 3. Mapa de capacidades (esqueleto, nível 1 e 2)

Capacidade é **o que** o negócio é capaz de fazer, estável e independente de organograma e de tecnologia. Esqueleto inicial, a ser completado (nível 3 e avaliação de maturidade).

| Nível 1 | Nível 2 |
|---|---|
| **Gestão financeira operacional** | Registro de contas a pagar e a receber; liquidação (baixas); parcelamentos e repetições; cadastros financeiros (categorias, contas, centros de custo, formas de pagamento); separação entre empresa e sócio |
| **Captação de dados financeiros** | Importação de extratos e planilhas; conexão bancária (Open Finance); revisão antes da gravação |
| **Conciliação bancária** | Conferência de saldos; reconhecimento de movimentos; tratamento de exceções e duplicidades; correção por estorno |
| **Relatórios gerenciais** | Apuração da DRE por competência; consultas, busca e totais; exportação para o contador |
| **Gestão de clientes BPO** | Atendimento a várias empresas clientes; separação entre clientes; pré-fechamento contábil |
| **Segurança da informação, privacidade e conformidade** | Gestão de identidade e acesso; segregação de funções; trilha de auditoria; proteção de dados pessoais (LGPD) |

## 4. Fluxo de valor: "Do extrato ao resultado do mês"

Fluxo horizontal, do ponto de vista do cliente. Cada etapa é habilitada por capacidades da seção 3.

| Etapa | Valor para o cliente | Capacidades que habilitam |
|---|---|---|
| 1. Receber dados | O extrato e os lançamentos chegam sem digitação | Captação de dados financeiros |
| 2. Classificar | Cada valor na categoria certa, com competência correta | Gestão financeira operacional |
| 3. Conciliar | Cada movimento do banco provado contra um lançamento, sem duplicar nem apagar | Conciliação bancária |
| 4. Apurar | DRE do mês por competência, comparável e explicável | Relatórios gerenciais |
| 5. Decidir | O dono sabe quanto ganhou, quanto tem e o que vai vencer | Relatórios gerenciais |

Interessado principal: o dono da empresa (MEI ou PME). No BPO, o operador executa as etapas 1 a 4 em nome do cliente.

## 5. Atores, papéis e mapa de organização

| Ator | Papel no sistema | Capacidades que usa |
|---|---|---|
| Dono da empresa (MEI ou sócio de PME) | Admin | Todas; aprova o que é sensível (segregação) |
| Funcionário financeiro | Operador | Gestão financeira operacional; captação; conciliação (propõe) |
| Consultor de BPO | Consultor BPO (só nas empresas vinculadas) | Gestão de clientes BPO; consultas |
| Auditor | Auditor (só leitura, com MFA para o original) | Trilha de auditoria; relatórios |
| Contador | Externo (recebe exportação) | Relatórios gerenciais (consome) |
| Banco e agregador Open Finance | Sistema externo | Captação de dados financeiros (fornece) |

Unidade organizacional: cada **empresa cliente** é a fronteira de dados e de acesso (Pilar 5 do manifesto). O escritório de BPO é uma unidade que atende várias empresas, sem misturá-las.

## 6. Serviços de Negócio

Nome com verbo no infinitivo. "Situação" diz onde o serviço está hoje.

| Serviço de Negócio | Valor entregue | Regras (contrato de serviço) | Situação |
|---|---|---|---|
| **Controlar o fluxo de caixa** | Saber o que entra, o que sai e o que vai vencer, com saldo confiável | [RULES-FC](../sdd/rules/REGRAS-FLUXO-DE-CAIXA.md) | Construído (ciclo 1, U4, U7, U8) |
| **Receber dados financeiros** | O extrato e os lançamentos chegam sem digitação, por arquivo, planilha ou conexão bancária | [RULES-CX](../sdd/rules/REGRAS-CONEXAO-BANCARIA.md), [ADR 015](../adr/015-ofx-permanece-adaptador-de-entrada-da-conciliacao.md) | Construído com banco simulado (U5, U6, U7); Pluggy real depois |
| **Conciliar o extrato bancário** | Provar cada movimento do banco contra um lançamento, sem duplicar nem apagar | [RULES-CB](../sdd/rules/REGRAS-CONCILIACAO-BANCARIA.md) | Construído (ciclo 1, U5, U5b, U6) |
| **Apurar a DRE gerencial** | Resultado do mês por competência, para decidir | [REGRAS-DRE](../sdd/rules/REGRAS-DRE.md), [guia](../sdd/rules/GUIA-BOAS-PRATICAS-DRE.md) | A fazer (U13) |
| **Acompanhar a adimplência** | Saber quem está devendo e avisar antes | PRODUCT-SPEC RF-1.4 | Fora do ciclo 2 (Q-15); a repetição de receita da U8 é a base |
| **Gerar inteligência financeira** | Sugerir e explicar com base no histórico | PRODUCT-SPEC RF-1.5 | Parcial: categorização por histórico na U12 |
| **Gestão financeira delegada (BPO)** | O escritório opera o financeiro de várias empresas clientes, cada uma isolada | Manifesto, Pilar 5; BR-U4-50 a 53 | Parcial: papel Consultor BPO construído; portal BPO a fazer |

## 7. Serviços de suporte (plataforma)

Presentes em todas as ofertas, mas não contratados como fim. Habilitam os serviços de negócio.

| Serviço de suporte | O que garante | Regras | Situação |
|---|---|---|---|
| Gerir identidade e acesso multiempresa | Cada empresa vê só o que é seu; quem faz não confirma sozinho | [ADR 004](../adr/004-maker-checker-revisor-unico.md) | Construído (plataforma) |
| Auditar as operações | Provar quem fez o quê e quando, sem dado pessoal no log | [ADR 002](../adr/002-auditoria-cadeia-hmac-crypto-shredding.md) | Construído (plataforma) |
| Encontrar e operar em lote | Busca, seleção, totais, operações em massa e exportação, com prévia (capacidade de aplicação, não serviço de negócio) | RULES-FC (FC-38) | A fazer (U9 a U11) |

## 8. Serviços de Aplicação, componentes e interfaces

O **serviço de aplicação** é a função que o software expõe; o **componente** é o módulo que o implementa; a **interface** é por onde se chega a ele. O contrato de serviço de cada interface é o OpenAPI mais as regras de negócio citadas.

| Serviço de Aplicação | Realiza | Componente | Interface | Desenho |
|---|---|---|---|---|
| Registrar lançamento; registrar baixa; consultar saldos e projeção; manter cadastros | Controlar o fluxo de caixa | `fluxocaixa` | `/api/v1/fluxo-caixa/...` | [services.md](../../aidlc-docs/inception/application-design/services.md) (S6, S7) |
| Gerenciar séries (parcelar e repetir) | Controlar o fluxo de caixa | `fluxocaixa` | `/api/v1/fluxo-caixa/series` | [U8](../../aidlc-docs/construction/u8-parcelamentos-e-repeticoes/functional-design/business-logic-model.md) |
| Importar extrato (OFX, planilha); importar lançamentos | Receber dados financeiros | `importacao`, `conciliacao`, `fluxocaixa` (porta `LancamentosApi`) | `/api/v1/importacao/...`, `/api/v1/conciliacao/importacoes` | [U6](../../aidlc-docs/construction/u6-importacao-flexivel-de-extrato/functional-design/business-logic-model.md), [U7](../../aidlc-docs/construction/u7-importacao-de-lancamentos/functional-design/business-logic-model.md) |
| Receber entrada canônica; sincronizar conexão bancária | Receber dados financeiros | `conciliacao` | `/api/v1/conciliacao/...` | services.md (S10), [U5](../../aidlc-docs/construction/u5-entrada-canonica-e-agregador/functional-design/), [U5b](../../aidlc-docs/construction/u5b-revisao-e-cancelamento-pela-origem/functional-design/business-logic-model.md) |
| Conciliar transação; tratar exceção e duplicidade; desfazer | Conciliar o extrato bancário | `conciliacao` | `/api/v1/conciliacao/...` | services.md (S1 a S5) |
| Consultar e totalizar lançamentos e transações | Encontrar e operar em lote (suporte) | `fluxocaixa`, `conciliacao`; `Selecao` e `Totais` no `shared-kernel` | parâmetros de busca nas listagens | U9 |
| Calcular DRE | Apurar a DRE gerencial | A definir (U13) | A definir | REGRAS-DRE |
| Autenticar, autorizar, segregar, garantir idempotência, registrar lotes | Gerir identidade e acesso (suporte) | `plataforma` | transversal | ADR 004, [ADR 008](../adr/008-idempotencia-de-escrita.md) |
| Registrar trilha de auditoria | Auditar as operações (suporte) | `plataforma` | transversal | ADR 002 |
| **Agregador Open Finance (externo: Pluggy)** | Receber dados financeiros | Componente de aplicação externo | API do fornecedor, atrás de adaptador | RULES-CX |

Dependências entre componentes: [component-dependency.md](../../aidlc-docs/inception/application-design/component-dependency.md) e [ADR 001](../adr/001-monolito-modular-hexagonal.md).

## 9. Serviços de Tecnologia

| Serviço de Tecnologia | Sustenta | Documento |
|---|---|---|
| Runtime Java 25 + Spring Boot 4, monólito modular hexagonal (Spring Modulith) | Todos os serviços de aplicação | [ADR 001](../adr/001-monolito-modular-hexagonal.md) |
| Persistência: memória atrás de portas (Fase A); PostgreSQL (Fase B, por último no ciclo 2) | Todos | [ADR 003](../adr/003-persistencia-em-fases.md) |
| Rede e TLS até o agregador | Receber dados financeiros | RULES-CX |
| Observabilidade (logs ECS sem dado pessoal, métricas) | Todos | `aidlc-docs/construction/build-and-test/` |

### 9.1 Proteção de dados (tema transversal aos quatro domínios)

| Domínio | O que cabe a ele | Onde está |
|---|---|---|
| Negócio | Política de privacidade e obrigações da LGPD; base legal; quem pode ver o quê | Manifesto (Pilar 5); BR-U4-50 a 53 |
| Dados | Classificação de dado pessoal; retenção e eliminação | [ADR 011](../adr/011-retencao-e-eliminacao-de-dados-pessoais.md) |
| Aplicação | Mascaramento; índice cego de documentos e contas; nada de dado pessoal em log | [ADR 005](../adr/005-indice-cego-numero-de-conta-nao-armazenado.md) |
| Tecnologia | Criptografia; crypto-shredding da trilha de auditoria; chaves | ADR 002 |

## 10. Princípios de arquitetura

Formato do TOGAF: nome, declaração, justificativa e implicações. Os princípios vêm dos mandatos do [manifesto de confiança e precisão](../sdd/rules/MANIFESTO-CONFIANCA-E-PRECISAO.md), que continua sendo a fonte; esta tabela só os resume.

| Princípio | Declaração | Justificativa | Implicações |
|---|---|---|---|
| **Dinheiro exato** (Pilar 1) | Valores são centavos inteiros, calculados pelo sistema, nunca arredondados em silêncio | Estrutura Conceitual: representação fiel | Nada de `double`; somas exatas; recusa em vez de arredondar |
| **Nada se apaga** (Pilar 2) | Toda correção é estorno ou complementação, com autor, data e motivo | ITG 2000 (R1) | Sem exclusão física; desfazer gera estorno; trilha de auditoria imutável |
| **Competência e classificação corretas** (Pilar 3) | Cada fato tem três datas e a natureza certa na DRE | NBC TG 1001 e 1002 | Competência nunca copiada do vencimento por regra automática; DRE só por competência |
| **O banco é conferido, nunca adivinhado** (Pilar 4) | O que vem do banco é conferido, sem duplicar nem sobrescrever; o duvidoso vai para uma pessoa | Integridade de saldos | Conciliação automática só com prova exata; exceções para decisão humana |
| **Cada empresa é uma entidade** (Pilar 5) | O patrimônio da empresa não se mistura com o do sócio nem com o de outra empresa | Princípio da Entidade; Código Civil, art. 49-A | Empresa é a fronteira dos dados; contas PF fora da DRE; BPO sem mistura entre clientes |
| **Um conceito, um nome** (Pilar 6) | O mesmo vocabulário no negócio, no código e na API | Linguagem ubíqua (DDD) | Glossário canônico; nomes de fornecedores só nos adaptadores |

## 11. Análise de lacunas (atual × alvo)

| Elemento | Atual (outubro de 2026) | Alvo | Lacuna |
|---|---|---|---|
| Controlar o fluxo de caixa | Construído | Construído | Nenhuma |
| Receber dados financeiros | Banco simulado | Pluggy real | Integração real (Fase B) |
| Conciliar o extrato bancário | Construído | Construído | Nenhuma |
| Apurar a DRE gerencial | Regras prontas | Construído | U13 |
| Encontrar e operar em lote | Desenho | Construído | U9 a U11 |
| Gestão financeira delegada (BPO) | Papel Consultor BPO | Portal BPO | Portal e pré-fechamento |
| Acompanhar a adimplência | Base (repetição de receita) | Construído | Fora do ciclo 2 |
| Gerar inteligência financeira | Categorização (U12) | Sugestões explicadas | Depois do ciclo 2 |
| Persistência | Memória (Fase A) | PostgreSQL (Fase B) | Fase B |
| Canais | API REST | WhatsApp, painel web, app | Fora do escopo do backend |
| Mapa de capacidades | Esqueleto (seção 3) | Nível 3 com maturidade | Completar |

## 12. Pacotes de trabalho (Fases E e F) — adaptação do ADM

Esta seção não é arquitetura de negócio: é o planejamento de entrega. No TOGAF, pacotes de trabalho pertencem às Fases E e F; aqui ficam as unidades do ciclo 2 (AI-DLC), ligadas aos serviços que realizam. É uma adaptação declarada do ADM.

| Unidade | Serviço de Negócio | Ofertas | Serviço de Aplicação | Serviço de Tecnologia |
|---|---|---|---|---|
| U4 Cadastros e titularidade | Controlar o fluxo de caixa | Starter, Pro, BPO | Manter cadastros; titularidade PF/PJ | Backend comum |
| U5 Entrada canônica e agregador | Receber dados financeiros | Pro, BPO | Receber entrada canônica; sincronizar conexão | Backend comum; agregador simulado |
| U5b Revisão pela origem | Conciliar o extrato bancário | Pro, BPO | Versões; estorno pelo sistema | Backend comum |
| U6 Importação de extrato | Receber dados financeiros | Pro, BPO | Importar extrato | Backend comum |
| U7 Importação de lançamentos | Receber dados financeiros; controlar o fluxo de caixa | Starter, Pro, BPO | Importar lançamentos | Backend comum |
| U8 Parcelamentos e repetições | Controlar o fluxo de caixa | Starter, Pro, BPO | Gerenciar séries | Backend comum |
| U9 Busca, seleção e totalizador | Encontrar e operar em lote (suporte) | Pro, BPO (Starter: busca básica) | Consultar e totalizar | Backend comum; índices e agregação no banco na Fase B |
| U10 e U11 Massa e exportação | Encontrar e operar em lote (suporte) | Pro, BPO | Operações em massa; exportação | A definir |
| U12 Categorização por histórico | Gerar inteligência financeira | Pro, BPO | Sugerir categoria | Backend comum |
| U13 DRE e relatórios | Apurar a DRE gerencial | Pro, BPO | Calcular DRE | A definir |

Situação de cada unidade: [unit-of-work.md](../../aidlc-docs/inception/application-design/unit-of-work.md) e [aidlc-state.md](../../aidlc-docs/aidlc-state.md).

## 13. Obstáculos e riscos

| Obstáculo | Mitigação |
|---|---|
| Ofertas citam serviços ainda não construídos (adimplência, inteligência) | A seção 11 marca a lacuna; nenhuma oferta é vendida com serviço não construído |
| Tentação de chamar endpoint, tela ou módulo de "serviço de negócio" | Seção 8 separa serviço de aplicação, componente e interface |
| Canais ainda não construídos; só a API REST existe | O canal é atributo da oferta, não do serviço |
| Mapa de capacidades só no nível 2, sem maturidade | Completar o nível 3 e o mapa de calor na próxima revisão |
| Contrato de arquitetura (acordo de governança das Fases F e G) não existe | Não se aplica no porte atual; os contratos de serviço são o OpenAPI e as regras |

## 14. Revisão crítica (caderno "Master TOGAF", 10/10/2026)

**Aceito:** tabela das quatro linhas como visão de realização (não como domínios); separação entre serviço de aplicação, componente e interface; Pluggy como componente de aplicação externo; proteção de dados como tema transversal; identidade, auditoria e operação em lote como serviços de suporte; "Atender várias empresas" trocado por "Gestão financeira delegada (BPO)"; capacidade de segurança com nome de negócio; princípios no formato de quatro partes; análise de lacunas e mapa de organização.

**Recusado:** remover a matriz de unidades (o documento é a referência de serviços do projeto em todas as fases; a matriz fica, separada e rotulada como Fases E e F); "OpenAPI não é contrato" (o caderno confundiu contrato de arquitetura, de governança, com contrato de serviço, que é o que o OpenAPI e as regras cumprem); "consultar e totalizar não é serviço de aplicação" (é uma função que o software expõe; só foi renomeado).

## 15. Como manter

Todo Functional Design novo traz a seção "Mapeamento de serviços (TOGAF)", como a [U8](../../aidlc-docs/construction/u8-parcelamentos-e-repeticoes/functional-design/business-logic-model.md) (seção 6). Ao fechar uma unidade, atualize as seções 6, 8, 11 e 12.
