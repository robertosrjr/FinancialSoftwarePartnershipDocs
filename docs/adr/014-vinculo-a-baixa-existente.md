# ADR 014 — Vínculo a uma baixa que já existe (M15)

- **Status**: Aceita · **Data**: 2026-10-06
- **Origem**: decisão do negócio de 06/10/2026 (M15: "se um lançamento já criou sua baixa e a mesma movimentação chegar depois, NÃO pode existir uma segunda baixa"); manifesto de confiança e precisão (saldo e ledger); ADR 012 (baixas imutáveis) e ADR 013 (ledger insert-only)

## Contexto
A conciliação só conhecia dois caminhos para uma transação do banco: alocar valor a um lançamento **em aberto** (e criar a baixa) ou criar um lançamento novo com baixa. A busca de candidatos só devolve lançamentos com valor em aberto. Quando o usuário já tinha registrado a baixa à mão, o lançamento estava quitado e **não aparecia como candidato**: a transação caía em "sem candidato", e o caminho natural, "criar lançamento da pendência", **duplicava o lançamento e o saldo**. O mesmo valia para a regra de reconhecimento de padrão, que cria lançamento e baixa sozinha. Isso já acontecia com o produto da Fase A, antes de qualquer importação nova.

## Decisão
1. **Novo tipo de vínculo, `BAIXA_EXISTENTE`**: liga a transação (inteira) a uma baixa **manual** ativa que já existe. Não cria baixa, não chama o fluxo de caixa para gravar nada e não muda o saldo. O vínculo guarda o lançamento e a baixa desde o nascimento. O evento do ledger é o mesmo da confirmação de alocação.
2. **Elegível** é a baixa de origem manual, não estornada, de lançamento não cancelado, da **mesma conta financeira e direção** da transação, de **valor exato** e, para o motor, dentro da tolerância de datas. A baixa criada pela conciliação nunca entra. O fluxo de caixa expõe isso em `buscarBaixasManuaisEmLote` e `buscarBaixaManual`, sem descrição, contraparte nem comprovante.
3. **O motor nunca decide sozinho nesse caso.** Havendo baixa compatível sem vínculo: uma só baixa e nenhum outro candidato gera uma **sugestão**; qualquer outra situação vai para a fila com o motivo `BAIXA_EXISTENTE_POSSIVEL`. O motor não vincula, não sugere lançamento, não cria lançamento por regra e não sugere lançamento novo. Valor, conta e data iguais não provam que é o mesmo pagamento (pode haver dois iguais no dia), então um humano confirma, como na regra de ouro. Cada baixa serve a uma só transação do lote. Ligação automática por chave exata (identificador do banco na baixa) **não foi implementada**.
4. **Uma baixa serve a uma só transação.** A verificação está no serviço e também no repositório de vínculos (`baixa_ja_vinculada`), para duas ligações simultâneas nunca ficarem abertas ou ativas. Na Fase B vira um índice único parcial.
5. **Desfazer o vínculo não estorna a baixa.** Ela nasceu antes e independente da conciliação e pode estar certa; a transação volta a pendente e a baixa volta a poder ser ligada. Em contraste, desfazer um vínculo comum estorna a baixa que a conciliação criou.
6. **"Criar lançamento da pendência" é guardado.** Com baixa compatível, responde 409 `baixa_existente_compativel`; só passa com um motivo informado de que é outro pagamento, e o motivo vai ao ledger (nunca a log). A confirmação de uma proposta de lançamento novo feita antes da baixa recusa o item pelo mesmo código. Vale também a baixa que o motor já sugeriu para a **mesma** transação.
7. **Só valor exato.** Diferença de valor não liga transação a baixa: cai em exceção para análise. Diferença de liquidação (juros, multa, desconto) ligada a baixa existente fica para depois, com regra própria.
8. **A segregação de funções não muda**: Admin com MFA liga na hora em revisor único; Operador, ou Admin em modo estrito, propõe, e outro Admin confirma. O vínculo fica fora do índice de reservas por lançamento, porque a baixa já está feita.

## Alternativas descartadas
- **Fazer a conciliação criar uma baixa "espelho"**: duplica saldo, o oposto do pedido.
- **Estender `LANCAMENTO` com uma bandeira "baixa já existe"**: mistura dois comportamentos de desfazer (estornar e não estornar) no mesmo tipo e esconde a diferença nos `switch`. O novo tipo faz o compilador apontar todo ponto que precisa decidir.
- **Ligar sozinho quando há um candidato só**: um pagamento igual no mesmo dia seria ligado errado sem ninguém ver.
- **Tolerância de valor**: amplia a mudança (o que fazer com a diferença) sem necessidade agora.

## Consequências
- (+) O pedido obrigatório do negócio está garantido por teste em três níveis: propriedade do motor, serviço e ponta a ponta com os módulos reais.
- (+) Corrige um risco que já existia, sem depender do banco de dados nem de nada que a Fase B refaça.
- (−) Ligação por chave exata ficou de fora; hoje toda ligação a baixa existente é confirmada por pessoa.
- (−) A consulta de baixas por valor percorre os lançamentos do tenant em memória; na Fase B precisa de índice por conta financeira, valor e data de pagamento.
- (−) Um motivo digitado para ignorar a baixa existente é texto livre e pode conter dado pessoal: fica só no ledger, como os demais motivos.
