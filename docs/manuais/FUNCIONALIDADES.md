# Funcionalidades do Agente de Bolso

Lista simples: o que é e qual é a regra. Situação: **Pronto** (código feito e testado), **Pronto (banco simulado)** (funciona contra o banco simulado; a Pluggy real entra na Fase B), **Desenhado** (design sem código) e **Planejado**. Valores sempre em centavos. Nada é apagado: correção é por estorno, inativação ou desfazer com histórico.

## Base do sistema

| Funcionalidade | O que é | Regra | Situação |
|---|---|---|---|
| Acesso por empresa | Login com token; cada empresa vê só os seus dados | Uma empresa nunca vê dados de outra | Pronto |
| Papéis | Admin, Operador, Consultor BPO e Auditor | Admin decide; Operador propõe; BPO só lê as empresas vinculadas; Auditor só lê | Pronto |
| Trilha de auditoria | Registro de toda escrita, em cadeia | Não pode ser alterada; ações privilegiadas pedem MFA | Pronto |
| Reenvio seguro | Chave de idempotência nas escritas | Repetir o mesmo pedido não cria nada em duplicidade | Pronto |

## Fluxo de caixa

| Funcionalidade | O que é | Regra | Situação |
|---|---|---|---|
| Lançamentos | Contas a pagar e a receber | Tem valor, vencimento, competência e categoria; com baixa ativa fica protegido | Pronto |
| Plano de contas | Categorias; já vem um plano inicial para serviços | Contador valida as 20 categorias | Pronto |
| Contas financeiras | Banco, caixa ou carteira digital | Toda conta tem titularidade PJ ou PF | Pronto |
| Baixa | Registrar o pagamento ou recebimento | Mostra a diferença de liquidação; pode ser parcial; estorno por nova baixa | Pronto |
| Situação do lançamento | Vencido, parcial ou pago | Calculada, nunca digitada | Pronto |
| Transferência entre contas | Dinheiro entre contas da empresa | Só entre contas da mesma titularidade (PJ↔PF é recusada) | Pronto |
| Saldo e projeção | Saldo realizado por conta e projeção de caixa | Padrão soma só contas PJ; PF só com permissão e declarada | Pronto |
| Histórico do lançamento | Quem mudou o quê e quando | Toda alteração fica registrada | Pronto |
| Centro de custo | Para quem ou onde o gasto aconteceu | Nome único (sem diferenciar caixa e acento); máximo de 100 ativos; inativa, nunca exclui | Pronto |
| Forma de pagamento | Pix, boleto, cartão etc. | Mesmas regras do centro de custo | Pronto |
| Titularidade da conta | PJ ou PF | Só se corrige sem movimento, por Admin com MFA e motivo | Pronto |
| Visibilidade de contas PF | Quem pode ver contas pessoais | Padrão: só Admin; os demais veem o pagamento como "conta restrita" | Pronto |

## Conciliação bancária

| Funcionalidade | O que é | Regra | Situação |
|---|---|---|---|
| Conta bancária | Cadastro da conta que recebe extrato | Ligada a uma conta financeira | Pronto |
| Importar OFX | Envio do extrato do banco | Arquivo inválido é recusado; arquivo repetido não reprocessa | Pronto |
| Sem duplicar | Reimportar o mesmo período | Transação já registrada não entra de novo | Pronto |
| Conferência de saldo | Compara o saldo do banco com o calculado | Três resultados: confere, diverge (com o valor) ou sem conferência | Pronto |
| Motor de conciliação | Liga transação do banco ao lançamento | Chave exata ou valor e data com prova; o que não é certo vai para a fila | Pronto |
| Fila de exceções | Casos que exigem decisão | Mostra os candidatos; nunca decide sozinha | Pronto |
| Confirmar em lote | Aceitar várias sugestões | Operador propõe, Admin confirma (segregação de funções) | Pronto |
| Vincular, criar lançamento, ignorar | Ações sobre uma pendência | Criar lançamento a partir da transação; ignorar exige motivo | Pronto |
| Transferência interna | Marcar par de transações | Recusada entre PJ e PF; esse par vai à fila como movimento entre titularidades | Pronto |
| Desfazer vínculo | Voltar uma conciliação | Mantém histórico | Pronto |
| Regras de reconhecimento | Tarifas e rendimentos | Só por regra cadastrada; o sistema aprende com as classificações | Pronto |
| Dividir e agrupar pagamentos | Um pagamento para vários lançamentos e o inverso | Soma bate com o valor | Pronto |
| Situação do período | Quanto já foi conciliado | Só leitura | Pronto |

## Entrada canônica e conexão bancária (U5)

| Funcionalidade | O que é | Regra | Situação |
|---|---|---|---|
| Entrada por linhas | Criar extrato a partir de linhas estruturadas, sem arquivo | Mesmo caminho do OFX: mesma deduplicação e conferência de saldo | Pronto (banco simulado) |
| Só efetivadas | Transação pendente do banco | Não entra; só entra quando efetivar, uma única vez | Pronto (banco simulado) |
| Conexão bancária | Ligação automática com o banco via agregador (somente leitura) | Estados: aguardando, rejeitada, ativa, com erro, expirada, revogada, desconectada | Pronto (banco simulado) |
| Consentimento | Autorização dada ao banco | Validade de até 12 meses; revogação vale na hora | Pronto (banco simulado) |
| Sincronização | Buscar movimentos da conta conectada | Repetir não duplica; falha não grava nada pela metade e a conexão fica com erro | Pronto (banco simulado) |
| Atualizar agora | Sincronização manual | Intervalo mínimo e orçamento de chamadas por conexão; valores sem padrão em produção | Pronto (banco simulado) |
| Reconectar | Renovar consentimento | Mantém a conexão e cobre o período parado; se faltar histórico, avisa a lacuna | Pronto (banco simulado) |
| Desconectar | Encerrar a conexão | Para a sincronização, não apaga nada; o OFX continua funcionando | Pronto (banco simulado) |
| Revisão ou cancelamento pelo banco | Banco muda ou cancela transação já registrada | Vai para a fila de exceções; nada é alterado sozinho | Pronto (banco simulado) |
| Banco simulado | Adaptador de teste, sem credenciais | Não inicia em produção | Pronto (banco simulado) |

## Importação de planilha, duplicidade e desfazer lote (U6, pronto)

| Funcionalidade | O que é | Regra | Situação |
|---|---|---|---|
| Leitor de XLSX e CSV | Lê a planilha pelo conteúdo, não pela extensão | Fórmula nunca é calculada; macro, PDF, executável e imagem são recusados; valor lido do texto cru, nunca como número de ponto flutuante | Pronto (leitura) |
| Proteção do arquivo | Defesa contra arquivo malicioso | Recusa bomba de descompressão, entradas duplicadas no zip, DOCTYPE/entidade externa, caminho perigoso, zip dentro de zip | Pronto (leitura) |
| Limites | Tamanho, linhas, colunas, célula, texto total, tempo | 10 MB, 5.000 linhas, 50 colunas, 1.000 caracteres por célula; passou do limite, recusa com motivo | Pronto (leitura) |
| Datas e valores | Interpreta datas do Excel (sistemas 1900 e 1904) e valores em pt-BR ou técnico | Mais de 2 casas decimais é recusado, nunca arredondado | Pronto (leitura) |
| Duplicidade entre origens | A mesma movimentação por OFX, planilha, banco ou entrada direta | Aparece lado a lado; manter as duas ou descartar uma, nada é apagado | Pronto (conciliação) |
| Identificador repetido em arquivo | Linha de OFX ou planilha com identificador já usado e valor diferente | Entra como movimentação nova com aviso; nunca vira "revisão do banco", que vale só para a conexão bancária | Pronto |
| Desfazer lote (motor) | Estorna conciliações e baixas de um lote inteiro | Tudo ou nada, com prévia; nada é apagado | Pronto (conciliação) |
| Lote de operação | Registro genérico de lotes na plataforma | Estados e itens com resultado | Pronto (plataforma) |
| Prévia, mapeador, layout lembrado, revisão, confirmação, desfazer pela API | Fluxo completo de importação | Aceitas, em revisão e recusadas | Pronto |

## Próximas etapas

| Funcionalidade | O que é | Situação |
|---|---|---|
| Estorno automático por revisão ou cancelamento do banco | Quando a conexão bancária muda valor, sentido ou data, ou cancela uma transação já registrada | A versão nova é guardada sem apagar a antiga; a conciliação é estornada pelo sistema e abre exceção; mudança só de descrição não mexe em nada; o saldo volta a ser conferível na importação seguinte; a pessoa confere a revisão para liberar o motor | Construído e testado (U5b) |
| Importar lançamentos por planilha | Contas a pagar e a receber de XLSX ou CSV, com revisão; linha com data de pagamento entra como já paga (lançamento e baixa); categoria e conta pelo nome, com memória; nunca cria cadastro; competência pela coluna ou pelo mês do lote; só o Admin confirma, e quem preparou o pagamento não confirma no modo estrito | Construído e testado (U7) |
| Já pago em conta pessoal | Linha paga pela conta PF do sócio | Vai para revisão: escolher conta da empresa, tratar como previsto ou recusar | Construído e testado (U7) |
| Valor recebido menor que a nota | Pagamento parcial, quitação com diferença ou retenção de imposto | Parcial só quando o pago é menor; para imposto retido, aviso até o contador definir a regra | Construído e testado (U7) |
| Recusados na gravação | Item que o fluxo de caixa recusou ao confirmar | Volta para revisão; depois o Admin reprocessa só esses itens, sem duplicar | Construído e testado (U7) |
| Desfazer lote de lançamentos | Cancela os lançamentos e estorna as baixas do lote | Tudo ou nada, validado antes; qualquer baixa conciliada bloqueia | Construído e testado (U7) |
| Parcelar uma compra ou venda | Valor total em 2 a 60 vezes, todo mês no mesmo dia | Cada parcela é um lançamento; o centavo que sobra vai na última; todas na competência da operação; prévia antes de gravar | Construído e testado (U8) |
| Repetir todo mês | Aluguel, mensalidade: valor de cada mês, de 2 a 60 meses | Competência mês a mês a partir da informada; pode estender depois | Construído e testado (U8) |
| Editar parcela ou sequência | "Só esta" ou "esta e as próximas" | Parcelas com baixa ou canceladas ficam de fora; nada é redistribuído | Construído e testado (U8) |
| Encerrar uma série | Cancelar "esta e as próximas" | Admin com MFA, motivo e segregação; baixadas ficam; nada é apagado | Construído e testado (U8) |
| Buscar | Texto livre nos lançamentos e nas transações do banco | Palavras por pedaço; valor só quando escrito como dinheiro; CPF ou CNPJ exato (inclusive o CNPJ novo, com letras); conta pessoal oculta para quem não pode ver | Construído e testado (U9) |
| Totalizar a seleção | Itens marcados em várias páginas, ou tudo que o filtro trouxer | Previsto, pago e em aberto; cancelados e desfeitos contados à parte; parte em conta pessoal declarada a quem pode ver | Construído e testado (U9) |
| Edição em massa e "dar baixa nos selecionados" | Operar muitos lançamentos de uma vez, com prévia e resultado item a item | Planejado (U10) |
| Exportação | Planilha com exatamente o filtro aplicado | Planejado (U11) |
| Categorização e nota de confiança | Sugestão explicada, nunca confirma sozinha | Planejado (U12) |
| DRE e relatórios | Resultado por competência, só PJ por padrão | Planejado (U13) |
| Banco de dados e Pluggy real | PostgreSQL e agregador real | Fase B |
| Pagamentos e IA externa | Etapa 8 | Fora de construção |
