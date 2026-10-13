# ADR 012 — Baixas imutáveis, situação derivada e travas por lançamento e por conta

- **Status**: Aceita · **Data**: 2026-10-03
- **Origem**: U2b (contas, baixas e transferências); regras FC-nn de `docs/sdd/rules/REGRAS-FLUXO-DE-CAIXA.md`; manifesto de confiança e precisão (centavos, ledger imutável, tripla de datas, fecho de saldo)

## Contexto
O fluxo de caixa passa a registrar pagamentos e recebimentos reais (baixas), parciais ou totais, e transferências entre contas próprias. Um sistema financeiro não pode ter um valor que "muda de mão": o saldo de uma conta é a soma de fatos que aconteceram, e a correção de um erro é um novo fato que o compensa (ITG 2000 R1). Ao mesmo tempo, vários usuários e o módulo de conciliação escrevem ao mesmo tempo no mesmo lançamento e nas mesmas contas.

## Decisão
1. **Baixa é imutável.** Uma baixa nunca é alterada nem apagada. `PUT` e `PATCH` em baixa respondem 409 `baixa_imutavel`. A correção é o **estorno compensatório** (`EstornoDeBaixa`), que aponta a baixa original, leva motivo e autor, e só pode acontecer uma vez por baixa. As portas de saída não têm método de atualização nem de exclusão (regra ArchUnit no build).
2. **A situação do lançamento é derivada, nunca gravada.** `PREVISTO`, `PARCIAL`, `PAGO`, `VENCIDO` e `CANCELADO` saem de três coisas: o valor previsto, a soma das baixas ativas e a data de hoje. Não existe coluna nem campo de situação para ficar defasado depois de um estorno. Na Fase B, a situação é calculada na consulta.
3. **A diferença de liquidação é derivada.** Juros, multa, desconto e tarifa são a diferença entre o previsto e o baixado, mapeada para quatro categorias de sistema; não é um lançamento gravado à parte.
4. **Transferência entre contas próprias** é um fato único com origem, destino e valor, estornável como um todo. Não é um par de lançamentos, então não entra na receita nem na despesa da DRE.
5. **Contas inativas têm saldo zero** (invariante). Inativar com saldo diferente de zero é recusado.
6. **Travas em duas camadas**, em memória na Fase A e por linha/consulta com `SELECT ... FOR UPDATE` na Fase B:
   - por **lançamento** (faixas de `ReentrantLock` por tenant), em toda escrita que mexe em baixa, estorno ou cancelamento;
   - por **conta financeira** (`ReentrantReadWriteLock` por tenant): **compartilhada** em baixa e transferência (várias podem correr juntas) e **exclusiva** ao inativar (ninguém baixa numa conta enquanto ela é conferida e desligada). Quando uma operação toca duas contas, a ordem de aquisição é pelo id, para não haver impasse.
7. **Idempotência em dois níveis.** Na API REST, pelo `Idempotency-Key` ([ADR 008](008-idempotencia-de-escrita.md)). Entre módulos, pelo `alocacaoId` da conciliação: repetir a mesma operação devolve o mesmo resultado; o mesmo `alocacaoId` com conteúdo diferente é 409 `alocacao_conflitante`.
8. **Privacidade.** O documento da contraparte guarda só um **índice cego** (HMAC, finalidade `DOCUMENTO_CONTRAPARTE`) e uma máscara. A trilha de auditoria, os eventos e as métricas nunca levam descrição, contraparte, documento nem comprovante.
9. **Conta bancária da conciliação ligada à conta financeira (U17).** Toda conta bancária da conciliação aponta para uma conta financeira do tipo `BANCARIA`, ativa, e cada conta financeira tem no máximo uma conta bancária (409 `conta_financeira_ja_ligada`). A conciliação valida isso só pelo contrato publicado, `FluxoCaixaApi.contaFinanceira(id)`.

## Consequências
- (+) O saldo e o extrato de uma conta podem ser reconstruídos a qualquer momento a partir dos fatos; auditoria externa confere a conta sem confiar em campo calculado.
- (+) Não há como a situação divergir das baixas: não existe onde divergir.
- (+) Concorrência previsível: dois estornos da mesma baixa, ou uma baixa e uma inativação, nunca produzem saldo errado; o teste de concorrência com threads virtuais cobre os dois casos.
- (−) A situação custa um cálculo a cada leitura. Aceito: o cálculo é por lançamento, e as listagens e o saldo usam consultas agregadas (meta de 500 ms no p95 com 50 mil baixas, testada em memória).
- (−) A trava em memória vale para uma instância. Na Fase B, a trava passa para o banco; os testes de concorrência da Fase A viram o contrato dessa troca.
- (−) O plano de 20 categorias é **provisório** até a validação do contador (BR-CG-09). Isso não bloqueia o código; muda dados de configuração, não o modelo.
