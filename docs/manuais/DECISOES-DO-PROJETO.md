# Decisões do Projeto

**Versão**: 1.1 · **Data**: 2026-10-11 · **Produto**: Agente de Bolso
**Para quem**: time de negócio, liderança técnica e auditoria. Registra o que foi decidido, por quê e onde está o detalhe. Decisões de arquitetura com texto completo ficam nos [ADRs](../adr/README.md); as regras de negócio estão em [REGRAS-DE-NEGOCIO.md](../sdd/REGRAS-DE-NEGOCIO.md).

**Situação**: **Decidida** (em vigor), **Proposta** (aguarda validação), **Substituída** (trocada por decisão posterior).

---

## 1. Produto e escopo

| # | Decisão | Motivo | Situação |
|---|---|---|---|
| P1 | É um **produto de mercado**, não uma prova de conceito: a primeira entrega reduz escopo funcional, nunca qualidade | Dados bancários de terceiros e compliance bloqueante | Decidida |
| P2 | **Service-First**: o núcleo de serviços é independente de canal. A primeira entrega expõe só **API REST**; WhatsApp, painel web e aplicativo serão adaptadores | Canais mudam, regras não | Decidida |
| P3 | Público: PMEs, MEIs, consultorias e escritórios de BPO **de prestação de serviços** | Foco de mercado da especificação do produto | Decidida |
| P4 | Começar pela **conciliação bancária**, com um **fluxo de caixa mínimo** ao lado, porque a conciliação precisa de lançamentos para cruzar | Entrega de valor mais cedo | Decidida |
| P5 | Fora desta entrega: Open Finance, CSV e PDF, IA nas sugestões, relatórios de DRE e DFC, CRM financeiro, cartão de crédito, moeda estrangeira, fechamento de períodos | Foco e qualidade | Decidida |
| P6 | Perguntas ao usuário sempre trazem uma **sugestão de resposta com justificativa** | Decisão mais rápida e rastreável | Decidida |

## 2. Regras de referência

| # | Decisão | Motivo | Situação |
|---|---|---|---|
| R1 | As regras de `docs/sdd/rules/` (fluxo de caixa e conciliação, v2.0.0) são a **referência do produto**, com emendas propostas a CB-16, CB-20, CB-55, CB-61 e FC-33 | Alinhar o produto às normas contábeis citadas e às boas práticas | Decidida (2026-10-02) |
| R2 | O **Manifesto de Confiança e Precisão** (4 pilares) é o critério de revisão de desenho e de código. Conflito com um pilar é levantado ao usuário | Piso de confiança do produto | Decidida (2026-10-03) |
| R3 | As fontes normativas em `docs/normas/` são **resumos**, sem o texto oficial. A validação por compliance e contador é pendência (risco R-05) | Evitar tratar resumo como norma | Decidida; validação pendente |

## 3. Decisões de requisitos (primeira entrega)

| # | Decisão | Situação |
|---|---|---|
| Q1 | Canal de entrada: somente API REST | Decidida |
| Q2 | Lançamentos: módulo `fluxocaixa` mínimo, consumido pela conciliação por API interna | Decidida; ampliada por V2-Q2 |
| Q3 | Formato de extrato: somente OFX | Decidida |
| Q4 | Vínculo: chave exata vale na hora; valor e data ficava sempre "sugerido" | **Substituída** por V2-Q14 |
| Q5 | Sugestão de categoria: regras determinísticas (histórico do usuário e regras fixas), atrás de uma porta para IA futura | Decidida |
| Q6 | Tolerância de data: configurável, padrão 3 dias | Decidida |
| Q7 | Maven multi-módulo, Spring Modulith, hexagonal por pacote, ArchUnit | Decidida |
| Q8 | Entre módulos: API interna síncrona e eventos de domínio | Decidida |
| Q9 | Persistência: PostgreSQL, Flyway, JPA, Testcontainers (em fases, ver ADR 003) | Decidida |
| Q10 | Multitenancy: `tenant_id`, filtro na aplicação e RLS no PostgreSQL | Decidida |
| Q11 | Autenticação: OAuth2 Resource Server com JWT; tenant, usuário e papéis vêm do token | Decidida |
| Q12 | Arquivo original: S3 (MinIO em desenvolvimento), atrás de uma porta | Decidida |
| C1 | Pacote base `br.com.robertosrjr.agentedebolso` | Decidida |
| C2 | Normas das skills de compliance tratadas como bloqueantes | Decidida |
| C3 | Segregação por papel: Operador importa e propõe, Admin confirma e desfaz, Consultor BPO só lê; usuário único faz tudo com o autor registrado | Decidida |
| Q14 | Extensão Security Baseline (SECURITY-01 a 15) **bloqueante** | Decidida |
| Q15 | Extensão Property-Based Testing (PBT-01 a 10) **bloqueante** | Decidida |

## 4. Decisões da atualização para as regras v2 (2026-10-02)

| # | Decisão | Situação |
|---|---|---|
| V2-Q1 | Regras `docs/sdd/rules/` como referência, com emendas | Decidida |
| V2-Q2 | A conta vive no fluxo de caixa (**conta financeira**); a conta bancária da conciliação aponta para ela e mantém a identificação protegida | Decidida |
| V2-Q3 | Ordem: fechar a U3a, depois U1b (idempotência), U2b (contas, baixas, transferências) e U3b | Decidida |
| V2-Q4 | Português em código, API e banco, com dicionário de termos para os nomes das regras | Decidida |
| V2-Q5 | FITID reutilizado com conteúdo diferente **entra com aviso**; o ordinal conta ocorrências de linhas idênticas | Decidida e implementada (ADR 006) |
| V2-Q6 | Lançamento exige descrição, vencimento, competência (data) e categoria; toda alteração exige motivo | Decidida; construção na U2b |
| V2-Q7 | Alocação parcial: uma transação em vários lançamentos e vice-versa, com tetos nos dois lados | Decidida; implementada na U3b |
| V2-Q8 | Estado do saldo: conferido, divergente ou sem conferência com motivo; a **primeira importação é "sem conferência"** | Decidida e implementada |
| V2-Q9 | DRE com 8 naturezas, DFC por atividade, 2 níveis de hierarquia, plano enxuto validado por contador | Decidida; **validação do contador pendente** |
| V2-Q10 | Contas: bancária, caixa e carteira digital agora; cartão de crédito no backlog | Decidida |
| V2-Q11 | Só OFX; Open Finance, CSV e PDF no backlog | Decidida |
| V2-Q12 | Documento da contraparte opcional, guardado só como índice cego | Decidida |
| V2-Q13 | `Idempotency-Key` obrigatório em toda escrita (24 h; 409 se o corpo muda; 400 se falta) | Decidida; construção na U1b |
| V2-Q14 | Camada 2 concilia sozinha só com candidato único **e** documento da contraparte coincidente | Decidida; implementada na U3b |
| V2-Q15 | Camada 3 cria lançamento e baixa sozinha só para **regras de reconhecimento cadastradas** por Admin com MFA | Decidida; implementada na U3b |

## 5. Decisões de arquitetura

| # | Decisão | Motivo | Onde |
|---|---|---|---|
| A1 | **Monolito modular** com arquitetura hexagonal e um único artefato implantável | Simplicidade operacional com fronteiras claras entre módulos | [ADR 001](../adr/001-monolito-modular-hexagonal.md) |
| A2 | Trilha de auditoria em **cadeia HMAC por empresa**, com cifragem por empresa e **crypto-shredding** ao fim da retenção | Provar que nada foi adulterado e atender à eliminação sem apagar a cadeia | [ADR 002](../adr/002-auditoria-cadeia-hmac-crypto-shredding.md) |
| A3 | **Persistência em fases**: memória (A), PostgreSQL local (B), AWS (C) | Avançar as regras sem esperar a infraestrutura | [ADR 003](../adr/003-persistencia-em-fases.md) |
| A4 | **Maker-checker** com modo estrito e exceção de revisor único | Segregação de funções sem travar quem trabalha sozinho | [ADR 004](../adr/004-maker-checker-revisor-unico.md) |
| A5 | **Número da conta bancária não é armazenado**: só índices cegos e 4 últimos dígitos | LGPD e sigilo bancário | [ADR 005](../adr/005-indice-cego-numero-de-conta-nao-armazenado.md) |
| A6 | Deduplicação por **FITID com conferência de conteúdo**, chave versionada e FITID como hash | Não duplicar, não perder, não forjar chave | [ADR 006](../adr/006-deduplicacao-por-fitid-com-conferencia-de-conteudo.md) |
| A7 | **Retenção e eliminação de dados pessoais**: 5 anos, anonimização da descrição, expiração do arquivo | LGPD arts. 15, 16 e 18 sem violar a imutabilidade contábil | [ADR 011](../adr/011-retencao-e-eliminacao-de-dados-pessoais.md) (**proposta**) |
| A8 | Importação **assíncrona**: o envio responde "recebido" e a pessoa consulta o andamento | Canais futuros são assíncronos; isola falhas de armazenamento | Desenho (Q3 do Application Design) |
| A9 | Ledger de conciliação e trilha de auditoria são **coisas distintas**; cada evento do ledger também gera registro na trilha | O ledger é regra de negócio; a trilha é transversal | Desenho (Q2) |
| A10 | Política de segregação na plataforma; o **estado** da proposta fica no agregado de cada módulo | Regra única, significado de "confirmar" por domínio | Desenho (Q4) |

### Decisões do desenho da versão 2 (Application Design aprovado em 2026-10-03)

| # | Decisão | Motivo |
|---|---|---|
| D7 | A **baixa** pertence ao agregado do lançamento; saldo e projeção usam consultas agregadas | Consistência do teto e da situação; desempenho de 500 ms com 50 mil baixas |
| D8 | A **diferença de liquidação é calculada**, não vira um segundo lançamento | Evita dupla contagem; estornar a baixa final desfaz a diferença |
| D9 | O **vínculo carrega o valor alocado**; o estado da transação é calculado; o ledger só recebe inclusões | Permite 1-para-N e N-para-1 com os dois tetos |
| D10 | O fluxo de caixa oferece à conciliação: baixa de conciliação, lançamento com baixa, estorno e transferência, todos idempotentes pelo identificador da alocação | Atomicidade e repetição segura entre módulos |
| D11 | O `Idempotency-Key` é um **filtro da plataforma** para os três módulos; chaves naturais no domínio são a segunda barreira | Um comportamento só; cobre falha entre gravar e responder |
| D12 | **Valor com sinal** (saldos) é um tipo à parte do **valor monetário** (sempre positivo) | Preserva o pilar 1 e permite saldo negativo |
| D13 | A conta bancária aponta para uma conta financeira, no máximo uma para uma | Saldo e baixa vivem na conta financeira |
| D14 | Critérios de automação das camadas 2 e 3 em parâmetros próprios; regras de reconhecimento (cadastradas) separadas das regras aprendidas (só sugerem) | Aprendizado nunca cria lançamento |
| D15 | Documento da contraparte como índice cego, com a proteção de identificadores já existente | LGPD |
| D16 | Eventos de domínio com nomes em português e **nunca** com valores, descrições ou documentos | LGPD |
| R1 | **Conciliar e registrar a baixa acontecem na mesma transação** (chamada síncrona) | Evita a janela em que o vínculo está ativo e o lançamento ainda é editável; o cenário "reinicia antes da baixa" deixa de existir e é coberto por teste de atomicidade e repetição |

## 6. Qualidade, segurança e processo

| # | Decisão | Situação |
|---|---|---|
| Q-1 | Fluxo de trabalho **AI-DLC** com aprovação a cada etapa; toda entrada do usuário é registrada em `aidlc-docs/audit.md` | Decidida |
| Q-2 | **Testes de propriedade** (jqwik) para as regras que admitem propriedade (saldo, deduplicação, tetos, repetição) | Decidida |
| Q-3 | **Gates do build**: cobertura mínima de 80% em domínio e aplicação, SpotBugs com FindSecBugs, ArchUnit e verificação do Modulith | Decidida |
| Q-4 | **Regras de arquitetura no build**: domínio sem framework; fluxo de caixa não conhece a conciliação; conciliação só enxerga a API do fluxo de caixa; nenhuma porta com exclusão física | Decidida |
| Q-5 | **Sem PII em logs, métricas, eventos e erros**, com teste de integração que prova | Decidida |
| Q-6 | Arquivos de teste com extratos reais **só anonimizados**, com gate automático e `.gitignore` | Decidida |
| Q-7 | **Parser OFX próprio**, sem biblioteca, com proteção contra XXE | Decidida |
| Q-8 | Revisões por especialistas (arquitetura, LGPD, dinheiro, conformidade) a cada unidade | Decidida |

## 7. Decisões tomadas no fechamento da U3a (2026-10-03)

| # | Decisão |
|---|---|
| F1 | A **chave de deduplicação passa a ser versionada** (`v1`) e o FITID entra como hash |
| F2 | O resumo da importação deixa de guardar o andamento da conciliação; ele é **calculado na consulta** |
| F3 | O processamento da importação em três tempos: marcar, ler e interpretar o arquivo **sem trava**, concluir sob trava; conflito é repetido até 3 vezes |
| F4 | A porta de conclusão recebe **dados**, não agregados já alterados |
| F5 | Escrita em memória de outro tenant dentro de uma operação é recusada |
| F6 | **Log de acesso** passa a cobrir o detalhe de conta e a origem da transação para todos os papéis, e a tentativa de download antes de qualquer verificação |
| F7 | `X-Correlation-Id` só aceita hexadecimal e hífen |
| F8 | Limite de multipart para que nenhum extrato passe pelo disco sem criptografia |

## 7a. Decisões do ciclo 2 (U4 a U9, outubro de 2026)

O registro completo, com o porquê de cada uma, está em [Decisões e perguntas](../sdd/DECISOES-E-PERGUNTAS.md). As principais:

| Decisão | O que ficou decidido |
|---|---|
| Ordem das fases | Funcionalidades das Etapas 1 a 7 primeiro, com dados em memória; banco de dados (PostgreSQL) e Pluggy real por último, na Fase B |
| Titularidade PJ e PF | Toda conta tem titularidade; somas só PJ por padrão; contas PF visíveis só ao Admin; transferência PJ↔PF recusada |
| Conexão bancária | Porta do agregador com banco simulado agora; revisão ou cancelamento pela origem estorna a conciliação e guarda as duas versões (U5b) |
| Planilhas | Leitura própria e protegida de XLSX e CSV; nunca calcula fórmula; mais de 2 casas decimais é recusado |
| Importar lançamentos | Só o Admin confirma; nunca cria cadastro sozinho; linha paga entra com a baixa; lote desfeito tudo ou nada |
| Parcelas | 2 a 60; centavo que sobra na última; competência da operação na compra parcelada e mês a mês na repetição |
| Busca (D-49) | CNPJ alfanumérico aceito desde já; valor só quando escrito como dinheiro; limites por usuário no caso de uso e teto por empresa na rota |

## 8. Pendências de decisão

| # | O que falta | Quem decide |
|---|---|---|
| 1 | Confirmar **prazos de retenção** (ADR 011): 5 anos para ledger e trilha; extrato e descrições pela política da empresa | Negócio e jurídico |
| 2 | Nomear o **encarregado (DPO)** para pedidos de titulares | Negócio |
| 3 | **Validar o plano de contas semeado** (natureza da DRE e atividade da DFC) | Contador |
| 4 | **Validar os resumos normativos** contra o texto oficial das normas | Compliance e contador |
| 5 | **Obrigatoriedade do comprovante** na baixa manual | Contador |
| 6 | **Gatilho do alerta de liquidez** | Negócio |
| 7 | **OFX reais anonimizados** (2 ou 3) para validar o comportamento de FITID de cada banco | Usuário |
| 8 | Alvo de produção (por exemplo, AWS ECS e RDS) | Liderança técnica, na Infrastructure Design |
| 9 | Emendas formais às regras CB-16, CB-20, CB-55, CB-61 e FC-33 | Dono das regras |
| 10 | Confirmar a mudança de ordem (banco de dados por último) e a pergunta 19 | Matheus |
| 11 | Juros embutidos no parcelamento (Q-20) e imposto retido na importação (Q-18) | Contador |
| 12 | Estrutura definitiva da DRE e lista de relatórios gerenciais (bloqueiam a U13) | Contador |

## 9. Onde está cada coisa

| Assunto | Documento |
|---|---|
| Plano de atualização para as regras v2 e as 15 decisões | `aidlc-docs/inception/plans/atualizacao-rules-v2-plan.md` e `atualizacao-rules-v2-questions.md` |
| Requisitos | `aidlc-docs/inception/requirements/requirements.md` |
| Histórias de usuário | `aidlc-docs/inception/user-stories/stories.md` |
| Desenho e unidades | `aidlc-docs/inception/application-design/` |
| Fechamento da U3a | `aidlc-docs/inception/plans/u3a-fechamento-plan.md` |
| Registro de tudo o que foi pedido e respondido | `aidlc-docs/audit.md` |
| Estado atual | `aidlc-docs/aidlc-state.md` |
