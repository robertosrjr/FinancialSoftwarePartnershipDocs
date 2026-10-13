# Guia Rápido — o que o Agente de Bolso faz

**Para quem**: quem está começando e quer saber, em poucas linhas, o que cada funcionalidade faz. Para o passo a passo, use os manuais de cada domínio (links no fim de cada seção).

**Fase atual**: dados em memória (somem ao reiniciar); a conexão com o banco é simulada. **Quem pode o quê**: **Admin** faz tudo; **Operador** importa extratos e prepara propostas; **Consultor BPO** e **Auditor** só consultam. Quem trabalha sozinho é o Admin. Nada é apagado: erro se corrige com cancelamento ou estorno.

---

## 1. Fluxo de Caixa — o que vou pagar e receber

| Funcionalidade | Você informa | O que faz | Resultado |
|---|---|---|---|
| **Categorias** (Admin) | Nome e, se quiser, a categoria-mãe | Organiza os lançamentos por tipo de receita e despesa | Categoria pronta para usar; pode renomear, ajustar a atividade do DFC, desativar e reativar |
| **Contas financeiras** (Admin) | Nome, tipo, saldo inicial e data base | Cadastra onde o dinheiro está (banco, caixa, cartão) | Conta que passa a ter saldo; pode inativar e reativar |
| **Lançamento** (Admin) | Entrada ou saída, valor, categoria, descrição, vencimento e competência | Registra o que vai pagar ou receber, com a opção de já dar a baixa | Lançamento com situação: previsto, parcial, pago, vencido ou cancelado |
| **Alterar lançamento** (Admin) | O que mudar e o motivo | Corrige os dados de um lançamento | Lançamento atualizado, com a mudança no histórico |
| **Baixa** (Admin) | Conta, valor, data do pagamento e se é a baixa final | Registra o pagamento ou recebimento, no todo ou em partes | Lançamento pago ou parcial; saldo da conta atualizado |
| **Estornar baixa** (Admin) | Motivo | Desfaz uma baixa sem apagá-la | Baixa estornada; o lançamento volta a ficar em aberto |
| **Cancelar lançamento** (Admin) | Motivo (mín. 10 caracteres) | Cancela um lançamento sem baixa | Lançamento cancelado, no histórico |
| **Transferência entre contas** (Admin) | Conta de origem, de destino, valor, data | Move dinheiro entre as suas contas | Saldos das duas contas atualizados; não conta como receita nem despesa |
| **Saldo** | Conta e, se quiser, uma data | Mostra quanto há na conta, ou em todas | Saldo na data pedida |
| **Projeção** | Horizonte em dias (opcional) | Estima o caixa com o que está previsto | Entradas, saídas e saldo esperados no período |
| **Consultar e acompanhar** | Filtros: situação, direção, categoria, conta, centro de custo, forma de pagamento, período | Lista e detalha lançamentos e transferências, com o histórico de cada um | Lista paginada; histórico de quem fez o quê e quando |
| **Centros de custo e formas de pagamento** (Admin) | Nome | Cadastra para quem foi o gasto e como foi pago | Pode renomear, inativar e reativar; até 100 ativos |
| **Titularidade da conta** (Admin) | PJ ou PF ao cadastrar | Separa contas da empresa e pessoais do sócio | Saldo e projeção somam só PJ; PF só para quem pode ver |
| **Importar lançamentos por planilha** (Admin e Operador preparam; Admin confirma) | Arquivo XLSX ou CSV e o mapeamento das colunas | Cria contas a pagar e a receber em lote, com revisão | Lançamentos criados; linha com data de pagamento entra já paga; desfazer o lote inteiro com autenticação forte |
| **Parcelar ou repetir** (Admin) | Valor, número de vezes (2 a 60), primeiro vencimento | Cria as parcelas ou as repetições, com prévia | Um lançamento por mês; editar "só esta" ou "esta e as próximas"; encerrar com autenticação forte |
| **Buscar** | Texto: palavras, valor (`1.250,50`), CPF ou CNPJ, código | Procura nos lançamentos, junto com os filtros | Lista dos que casam; documento mascarado |
| **Somar a seleção** | Itens marcados (até 1.000) ou o filtro | Soma o que foi escolhido | Previsto, pago, em aberto, cancelados à parte, parte em conta pessoal (para quem vê) |

Mais detalhes: [Manual do Fluxo de Caixa](MANUAL-FLUXO-DE-CAIXA.md).

---

## 2. Conciliação Bancária — conferir o extrato com o fluxo de caixa

| Funcionalidade | Você informa | O que faz | Resultado |
|---|---|---|---|
| **Conta bancária** (Admin) | Conta financeira ligada, nome, banco, agência e conta | Cadastra a conta do banco que terá extrato | Conta pronta para importar; só os 4 últimos dígitos ficam visíveis |
| **Importar extrato** (Admin, Operador) | Arquivo OFX e a conta | Lê o extrato, descarta o que já entrou e confere o saldo | Resumo: transações novas, repetidas, avisos e estado do saldo |
| **Reimportar o mesmo arquivo** | O mesmo arquivo | Reconhece o que já foi importado | Nada duplica |
| **Reprocessar importação** (Admin) | Importação escolhida | Refaz a leitura de um extrato | Importação refeita |
| **Pendências** | Filtros: conta, período, estado, motivo | Lista as transações ainda sem lançamento, com sugestões | Lista paginada, com o que o sistema propõe para cada uma |
| **Confirmar sugestões** (Admin) | Sugestões escolhidas, ou a importação inteira | Aceita de uma vez o que o sistema propôs | Transações ligadas aos lançamentos; baixas feitas |
| **Ligar transação a baixa que já existe** (Admin, Operador) | A baixa manual (escolhida entre as compatíveis) | Liga a transação a um pagamento que você já tinha registrado, sem criar outra baixa | Transação conciliada; saldo e lançamento inalterados. Desfazer não estorna a baixa |
| **Ligar transação a lançamento** (Admin, Operador) | Lançamento e valor | Liga uma transação a um lançamento, no todo ou em parte | Transação conciliada; baixa feita (confirmada pelo Admin, se foi o Operador) |
| **Criar lançamento da pendência** (Admin, Operador) | Categoria e contraparte | Cria o lançamento que falta a partir da transação | Lançamento criado e ligado |
| **Transferência interna** (Admin, Operador) | Conta de contrapartida e, se houver, a outra ponta | Marca uma transação como movimento entre as suas contas | As duas pontas conciliadas, sem virar receita ou despesa |
| **Par de estorno** (Admin, Operador) | A transação do estorno | Casa uma cobrança com o seu estorno | As duas conciliadas, sem mexer no caixa |
| **Rejeitar sugestão** (Admin) | Motivo | Recusa o que o sistema propôs | Transação volta para as pendências |
| **Desfazer** (Admin) | Motivo | Desfaz uma ligação sem apagá-la | Transação volta a ficar pendente; saldos devolvidos |
| **Ignorar e reativar** (Admin) | Motivo | Tira uma transação da fila (ex.: tarifa já tratada) ou a traz de volta | Transação ignorada ou pendente de novo |
| **Regras de reconhecimento** (Admin) | Texto a reconhecer, direção e categoria | Ensina o sistema a reconhecer transações parecidas | Regra ativa; pode ativar e desativar |
| **Situação do período** | Conta e período | Resume quantas transações estão em cada estado | Quantidade e valor por estado, e o estado do saldo |
| **Histórico** | A transação ou o lançamento | Mostra tudo o que aconteceu com ela | Linha do tempo com quem, quando e por quê |
| **Importar extrato por planilha** (Admin, Operador) | Arquivo XLSX ou CSV, conta e mapeamento | Lê a planilha com proteção contra arquivo malicioso e segue o caminho do OFX | Mesmo resumo do OFX; layout lembrado |
| **Duplicidade entre origens** (Admin decide) | Manter as duas ou descartar uma | Mostra lado a lado a mesma movimentação vinda de origens diferentes | Nada é descartado sozinho |
| **Desfazer importação** (Admin, autenticação forte) | Número da prévia e motivo | Estorna tudo o que a importação gerou, tudo ou nada | Importação desfeita; pode reimportar em seguida |
| **Conexão com o banco** (Admin; banco simulado) | Autorização no banco e as contas | Traz o extrato sozinho, só leitura | Sincronização automática, "atualizar agora", reconectar e desconectar |
| **Revisão pelo banco** (banco simulado) | — | Quando o banco muda ou cancela uma transação, guarda as duas versões e estorna a conciliação | Exceção na fila para conferir; meses fechados mexidos ficam listados |
| **Buscar e somar transações** | Texto e filtros; itens ou filtro | Procura e soma transações do banco | Entradas, saídas e líquido; ignoradas somam, desfeitas e canceladas ficam fora |

Mais detalhes: [Manual da Conciliação Bancária](MANUAL-CONCILIACAO-BANCARIA.md).

---

## 3. Plataforma — acesso, equipe e confiança

| Funcionalidade | Você informa | O que faz | Resultado |
|---|---|---|---|
| **Usuários** (Admin, Auditor) | — | Lista quem tem acesso à empresa e com qual papel | Lista de usuários |
| **Desativar usuário** (Admin) | Motivo (mín. 10 caracteres) | Bloqueia o acesso de uma pessoa | Usuário sem acesso; histórico preservado |
| **Reativar usuário** (Admin) | — | Devolve o acesso | Usuário ativo de novo |
| **Modo de segregação** (consulta: todos; alteração: Admin) | Modo (estrito ou revisor único) e motivo | Define se quem criou um item pode confirmá-lo | Estrito: outra pessoa confirma. Revisor único: o Admin confirma, com marcação e relatório |
| **Relatório de controle** (Admin, Consultor BPO, Auditor) | Data inicial e final | Lista o que foi confirmado sem revisão independente | Relatório do período |
| **Verificação da trilha de auditoria** (Admin, Auditor) | Completa ou só o novo | Confere que o histórico não foi alterado | Trilha íntegra, ou o ponto em que há divergência |

Valem para tudo: cada empresa vê só os próprios dados, ações críticas pedem confirmação de segurança recente, e o histórico nunca é apagado. Detalhes: [Manual do Usuário](MANUAL-DO-USUARIO.md).

---

Quer ir mais fundo? [Manual do Usuário (índice)](MANUAL-DO-USUARIO.md) · [Regras de negócio](../sdd/REGRAS-DE-NEGOCIO.md) · [Jornada técnica](../jornadas/JORNADA-TECNICA.md)
