# ADR 011 — Retenção e eliminação de dados pessoais da conciliação

- **Status**: Proposta (os prazos dependem de decisão do negócio e do jurídico; a implementação fica para a Fase B) · **Data**: 2026-10-03
- **Origem**: revisão de LGPD da U3a (achado alto), LGPD arts. 6º III, 15, 16 e 18 (IV e VI), NFR-31, ITG 2000 (R1), ADR 002 (crypto-shredding)

## Contexto
A conciliação guarda o arquivo original do extrato e a descrição de cada transação, e a descrição pode conter nome de pessoa. Hoje não existe prazo de retenção nem caminho de eliminação, e uma regra ArchUnit **proíbe** qualquer método de remoção nas portas de saída (`RepositorioDeContas`, `RepositorioDeImportacoes`, `RepositorioDeTransacoes`, `ArmazenamentoDeExtratos`). A proibição existe por boa razão (ITG 2000 R1: registro contábil não se apaga, só se estorna), mas ela hoje também impede a eliminação do que é dado pessoal e não registro contábil. Um pedido de titular (art. 18, VI) ou o fim do tratamento (arts. 15 e 16) não teria resposta técnica. Na Fase C, o Object Lock do armazenamento (WORM) agrava o problema se o prazo de retenção for infinito.

## Decisão (proposta)

### 1. Classes de dado, base legal e prazo

| Classe | Exemplos | Base legal | Prazo proposto | Fim do prazo |
|---|---|---|---|---|
| Registro contábil e trilha | ledger de conciliação, baixas, estornos, trilha de auditoria (conteúdo cifrado) | Obrigação legal (art. 16, I; ITG 2000 R1) | **5 anos** a partir do fato | Eliminação por crypto-shredding da chave do tenant para o conteúdo; o hash da cadeia permanece |
| Suporte documental | arquivo OFX original | Obrigação legal e exercício regular de direitos | **5 anos**, ou a política do tenant se maior | Expiração do objeto (retenção finita no Object Lock), não exclusão manual |
| Descrição da transação | texto livre do extrato | Obrigação legal enquanto houver o registro; depois, nenhuma | **Igual ao suporte documental** | **Anonimização**: a descrição é substituída pela descrição normalizada sem nomes ou por marcador, preservando valor, data, direção e vínculos |
| Identificadores de conta | índices cegos e 4 últimos dígitos | Legítimo interesse e execução do contrato | Enquanto a conta existir | Conta inativada: índices removidos em 90 dias após o último uso |
| Telemetria e logs | métricas, logs sem PII | Legítimo interesse | 30 a 90 dias | Expiração do armazenamento de logs |

### 2. Ciclo de vida sem violar a imutabilidade
- Cria-se uma porta própria, **`PoliticaDeRetencao`** (`anonimizar`, `expirar`), separada dos repositórios. A regra ArchUnit passa a ter uma **exceção explícita e única** para essa porta; as portas de repositório continuam sem método de exclusão.
- A anonimização **não apaga a transação**: troca só o campo de texto livre. Valor, data, direção, vínculos e baixas permanecem, o que preserva DRE, DFC e conciliação.
- O arquivo original some por **expiração**: Object Lock com retenção **finita**, igual ao prazo da classe. Nunca por `DELETE` manual.
- O expurgo é um job agendado, por tenant, que registra na trilha (como sistema) quantos itens anonimizou ou expirou, sem conteúdo.
- Pedido de titular (art. 18): procedimento operacional que localiza as transações pelo documento da contraparte (índice cego), avalia se há base legal para manter e, se não houver, anonimiza. A resposta registra a decisão.

### 3. O que muda no código, e quando
- **Agora (U3a)**: nada de código. Esta ADR registra a decisão e o prazo a validar. O limite do multipart, a máscara no `toString` e o gate das fixtures já reduzem a superfície.
- **Fase B**: coluna de data de retenção por classe, `PoliticaDeRetencao`, job de anonimização e exceção na regra ArchUnit, com testes de propriedade (anonimizar não altera saldos, totais nem vínculos).
- **Fase C**: Object Lock com retenção finita por classe e rotina de expiração.

## Pendências de decisão
1. **Prazos**: confirmar os 5 anos com o jurídico e o contador. Se o tenant (ex.: BPO) quiser prazo maior, a política passa a ser configurável por tenant, nunca menor que o legal.
2. **Texto livre**: confirmar que a anonimização da descrição (e não a eliminação da transação) atende ao art. 16 para a base legal escolhida.
3. **Encarregado (DPO)**: nomear o responsável pelo atendimento de pedidos de titular.

## Consequências
- (+) A imutabilidade contábil e a eliminação de dado pessoal deixam de se contradizer: o registro fica, o dado pessoal sai.
- (+) A Fase B nasce com um desenho de ciclo de vida, em vez de descobrir a lacuna com dados reais.
- (−) A anonimização é uma mutação controlada em dado que hoje é insert-only: exige exceção estreita na regra ArchUnit e testes próprios.
- (−) Até a Fase B, o sistema não tem eliminação. Isso é aceitável enquanto a persistência é em memória e sem dado real, e deixa de ser antes de qualquer cliente usar o produto.

## Nota (2026-10-10, U9)
- A U9 guarda, no repositório, a forma normalizada da descrição e da contraparte (lançamento e transação) para a busca. Ela é dado pessoal derivado: a porta `PoliticaDeRetencao`, quando existir, DEVE recalcular essa entrada ao anonimizar ou eliminar (na Fase B, coluna gerada que acompanha o original).
- Lacuna: esta ADR trata da conciliação; a **contraparte e a descrição do lançamento** (fluxo de caixa) também precisam de classe, prazo e caminho de eliminação. A decidir junto com os prazos desta proposta.
