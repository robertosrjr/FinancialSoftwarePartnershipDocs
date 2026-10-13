# Agente de Bolso

**Gestão financeira para empresas de serviços, com um caixa em que dá para confiar.**

O Agente de Bolso junta **fluxo de caixa** e **conciliação bancária** num núcleo de serviços preciso e auditável. O empresário manda o extrato do banco, o sistema cruza com o que ele previu pagar e receber, e mostra quanto tem e quanto vai ter, sem planilha e sem digitar tudo de novo.

## Para quem

Micro e pequenas empresas, MEIs, consultorias e **escritórios de BPO financeiro** que prestam serviços.

## O problema

- O extrato do banco e o controle financeiro nunca batem, e conferir toma horas todo mês.
- Reimportar um extrato duplica lançamentos; transferências entre contas próprias viram receita.
- Ninguém sabe dizer quem alterou o quê, nem provar ao contador que nada foi mexido.
- O caixa projetado esconde o que está atrasado.

## O que o produto entrega

| Valor | Como |
|---|---|
| **Menos trabalho** | Importa o extrato OFX, reconhece o que já foi importado, concilia sozinho só o que é seguro e pede decisão no resto |
| **Números confiáveis** | Valores em centavos exatos, saldo conferido a cada extrato, três datas por lançamento (vencimento, competência, liquidação), atrasos em destaque |
| **Auditoria de verdade** | Nada é apagado: erro se corrige com estorno. Toda ação fica numa trilha com selo de integridade que qualquer auditor pode verificar |
| **Trabalho em equipe seguro** | Quem propõe não aprova. Operador prepara, Admin confirma, e o sistema registra cada exceção |
| **Visão para BPO** | Um acesso para consultar várias empresas clientes, somente leitura |
| **Privacidade** | O número da conta bancária nem é guardado; logs e eventos não carregam dados pessoais |

## Em que pé está

| Capacidade | Situação |
|---|---|
| Acesso por empresa, papéis, autenticação forte, trilha de auditoria, segregação de funções | Pronto |
| Lançamentos e plano de contas para serviços | Pronto |
| Cadastro de contas bancárias e importação de extrato OFX, sem duplicar, com conferência de saldo | Pronto |
| Reenvio seguro de operações (idempotência) | Pronto |
| Contas financeiras, baixas e pagamento parcial, transferências, saldo e projeção de caixa | Pronto |
| Conciliação automática em três camadas, fila de exceções, confirmar e desfazer com estorno, transferência entre contas, regras de reconhecimento, pendências, situação e histórico | Pronto |
| Ligar a transação do extrato à **baixa que já foi registrada à mão**, sem duplicar baixa nem saldo | Pronto (branch `baixa-existente`, aguardando merge) |
| Centro de custo, forma de pagamento e conta PJ ou PF com visibilidade restrita (U4) | Pronto |
| Entrada genérica de extrato, conexão bancária com banco simulado e estorno automático quando o banco muda ou cancela (U5, U5b) | Pronto |
| Importação de extrato e de contas a pagar e a receber por planilha XLSX e CSV, com revisão, duplicidade e desfazer (U6, U7) | Pronto |
| Parcelamentos e repetições (U8) | Pronto |
| Busca, seleção e totalizador (U9) | Em desenho (Functional Design e NFR aprovados) |
| Edição e baixa em massa, exportação, categorização por histórico, DRE e relatórios (U10 a U13) | A fazer |
| Banco de dados (PostgreSQL), Pluggy real, pagamentos pelo banco, WhatsApp, painel web e inteligência artificial externa | Depois (sem telas nesta fase: a entrega é a API) |

Hoje o produto é um **serviço de API**; os canais virão sobre a mesma base (Service-First). "Pronto" significa implementado e testado com os dados **em memória** (Fase A): a persistência em PostgreSQL e a implantação na AWS são as próximas fases, e antes delas faltam a varredura de dependências e o CodeQL no CI, a validação do contador sobre o plano de contas e OFX reais anonimizados.

## Princípios

1. **Precisão**: todo cálculo é do sistema, nunca de IA; dinheiro é inteiro em centavos.
2. **Imutabilidade**: nenhum registro financeiro é apagado ou sobrescrito.
3. **Rigor contábil**: regimes de competência e de caixa separados; impostos sobre faturamento como dedução da receita.
4. **Banco conferido**: saldo fechado e deduplicação estrita antes de confiar num extrato.

Detalhe no [Manifesto de Confiança e Precisão](docs/sdd/rules/MANIFESTO-CONFIANCA-E-PRECISAO.md).

## Documentação

| Para | Leia |
|---|---|
| **Acompanhar o projeto**: o que está pronto, o que vem em cada etapa e o que falta decidir (atualizado a cada marco) | [Acompanhamento do Projeto](docs/manuais/ACOMPANHAMENTO-DO-PROJETO.md) |
| **Resumo numa página e as perguntas ainda abertas** (para enviar ao negócio) | [Resumo e perguntas para o Matheus](docs/brainstorming/Matheus%20dia%2002/RESUMO-E-PERGUNTAS-PARA-O-MATHEUS-2026-10-06.md) |
| Ver **numa tabela o que cada funcionalidade pede, faz e entrega** | [Guia Rápido](docs/manuais/GUIA-RAPIDO.md) |
| Ver **a lista simples de funcionalidades, o que é e a regra** | [Funcionalidades](docs/manuais/FUNCIONALIDADES.md) |
| **Validar o produto com os sócios** (roteiro, resultado esperado e marcação de aceite compartilhada) | [Validação do Agente de Bolso](https://claude.ai/artifact/7MgmPXip1AAGJTB1sAwW7A) |
| Ver **como o produto se organiza nas camadas de serviço do TOGAF** (ofertas, serviços de negócio, de aplicação e de tecnologia, com a rastreabilidade por unidade) | [Arquitetura de serviços (TOGAF)](docs/togaf/README.md) |
| Ver **como o produto trata segurança, OWASP, LGPD e pentest** (e o que falta) | [Segurança e Privacidade](docs/manuais/SEGURANCA.md) |
| Entender **o que o produto faz e como usar** | [Manual do Usuário](docs/manuais/MANUAL-DO-USUARIO.md): [Fluxo de Caixa](docs/manuais/MANUAL-FLUXO-DE-CAIXA.md) e [Conciliação Bancária](docs/manuais/MANUAL-CONCILIACAO-BANCARIA.md) |
| Conhecer **as regras de negócio** | [Regras de Negócio](docs/sdd/REGRAS-DE-NEGOCIO.md) e as regras oficiais em [`docs/sdd/rules/`](docs/sdd/rules/) |
| Saber **o que foi decidido e por quê** | [Decisões do Projeto](docs/manuais/DECISOES-DO-PROJETO.md) e os [ADRs](docs/adr/README.md) |
| Entender **como funciona por dentro** | [Jornada Técnica](docs/jornadas/JORNADA-TECNICA.md) (plataforma), [Fluxo de Caixa](docs/jornadas/JORNADA-TECNICA-FLUXO-DE-CAIXA.md) e [Conciliação](docs/jornadas/JORNADA-TECNICA-CONCILIACAO.md), com diagramas de sequência |
| **Subir a aplicação, chamar a API e rodar os testes** | [Manual Técnico](docs/manuais/MANUAL-TECNICO.md) e [application/README.md](application/README.md) |
| Saber **o que é testado**, por domínio | [Cenários de teste](docs/testes/README.md) |
| Ver a **visão de produto** | [Especificação de Produto](docs/geral/PRODUCT-SPEC-v4.md) e a [Visão 2027](docs/brainstorming/VISAO-DISRUPTIVA-2027.md) |
| Ver as **conversas e decisões com o negócio** (respostas do Matheus, avaliação, consolidado e perguntas) | [`docs/brainstorming/`](docs/brainstorming/): [decisões e perguntas](docs/sdd/DECISOES-E-PERGUNTAS.md), [consolidado para o Matheus](docs/brainstorming/Matheus%20dia%2002/olds/CONSOLIDADO-PARA-O-MATHEUS-2026-10-06-v2.md) e [perguntas ao Matheus](docs/brainstorming/Matheus%20dia%2002/olds/PERGUNTAS-PARA-O-MATHEUS-2026-10-06.md) |
| Responder as **perguntas de requisitos do ciclo 2** (AI-DLC) | [Perguntas de requisitos](aidlc-docs/inception/requirements/ciclo-2-requirement-verification-questions.md) |
| Ler os **requisitos do ciclo 2** (Etapas 1 a 7, sem banco e sem frontend) | [Requisitos do ciclo 2](aidlc-docs/inception/requirements/requirements-ciclo-2.md) |
| Ver o **plano de execução do ciclo 2** (unidades U4 a U13, módulos e ordem) | [Plano de execução do ciclo 2](aidlc-docs/inception/plans/execution-plan-ciclo-2.md) |
| Revisar o **desenho do ciclo 2** (Application Design: módulos `importacao` e `relatorios`, portas, fluxos, dependências) | [Plano e respostas](aidlc-docs/inception/plans/application-design-ciclo-2-plan.md), [Componentes](aidlc-docs/inception/application-design/components.md#8-ciclo-2-2026-10-06-componentes-novos-e-alterados), [Métodos](aidlc-docs/inception/application-design/component-methods.md), [Serviços](aidlc-docs/inception/application-design/services.md), [Dependências](aidlc-docs/inception/application-design/component-dependency.md) |
| Responder as **perguntas do Functional Design da U5** (entrada canônica e porta do agregador) | [Plano da U5](aidlc-docs/construction/plans/u5-entrada-canonica-e-agregador-functional-design-plan.md) |
| Revisar o **código da U4** (resumo, desvios e limitações) | [Resumo do código da U4](aidlc-docs/construction/u4-cadastros-e-titularidade/code/code-summary.md) |
| **U9** (busca, seleção e totalizador): código aprovado (11/10/2026) | [Resumo do código](aidlc-docs/construction/u9-busca-selecao-e-totalizador/code/code-summary.md), [Padrões](aidlc-docs/construction/u9-busca-selecao-e-totalizador/nfr-design/nfr-design-patterns.md), [Componentes](aidlc-docs/construction/u9-busca-selecao-e-totalizador/nfr-design/logical-components.md), [Plano de NFR Design](aidlc-docs/construction/plans/u9-busca-selecao-e-totalizador-nfr-design-plan.md),  [NFR](aidlc-docs/construction/u9-busca-selecao-e-totalizador/nfr-requirements/nfr-requirements.md), [Tecnologia](aidlc-docs/construction/u9-busca-selecao-e-totalizador/nfr-requirements/tech-stack-decisions.md), [Plano de NFR](aidlc-docs/construction/plans/u9-busca-selecao-e-totalizador-nfr-requirements-plan.md),  [Plano e respostas](aidlc-docs/construction/plans/u9-busca-selecao-e-totalizador-functional-design-plan.md), [Entidades](aidlc-docs/construction/u9-busca-selecao-e-totalizador/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u9-busca-selecao-e-totalizador/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u9-busca-selecao-e-totalizador/functional-design/business-logic-model.md), [Histórias](aidlc-docs/inception/user-stories/stories-ciclo-2.md), [D-49](docs/sdd/DECISOES-E-PERGUNTAS.md), [ADR 005](docs/adr/005-indice-cego-numero-de-conta-nao-armazenado.md) |
| **U8 concluída** (parcelas e repetições; código e Build and Test aprovados em 10/10) | [Resumo do Build and Test](aidlc-docs/construction/build-and-test/build-and-test-summary.md),  [Resumo do código](aidlc-docs/construction/u8-parcelamentos-e-repeticoes/code/code-summary.md), [Lógica e contrato](aidlc-docs/construction/u8-parcelamentos-e-repeticoes/functional-design/business-logic-model.md) | [Plano e respostas](aidlc-docs/construction/plans/u8-parcelamentos-e-repeticoes-functional-design-plan.md), [Entidades](aidlc-docs/construction/u8-parcelamentos-e-repeticoes/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u8-parcelamentos-e-repeticoes/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u8-parcelamentos-e-repeticoes/functional-design/business-logic-model.md), [Histórias](aidlc-docs/inception/user-stories/stories-ciclo-2.md) |
| **U7 concluída** (código e Build and Test aprovados em 10/10) | [Resumo do Build and Test](aidlc-docs/construction/build-and-test/build-and-test-summary.md), [Resumo do código](aidlc-docs/construction/u7-importacao-de-lancamentos/code/code-summary.md),  [Plano e respostas](aidlc-docs/construction/plans/u7-importacao-de-lancamentos-functional-design-plan.md), [Entidades](aidlc-docs/construction/u7-importacao-de-lancamentos/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u7-importacao-de-lancamentos/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u7-importacao-de-lancamentos/functional-design/business-logic-model.md), [Histórias](aidlc-docs/inception/user-stories/stories-ciclo-2.md) |
| **U5b concluída** (código e Build and Test aprovados em 10/10) | [Resumo do código](aidlc-docs/construction/u5b-revisao-e-cancelamento-pela-origem/code/code-summary.md), [Resumo do Build and Test](aidlc-docs/construction/build-and-test/build-and-test-summary.md), [Instruções](aidlc-docs/construction/build-and-test/build-and-test-instructions.md) |
| Revisar o **Functional Design da U5b** (estorno automático quando o banco revisa ou cancela) | [Plano e respostas](aidlc-docs/construction/plans/u5b-revisao-e-cancelamento-pela-origem-functional-design-plan.md), [Entidades](aidlc-docs/construction/u5b-revisao-e-cancelamento-pela-origem/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u5b-revisao-e-cancelamento-pela-origem/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u5b-revisao-e-cancelamento-pela-origem/functional-design/business-logic-model.md), [Histórias](aidlc-docs/inception/user-stories/stories-ciclo-2.md) |
| Ler a **ADR 015** (OFX permanece na conciliação; revisão pela origem só no agregador) | [ADR 015](docs/adr/015-ofx-permanece-adaptador-de-entrada-da-conciliacao.md) |
| Revisar o **plano da U6b** (arquivado) (mover o OFX para o módulo `importacao`; história, estado atual, proposta e perguntas ao revisor) | [Plano da U6b](aidlc-docs/construction/plans/u6b-ofx-no-modulo-importacao-plan.md) |
| Revisar o **código da U6** (importação de planilha, duplicidade e desfazer lote: resumo, desvios e limitações) | [Resumo do código da U6](aidlc-docs/construction/u6-importacao-flexivel-de-extrato/code/code-summary.md), [Funcionalidades](docs/manuais/FUNCIONALIDADES.md), [Acompanhamento](docs/manuais/ACOMPANHAMENTO-DO-PROJETO.md) |
| Revisar o **NFR Requirements da U6** (segurança do arquivo, limites, modelo de ameaças) | [Plano e respostas](aidlc-docs/construction/plans/u6-importacao-flexivel-de-extrato-nfr-requirements-plan.md), [NFR](aidlc-docs/construction/u6-importacao-flexivel-de-extrato/nfr-requirements/nfr-requirements.md), [Decisões de stack](aidlc-docs/construction/u6-importacao-flexivel-de-extrato/nfr-requirements/tech-stack-decisions.md) |
| Revisar o **Functional Design da U6** (importação de planilha) | [Plano e respostas](aidlc-docs/construction/plans/u6-importacao-flexivel-de-extrato-functional-design-plan.md), [Histórias do ciclo 2](aidlc-docs/inception/user-stories/stories-ciclo-2.md), [Entidades](aidlc-docs/construction/u6-importacao-flexivel-de-extrato/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u6-importacao-flexivel-de-extrato/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u6-importacao-flexivel-de-extrato/functional-design/business-logic-model.md) |
| Revisar o **código da U5** (entrada canônica, conexão bancária, desvios e limitações) | [Resumo do código da U5](aidlc-docs/construction/u5-entrada-canonica-e-agregador/code/code-summary.md) |
| Revisar o **Functional Design da U5** (entrada canônica e conexão bancária) | [Plano e respostas](aidlc-docs/construction/plans/u5-entrada-canonica-e-agregador-functional-design-plan.md), [Histórias do ciclo 2](aidlc-docs/inception/user-stories/stories-ciclo-2.md), [Entidades](aidlc-docs/construction/u5-entrada-canonica-e-agregador/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u5-entrada-canonica-e-agregador/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u5-entrada-canonica-e-agregador/functional-design/business-logic-model.md) |
| Revisar o **Functional Design da U4** (cadastros e titularidade da conta) | [Plano e respostas](aidlc-docs/construction/plans/u4-cadastros-e-titularidade-functional-design-plan.md), [Histórias do ciclo 2](aidlc-docs/inception/user-stories/stories-ciclo-2.md), [Entidades](aidlc-docs/construction/u4-cadastros-e-titularidade/functional-design/domain-entities.md), [Regras](aidlc-docs/construction/u4-cadastros-e-titularidade/functional-design/business-rules.md), [Lógica e contrato](aidlc-docs/construction/u4-cadastros-e-titularidade/functional-design/business-logic-model.md) |
| Revisar as **unidades do ciclo 2** (U4 a U13, ordem, dependências e blocos de teste) | [Unidades](aidlc-docs/inception/application-design/unit-of-work.md), [Dependências entre unidades](aidlc-docs/inception/application-design/unit-of-work-dependency.md), [Requisitos por unidade](aidlc-docs/inception/application-design/unit-of-work-story-map.md) |
| Acompanhar o **processo de construção** | [`aidlc-docs/`](aidlc-docs/): requisitos, histórias, desenho, estado e registro de decisões |

## Stack

Java 25, Spring Boot 4, Spring Modulith, arquitetura hexagonal, monolito modular. Persistência em memória nesta fase; PostgreSQL com isolamento por empresa e AWS nas fases seguintes.
