# ADR 015 — OFX permanece adaptador de entrada da conciliação; revisão pela origem só no agregador

- **Status**: Aceita · **Data**: 2026-10-09
- **Origem**: plano da U6b (`aidlc-docs/construction/plans/u6b-ofx-no-modulo-importacao-plan.md`, seção 15), CB-15, CB-19, CB-22, ADR 006

## Contexto
Hoje há dois módulos que leem arquivo de extrato:
- **`conciliacao`** lê o OFX. Foi o primeiro formato (U3a) e é o único que entra lendo o arquivo dentro da conciliação.
- **`importacao`** lê planilhas XLSX e CSV (U6) e entrega linhas canônicas pela `EntradaDeExtrato` (U5).

Avaliou-se mover o OFX para `importacao`, para que um só módulo lesse todos os formatos. A revisão do plano, feita com leitura do código, mostrou três coisas:
1. Não há regra de negócio duplicada. `ProcessamentoDaEntradaCanonica` faz uma triagem e depois chama o próprio `ProcessamentoDoExtrato`. A duplicação real é só o trecho que lê o arquivo guardado em `ServicoDeProcessamento.lerEInterpretar`.
2. A triagem da entrada canônica não olhava a origem. Um identificador já registrado com outro conteúdo virava **revisão pela origem** (fila de exceções e saldo `SEM_CONFERENCIA`) em qualquer origem. Isso contraria a CB-19: "em arquivo, um identificador do banco reutilizado com conteúdo diferente é tratado como transação nova, com aviso, e não como revisão". Também contraria a CB-22, que diz "para arquivos, vale CB-19". Passar o OFX pela entrada canônica teria levado esse defeito ao OFX. A planilha da U6 **já tinha esse defeito** quando a coluna de identificador era mapeada.
3. O caminho canônico fixa a moeda em BRL. O aviso `moeda_assumida_brl` do OFX sumiria.

## Decisão
1. **O OFX permanece como adaptador de entrada da conciliação** (`conciliacao/infrastructure/ofx`, endpoint `POST /api/v1/conciliacao/importacoes`). A assimetria com as planilhas é consciente, não acidental.
2. **Revisão e cancelamento pela origem valem só para a família `AGREGADOR:<fonte>`** (`FamiliaDeOrigem.temSemanticaDeRevisao`), a única origem com contrato de que o mesmo identificador é a mesma transação. Na triagem da entrada canônica:

| Parte da triagem | Regra | Vale para |
|---|---|---|
| `PENDENTE` nunca vira transação | CB-15 | Todas as origens |
| `CANCELADA` de transação já registrada vira ocorrência na fila | CB-22 | Só o agregador; nas outras origens a linha é só contada |
| Identificador existente com outro conteúdo vira `REVISADA` | CB-22 | Só o agregador; nas outras origens segue para o `Deduplicador` e entra como transação nova com aviso `fitid_inconsistente` (CB-19, ADR 006) |

A `ENTRADA_DIRETA` (linhas pela API) fica com o comportamento de arquivo: quem envia pela API não garante que o mesmo identificador seja a mesma transação. **A confirmar com o Matheus**, como pergunta do resumo dele, com a sugestão de seguir a CB-19.

## Gatilhos para reabrir a decisão 1
Basta um deles:
- um quarto formato de arquivo (CNAB, por exemplo);
- o OFX precisar de prévia ou de revisão antes de entrar;
- o redesenho da persistência na Fase B (antes de fixar o esquema);
- o endpoint duplicado começar a confundir o cliente da API.

Se reabrir, o plano da U6b precisa mudar em quatro pontos:
- a entrada canônica carrega a moeda informada, ou a ausência dela;
- o reprocessamento relê o arquivo original, não o canônico;
- a conta é identificada por índices cegos calculados no módulo de importação, com uma só implementação publicada;
- a Fase 6 (unificar processadores) sai do plano, porque já são um só.

## Consequências
- (+) Corrige um defeito real da U6: planilha reenviada com o mesmo identificador e outro valor não suspende mais a conferência nem vai para a fila como revisão.
- (+) O OFX continua com o comportamento testado desde a U3a, sem risco de regressão por uma migração sem dor presente.
- (+) A regra fica num lugar só (`FamiliaDeOrigem`), com teste por origem: arquivo, entrada direta e agregador.
- (−) Dois módulos leem arquivo de extrato, e há dois prefixos de URL para "enviar extrato". Isso está documentado aqui, com gatilhos.
- (−) A `ENTRADA_DIRETA` perdeu a revisão pela origem que tinha desde a U5, até o Matheus confirmar a leitura. O teste da U5 que a exercitava passou a usar o agregador.
