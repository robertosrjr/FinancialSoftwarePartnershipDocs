# ADR 013 — Vínculo, ledger insert-only e identificador determinístico na conciliação

- **Status**: Aceita · **Data**: 2026-10-04
- **Origem**: U3b (motor, vínculos, exceções e consultas); `docs/sdd/rules/REGRAS-CONCILIACAO-BANCARIA.md` (CB-06, CB-30, CB-31, CB-80, CB-81); manifesto de confiança e precisão (ledger imutável); [ADR 012](012-baixas-imutaveis-situacao-derivada-e-travas.md)

## Contexto
A conciliação liga valores de transações do extrato a lançamentos do fluxo de caixa. Uma transação pode ser repartida entre vários lançamentos e um lançamento pode receber várias transações; confirmar cria uma baixa em **outro módulo**; várias pessoas e o motor escrevem ao mesmo tempo; e uma operação pode parar no meio (falha de auditoria, de ledger ou do fluxo de caixa depois de a baixa existir). Num sistema financeiro, o histórico não pode ser reescrito e a repetição de uma operação não pode duplicar dinheiro.

## Decisão
1. **O ledger é insert-only.** `EventoDoLedger` nunca muda; a porta só tem `anexar` e consultas (regra ArchUnit no build). Desfazer é um evento novo (`ALOCACAO_ESTORNADA`) que aponta o evento de confirmação, que permanece. A fila de exceções segue o mesmo princípio (só anexar): a exceção está aberta enquanto a transação está em `EXCECAO`.
2. **O estado da transação é derivado, nunca gravado.** `PENDENTE`, `PARCIAL`, `CONCILIADA`, `EXCECAO` e `IGNORADA` saem da soma dos vínculos ativos, do último evento entre ignorar e reativar e da presença na fila. Sugestões e propostas não consomem saldo livre; só o vínculo `ATIVO` consome. O estado de cada vínculo é reconstruível reaplicando os eventos do ledger (propriedade testada).
3. **O vínculo tem identificador determinístico**: hash de tenant, transação, destino, valor e ordinal dos vínculos anteriores da mesma transação para o mesmo destino. Esse identificador é também o `alocacaoId` enviado ao fluxo de caixa. Se qualquer passo depois da baixa falhar, repetir a operação chega ao mesmo identificador, o fluxo de caixa devolve a mesma baixa e o restante se completa, sem baixa duplicada.
4. **Ordem das gravações**: fluxo de caixa, auditoria, ledger, vínculo. O ledger é idempotente por (vínculo, tipo de evento). Se o fluxo de caixa falha, nada é gravado na conciliação. Em grupos (transferência com par, par de estorno), o vínculo da **outra ponta** é gravado **antes** do principal e tem identificador derivado dele, para a repetição convergir.
5. **Dois tetos**: a alocação não passa do saldo livre da transação (conferido sob a trava da transação) nem do valor em aberto do lançamento (decidido, de forma atômica, pelo fluxo de caixa). A conciliação envia sempre `baixaFinal = false`: o fluxo de caixa fecha o lançamento quando o valor em aberto zera.
6. **Travas**: conta bancária (só o motor e a importação), depois transação (várias, em ordem de identificador), depois as travas do fluxo de caixa. O fluxo de caixa nunca chama a conciliação, então não há ciclo.
7. **O motor é puro** (`MotorDeConciliacao`, sem I/O): toda a decisão vive nele, o `ExecutorDoMotor` só traduz entradas em chamadas. As propriedades de segurança (nunca automático com mais de um candidato, só com prova, nunca com saldo não conferido, reserva no lote) são testadas com milhares de casos gerados.
8. **Segregação de funções por vínculo**: o estado inicial depende do papel e do modo do tenant (Operador e Admin em modo estrito propõem; Admin em revisor único concilia na hora, com marca e auditoria); a confirmação pergunta à política antes de qualquer efeito no fluxo de caixa; os tetos são revalidados na confirmação.
9. **Privacidade**: descrição, contraparte, documento e o motivo digitado nunca vão à trilha, aos eventos, às métricas nem aos spans; o motivo fica só no ledger; o documento extraído da descrição vira índice cego mais os quatro últimos dígitos.

## Consequências
- (+) Qualquer saldo e qualquer estado podem ser reconstruídos a partir de fatos; a auditoria confere sem confiar em campo calculado.
- (+) A repetição depois de falha parcial é segura por construção, o que permite reprocessar e repetir com `Idempotency-Key` sem medo de duplicar baixa.
- (+) O motor é testável como função, e o comportamento automático tem prova (zero falso positivo é uma propriedade, não uma esperança).
- (−) O estado custa um cálculo a cada leitura; a leitura em massa depende de índices (em memória, um índice por data e identificador; no PostgreSQL, agregação SQL com índices). As metas de 500 ms com 50 mil transações foram medidas em memória e precisam do teste de carga da Fase B (RK-U3b-1).
- (−) Desfazer um grupo grava os membros em sequência: se a gravação parar entre eles, repetir pelo membro ainda ativo conclui, e o já desfeito responde `vinculo_nao_ativo`. Na Fase B, o grupo passa a uma única transação de banco.
- (−) As travas em memória valem para uma instância; na Fase B passam para o banco (os testes de concorrência da Fase A viram o contrato dessa troca).
