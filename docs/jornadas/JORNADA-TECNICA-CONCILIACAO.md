# Jornada Técnica — Conciliação Bancária

**Versão**: 1.1 · **Data**: 2026-10-04 · **Produto**: Agente de Bolso
**Para quem**: engenheiros que vão manter, integrar ou revisar o módulo `conciliacao` (backend, integradores de canal, SRE, segurança).
**Documentos irmãos**: [Fluxo de Caixa](JORNADA-TECNICA-FLUXO-DE-CAIXA.md) · [Plataforma e visão geral](JORNADA-TECNICA.md) (requisição, idempotência, auditoria, segregação, dados, observabilidade) · usuário final: [Manual de Conciliação](MANUAL-CONCILIACAO-BANCARIA.md).

## Como ler

| Selo | Significado |
|---|---|
| **[Implementado]** | Está no código e coberto por testes (U3a e U3b; `./mvnw verify` verde) |
| **[Desenhado]** | Aprovado, ainda sem código (persistência definitiva, Fase B) |

## Onde a conciliação se encaixa

```mermaid
flowchart LR
    CANAL["Canais"] -->|"REST e JWT"| CONC["conciliacao"]
    CONC -->|"api sincrona: candidatos, baixa, lancamento com baixa, transferencia"| FC["fluxocaixa"]
    CONC -->|"api: auditoria, seguranca, segregacao, indice cego"| PLAT["plataforma"]
    CONC -.->|"arquivo original"| S3[("S3 ou MinIO, Fase C")]
    CONC -.->|"Fase B"| PG[("PostgreSQL com RLS")]
```

Texto equivalente: a conciliação recebe chamadas dos canais, usa a API do fluxo de caixa de forma síncrona para candidatos, baixas, lançamento com baixa e transferência, usa a API da plataforma para auditoria, segurança, segregação e índice cego, e guarda o arquivo original num armazenamento externo (memória na Fase A).

A conciliação **só enxerga `fluxocaixa.api`** (regra de arquitetura no build) e nunca lê ou grava tabelas do fluxo de caixa (CB-05). Convenções gerais (pipeline da requisição, erros, MFA, tenant, auditoria) estão em [JORNADA-TECNICA.md](JORNADA-TECNICA.md).

---

## 1. Jornada: cadastrar conta bancária **[Implementado]**

`POST /api/v1/conciliacao/contas` · Admin + MFA · `GET /contas` e `GET /contas/{id}` para os quatro papéis.

```mermaid
sequenceDiagram
    autonumber
    participant K as ContaController
    participant U as ServicoDeContas
    participant P as ProtecaoDeDadosApi
    participant L as TravaPorConta
    participant R as RepositorioDeContas
    participant A as Auditoria

    K->>U: Cadastrar nome, banco, agencia, conta
    U->>U: Exige Admin e MFA
    U->>U: Normaliza e valida a identificacao
    U->>P: Calcular indice completo e indice sem agencia
    P-->>U: HMAC por tenant e finalidade
    U->>L: Trava por tenant
    L->>R: Limite de 50, unicidade do indice e do nome
    R-->>U: Livre
    U->>R: Salvar conta, com 4 ultimos digitos
    U->>A: CONTA_BANCARIA_CADASTRADA, so o codigo do banco
    U-->>K: Conta com numero mascarado
```

**Proteção do número** ([ADR 005](adr/005-indice-cego-numero-de-conta-nao-armazenado.md)): a agência e a conta nunca são persistidas. A normalização (maiúsculas, só letras e dígitos, sem zeros à esquerda) é a mesma no cadastro e na leitura do OFX, o que torna o reconhecimento por igualdade possível. O HMAC inclui tenant e finalidade; rotacionar a chave exige reindexar.

---

## 2. Jornada: importar o extrato **[Implementado]**

`POST /api/v1/conciliacao/importacoes` (multipart, Admin ou Operador, 20 por minuto) · `GET /importacoes/{id}?aguardar=30` · `GET /importacoes` · `POST /importacoes/{id}/reprocessamento` (Admin + MFA) · `GET /importacoes/{id}/arquivo` (Admin ou Auditor + MFA) · `GET /transacoes/{id}/origem`.

### 2.1 Recebimento (síncrono) e processamento (assíncrono)

```mermaid
sequenceDiagram
    autonumber
    participant C as Cliente
    participant K as ImportacaoController
    participant R as ServicoDeRecebimento
    participant O as LeitorDeOfx
    participant L as TravaPorConta
    participant ST as ArmazenamentoDeExtratos
    participant RI as RepositorioDeImportacoes
    participant EV as PublicadorDeEventos
    participant D as DisparadorDeProcessamento
    participant PR as ServicoDeProcessamento

    C->>K: POST arquivo OFX
    K->>R: Receber conteudo
    R->>R: Papel, tamanho ate 10 MB
    R->>O: Ler e validar o arquivo inteiro
    O-->>R: ExtratoCanonico ou erro 415, 400, 413
    R->>R: Moeda BRL, ate 5000 transacoes, resolver a conta
    R->>ST: Guardar o arquivo por hash, fora da trava
    R->>L: Trava por conta e arquivo
    L->>RI: Importacao mais recente com o mesmo hash
    alt Ja concluida
        RI-->>R: Existe CONCLUIDA
        R->>RI: Salvar importacao REPETIDA
        R-->>C: 200 com novas = 0
    else Em andamento
        R-->>C: 202 com a importacao existente
    else Nova
        R->>RI: Salvar importacao RECEBIDA
        R->>R: Auditar IMPORTACAO_RECEBIDA
    end
    R->>EV: ImportacaoRecebida, depois da gravacao e fora da trava
    R-->>C: 202 com Location
    EV->>D: Evento
    D->>PR: processar, uma thread virtual, ate 8 simultaneos
```

Decisões embutidas:
- **O arquivo inteiro é lido e validado no recebimento.** Qualquer erro recusa tudo, sem gravar. O parser OFX é próprio (SGML do 1.x, StAX do 2.x), sem DTD, sem entidades externas, com limites de profundidade, elementos e texto. A mensagem de erro nunca repete o conteúdo.
- **Gravar o arquivo vem antes da trava**, porque ele é endereçado por hash (gravar duas vezes é inofensivo) e o armazenamento externo pode demorar.
- **O evento sai depois de gravar e fora da trava.** Na Fase A não há transação e o consumidor roda assim que o evento sai; na Fase B a entrega passa a ser depois do commit, pelo registro persistente de eventos do Modulith.

### 2.2 Processamento em três tempos

```mermaid
sequenceDiagram
    autonumber
    participant PR as ServicoDeProcessamento
    participant L as TravaPorConta
    participant RI as RepositorioDeImportacoes
    participant ST as ArmazenamentoDeExtratos
    participant OF as LeitorDeExtrato
    participant RC as RepositorioDeContas
    participant DD as Deduplicador e ConferenciaDeSaldo
    participant CO as ConclusaoDeImportacao
    participant A as Auditoria e eventos

    PR->>RI: Importacao RECEBIDA?
    alt Nao existe ou nao esta RECEBIDA
        PR->>PR: Ignora com log e metrica por motivo
    end
    PR->>L: Tempo 1, sob trava
    L->>RI: Marcar PROCESSANDO e guardar a tentativa
    PR->>ST: Tempo 2, sem trava: ler o arquivo
    PR->>PR: Conferir o SHA-256 com o registrado
    PR->>OF: Interpretar o OFX
    PR->>L: Tempo 3, sob trava
    L->>RC: Ler a conta com o saldo conferido e a versao
    L->>DD: Classificar linhas, deduplicar, conferir o saldo
    DD-->>L: Resumo, transacoes novas, novo saldo
    L->>CO: Concluir com dados, a tentativa e a versao da conta
    CO-->>L: Gravou tudo ou descartou
    PR->>A: Fora da trava: auditoria, metrica, ImportacaoConcluida
```

- **Atomicidade**: transações novas, importação concluída e saldo conferido da conta são gravados **juntos ou nada**. A porta `ConclusaoDeImportacao` recebe **dados** e carrega os agregados dentro da unidade atômica, para que "descartar" nunca deixe efeito (nem com entidades gerenciadas por JPA na Fase B).
- **Token de tentativa**: se a importação expirou ou foi reprocessada durante o processamento, o resultado é descartado sem gravar nada.
- **Conflito transitório** (conta alterada por outra operação, chave já existente) é repetido até 3 vezes antes de a importação falhar.
- **Falhas** viram `FALHOU` com código estável: `arquivo_indisponivel`, `arquivo_corrompido`, `capacidade_excedida`, `tempo_esgotado`, `erro_interno`.
- **Etapa pós-importação**: a porta `EtapaPosImportacao` (nula na U3a) é onde a U3b ligará o motor. Falha nela é contida e gera métrica; a importação continua concluída.

### 2.3 Estados da importação

```mermaid
stateDiagram-v2
    [*] --> RECEBIDA
    RECEBIDA --> PROCESSANDO: processamento inicia
    PROCESSANDO --> CONCLUIDA: grava tudo
    PROCESSANDO --> FALHOU: erro ou tempo esgotado
    RECEBIDA --> FALHOU: expirou em 10 minutos
    FALHOU --> RECEBIDA: reprocessar, Admin com MFA, ate 5 vezes
    [*] --> REPETIDA: mesmo arquivo ja concluido
```

Um job a cada minuto expira importações sem andamento há mais de 10 minutos e alerta; nunca reprocessa sozinho. `aguardar` faz long-polling de até 30 s, com teto de esperas simultâneas por empresa.

### 2.4 Deduplicação ([ADR 006](adr/006-deduplicacao-por-fitid-com-conferencia-de-conteudo.md))

```mermaid
flowchart TD
    L["Linha do arquivo"] --> F{"Data depois de hoje?"}
    F -- "Sim" --> FU["Futura: fora e contada"]
    F -- "Nao" --> Z{"Valor zero?"}
    Z -- "Sim" --> ZE["Zerada: fora e contada"]
    Z -- "Nao" --> H{"Tem FITID?"}
    H -- "Nao" --> C1{"Chave C v1 com o hash existe na conta?"}
    C1 -- "Sim" --> DUP["Duplicada: descartada"]
    C1 -- "Nao" --> NOV["Nova"]
    H -- "Sim" --> K1{"Chave F v1 do FITID existe?"}
    K1 -- "Nao" --> K2{"Chave FC existe?"}
    K2 -- "Sim" --> DUP
    K2 -- "Nao" --> NOVF["Nova, aviso possivel_duplicata se o conteudo ja existia antes"]
    K1 -- "Sim, mesmo conteudo" --> DUP
    K1 -- "Sim, conteudo diferente" --> K3{"Chave FC existe?"}
    K3 -- "Sim" --> DUP
    K3 -- "Nao" --> NOVI["Nova, aviso fitid_inconsistente"]
```

- O **hash** é SHA-256 de `v1 | conta | data | direção | centavos | descrição normalizada | ordinal`, com serialização explícita, e o **ordinal** conta ocorrências de linhas idênticas no arquivo.
- O FITID entra na chave **como hash**, para um valor com `:` não forjar a chave de outra linha. O formato é versionado e há testes com valor congelado.
- No máximo **duas consultas** ao repositório por importação (em lote). A unicidade (conta, chave) também é conferida na gravação, como última barreira.
- `linhasNoArquivo = novas + duplicadas + futuras + zeradas`, sempre.

### 2.5 Conferência de saldo

```mermaid
flowchart TD
    S["Extrato com saldo S na data D"] --> A{"Tem saldo no arquivo?"}
    A -- "Nao" --> N1["SEM_CONFERENCIA: SEM_SALDO_NO_EXTRATO"]
    A -- "Sim" --> B{"D depois de hoje?"}
    B -- "Sim" --> N2["SEM_CONFERENCIA: SALDO_COM_DATA_FUTURA"]
    B -- "Nao" --> C{"Conta ja tem saldo conferido?"}
    C -- "Nao" --> N3["SEM_CONFERENCIA: PRIMEIRA_IMPORTACAO, saldo vira a referencia"]
    C -- "Sim" --> D{"D depois da data conferida?"}
    D -- "Nao" --> N4["SEM_CONFERENCIA: ANTERIOR_AO_SALDO_CONFERIDO"]
    D -- "Sim" --> E{"Extrato cobre a partir do dia seguinte ao conferido?"}
    E -- "Nao" --> N5["SEM_CONFERENCIA: LACUNA_ENTRE_EXTRATOS"]
    E -- "Sim" --> F{"S menos transacoes depois do conferido igual ao saldo conferido?"}
    F -- "Sim" --> OK["SALDO_CONFERIDO: saldo da conta avanca"]
    F -- "Nao" --> DV["SALDO_DIVERGENTE com a diferenca: saldo da conta nao avanca, alerta"]
```

O OFX não traz saldo inicial: ele é **derivado** (saldo do banco menos as transações até a data do saldo) e o que se confere é a **continuidade** com o último saldo conferido. O saldo conferido da conta só avança em `SALDO_CONFERIDO` ou na `PRIMEIRA_IMPORTACAO`.

### 2.6 Baixar o arquivo original e rastrear a origem

```mermaid
sequenceDiagram
    autonumber
    participant C as Admin ou Auditor
    participant K as ImportacaoController
    participant U as ServicoDeConsultas
    participant A as Trilha de acesso
    participant ST as ArmazenamentoDeExtratos

    C->>K: GET importacoes id arquivo
    K->>U: Obter arquivo original
    U->>U: Papel Admin ou Auditor e MFA recente
    U->>A: Registrar o acesso, antes das verificacoes
    U->>ST: Ler o arquivo
    U->>U: Recalcular o SHA-256 e comparar
    alt Diferente
        U-->>C: 503 arquivo_corrompido e alerta
    else Igual
        U-->>C: Conteudo, sem cache, nome generico
    end
```

---

## 3. Jornada: conciliar **[Implementado, U3b]**

Os adaptadores continuam em memória (Fase A). As garantias de **ordem de gravação e repetição segura** não dependem de transação de banco: valem hoje e continuam valendo no PostgreSQL.

### 3.1 Motor em três camadas, logo depois da importação

```mermaid
sequenceDiagram
    autonumber
    participant PR as ProcessarImportacao
    participant EX as ExecutorDoMotor
    participant CT as CatalogoDeLancamentos
    participant FC as FluxoCaixaApi
    participant M as MotorDeConciliacao
    participant V as Vinculos, ledger e fila
    participant A as Auditoria

    PR->>EX: aposPersistir, importacao concluida
    EX->>EX: sob a trava da conta, so as transacoes elegiveis
    EX->>EX: sugere pareamento de transferencia ja marcada
    EX->>CT: candidatos em lote (blocos de 500) e lancamento por chave
    CT->>FC: buscar candidatos
    FC-->>EX: candidatos por transacao
    EX->>M: decidir, regras ativas, saldo conferido, valor ja reservado
    alt Camada 1 chave exata
        M-->>EX: Vincular
    else Camada 2 candidato unico com documento coincidente
        M-->>EX: Vincular
    else Camada 2 candidato unico sem documento
        M-->>EX: Sugerir
    else Dois ou mais candidatos, valor proximo ou documento diferente
        M-->>EX: EnviarParaExcecao com motivo e candidatos
    else Camada 3 regra cadastrada, sem candidato nas camadas 1 e 2
        M-->>EX: ReconhecerPadrao ou SugerirNovoLancamento
    else Nada
        M-->>EX: EnviarParaExcecao SEM_CANDIDATO
    end
    EX->>FC: baixa de conciliacao ou lancamento com baixa, alocacaoId igual ao id do vinculo
    FC-->>EX: BaixaId
    EX->>A: Registrar, ator sistema
    EX->>V: Anexar evento ao ledger e gravar o vinculo
```

- **Regra de ouro**: com 2 ou mais candidatos a camada 2 nunca decide. Com saldo **divergente ou sem conferência**, o motor só sugere (e põe o item na fila com o motivo "saldo não conferido").
- O `MotorDeConciliacao` é **puro e determinístico**: sem I/O, sem IA (CB-07). Toda a decisão vive nele; o `ExecutorDoMotor` só traduz entradas em chamadas e executa. Isso permite testar as propriedades de segurança com milhares de casos gerados.
- O executor **reavalia a elegibilidade** de cada transação sob a trava dela, imediatamente antes de agir: se uma ação manual a alocou no meio-tempo, o motor a pula.
- **Ordem de gravação** (BR-LD-07): fluxo de caixa, auditoria, ledger, vínculo. O **identificador do vínculo é determinístico** (hash de tenant, transação, destino, valor e ordinal), e é também o `alocacaoId` enviado ao fluxo de caixa: se algo falhar depois da baixa, repetir chega ao mesmo identificador, o fluxo de caixa devolve a mesma baixa e o resto se completa. O ledger é idempotente por (vínculo, tipo de evento).
- Uma recusa ou indisponibilidade do fluxo de caixa deixa **só aquela transação** pendente (métrica `conciliacao.automacao`); reprocessar tenta de novo.
- Trava, na ordem: **conta bancária** (só o motor e a importação), **transação** (por identificador, quando o grupo tem duas), **fluxo de caixa**. O fluxo de caixa nunca chama a conciliação, então não há ciclo.

### 3.2 Alocar, criar da pendência e confirmar

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin ou Operador
    participant S as ServicoDeVinculos
    participant G as PoliticaDeSegregacaoApi
    participant FC as FluxoCaixaApi
    participant L as Ledger
    participant A as Auditoria

    U->>S: Alocar valor a um lancamento, ou criar o lancamento da pendencia
    S->>S: Travar a transacao, conferir os tetos e o estado
    alt Operador, ou Admin em modo estrito
        S->>L: PROPOSTA_CRIADA, vinculo PROPOSTO (nada vai ao fluxo de caixa)
    else Admin em revisor unico
        S->>FC: registrarBaixaDeConciliacao ou criarLancamentoComBaixa
        S->>G: avaliarConfirmacao, registra a auto-aprovacao
        S->>A: VINCULO_CONFIRMADO_SEM_REVISOR
        S->>L: ALOCACAO_CONFIRMADA, vinculo ATIVO
    end
```

**Confirmar** (`ServicoDeConfirmacoes`, um ou lote de até 500, resultado parcial): para cada item, em ordem estável e de forma **independente**, o serviço carrega o vínculo, pergunta à segregação de funções se o confirmador pode (`sod_violation` auditado, antes de qualquer efeito), trava a transação (e a da outra ponta, em transferência e par), **revalida os tetos**, executa a baixa, a criação com baixa ou a transferência e só então audita, anexa ao ledger e grava. Falha de um item vai para a lista de recusas com o código; nunca aborta os demais. A confirmação por importação (`importacaoId`) pega as sugestões do sistema daquela importação, até o limite do lote, e informa quantas ficaram de fora.

**Criar da pendência** só vale pelo valor **inteiro** da transação (saldo livre igual ao valor). A categoria é a escolhida ou a sugestão em cadeia (3.5); em qualquer caso é validada (existe, ativa, aceita a direção). Só a escolha explícita é aprendida (`aprenderClassificacao`).

### 3.3 Transferência interna e par de estorno

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin ou Operador
    participant S as ServicoDeTransferenciasEPares
    participant FC as FluxoCaixaApi
    participant L as Ledger

    U->>S: Marcar transferencia (contraconta, par opcional)
    S->>S: Travar as duas transacoes, conferir livres, contas, direcao, valor e data
    S->>FC: registrarTransferenciaDeConciliacao, alocacaoId igual ao id do vinculo
    S->>L: TRANSFERENCIA_INTERNA_CONFIRMADA nas duas pontas
    Note over S: a outra ponta e gravada antes do vinculo principal e tem id derivado dele
```

- **Dois vínculos por grupo** (um por transação), ligados pela mesma transferência (ou pelo `parDeTransacaoId`). O vínculo da outra ponta é gravado **antes** do principal, com identificador derivado do principal e ordinal fixo: repetir depois de uma falha no meio converge.
- **Pareamento posterior**: se a outra ponta ainda não foi importada, o vínculo existe só para a transação marcada. Quando outra transação compatível (outra conta, contraconta igual a esta conta, direção oposta, valor igual, data dentro da tolerância) é importada, o motor cria uma **sugestão** ligada à mesma transferência; confirmar liga, sem registrar outra transferência. Só com **um** candidato; sugestão rejeitada não volta.
- **Par de estorno**: duas transações da mesma conta, direção oposta, valor igual; só ledger, sem chamar o fluxo de caixa.
- Operador propõe e Admin confirma (segregação). A outra ponta só nasce na confirmação, que revalida que ela continua livre.

### 3.4 Desfazer

```mermaid
sequenceDiagram
    autonumber
    participant C as Admin com MFA
    participant U as ServicoDeDesfazimento
    participant FC as FluxoCaixaApi
    participant L as Ledger
    participant A as Auditoria

    C->>U: Desfazer o vinculo, com motivo
    U->>U: Travar as transacoes do grupo, vinculo precisa estar ATIVO
    alt Alocacao ou novo lancamento
        U->>FC: estornarBaixaDeConciliacao (cancela o lancamento criado pela conciliacao)
    else Transferencia interna
        U->>FC: estornarTransferenciaDeConciliacao
    else Par de estorno
        U->>U: Nada a estornar no fluxo de caixa
    end
    U->>A: VINCULO_DESFEITO, so identificadores
    U->>L: ALOCACAO_ESTORNADA apontando a confirmacao, com o motivo
    U->>U: Marcar os vinculos do grupo como DESFEITO
```

- Estorna **só a baixa do próprio vínculo**; uma baixa manual do usuário nunca é tocada. Baixa ou transferência **já estornada** (tentativa anterior parou no meio) é tratada como já concluída (BR-DS-06).
- Em grupo, os membros são gravados em sequência. Se a gravação parar entre eles, repetir pelo membro ainda ativo conclui; o já desfeito responde `vinculo_nao_ativo`.

### 3.5 Regras de reconhecimento e sugestão de categoria

- **Regras** (`ServicoDeRegras`): só Admin com MFA escreve; a categoria precisa ser de resultado financeiro ou despesa operacional, ativa e da mesma direção; uma ativa por (padrão, direção), até 100 ativas; nunca excluída. A auditoria leva só identificadores (nunca o padrão).
- **Camada 3**: a regra de padrão mais longo cria lançamento e baixa na categoria dela (ator sistema, `RECONHECIMENTO_DE_PADRAO`, `regraId` no ledger); com saldo não conferido só sugere. Desfazer cancela o lançamento criado.
- **Sugestão em cadeia** (`SugestorDeCategoriaEmCadeia`): regra aprendida do usuário, depois a tabela fixa (tarifas, rendimentos, estorno, resolvidas pela categoria padrão no fluxo de caixa). Só sugere.

### 3.6 Ignorar e reativar

Ignorar (Admin, MFA, motivo) só vale sem vínculo ativo; rejeita, em nome do sistema, as sugestões e propostas abertas e anexa `TRANSACAO_IGNORADA`. O estado ignorado é o último evento entre ignorar e reativar no ledger, então reativar devolve a transação ao que o resto dos fatos diz (pendente ou em exceção).

### 3.7 Consultas

`ServicoDeConsultaDeConciliacao` só lê. A situação de cada transação é **derivada** (soma dos vínculos ativos, ignorada pelo ledger, item na fila) por consultas de custo constante, sem carregar agregado de vínculo; o repositório de transações mantém um índice por data e identificador (`noPeriodo`) e devolve as instâncias guardadas, imutáveis. Pendências detalham só a página (candidatos numa chamada em lote ao fluxo de caixa; se ele não responde, a lista sai com `candidatosIndisponiveis`). O Consultor BPO lê a situação de outra empresa só se ela estiver na claim `bpo_tenants`. Leituras de Auditor e Consultor BPO geram log de acesso sem o conteúdo.

O ledger, a fila de exceções, os vínculos e as transações são **sem exclusão**; uma regra de arquitetura no build proíbe métodos de alteração e exclusão nas portas do ledger e da fila.

---

## 4. Mapa de endpoints

Prefixo `/api/v1/conciliacao`. Toda escrita exige `Idempotency-Key`. Papel e MFA são verificados no endpoint **e** no caso de uso.

| Método e caminho | Papéis | MFA | Resposta |
|---|---|---|---|
| `POST /contas` | Admin | sim | 201 |
| `GET /contas`, `/contas/{id}` | todos | não | 200 |
| `POST /importacoes` | Admin, Operador | não | 202 |
| `GET /importacoes`, `/importacoes/{id}?aguardar=` | todos | não | 200 |
| `POST /importacoes/{id}/reprocessamento` | Admin | sim | 202 |
| `GET /importacoes/{id}/arquivo` | Admin, Auditor | sim | 200 |
| `GET /transacoes/{id}/origem` | todos | não | 200 |
| `POST /transacoes/{id}/alocacoes` | Admin (ativo ou proposto), Operador (propõe) | Admin: sim | 201 |
| `POST /transacoes/{id}/baixa-existente` (M15) | Admin (ativo ou proposto), Operador (propõe) | Admin: sim | 201 |
| `POST /transacoes/{id}/lancamento` | Admin, Operador (propõe) | Admin: sim | 201 |
| `POST /transacoes/{id}/transferencia-interna` | Admin, Operador (propõe) | Admin: sim | 201 |
| `POST /transacoes/{id}/estorno` | Admin, Operador (propõe) | Admin: sim | 201 |
| `POST /transacoes/{id}/ignorar`, `/reativar` | Admin | sim | 204 |
| `POST /vinculos/confirmacoes` | Admin | sim | 200 (resultado parcial) |
| `POST /vinculos/{id}/rejeicao` | Admin | sim | 204 |
| `POST /vinculos/{id}/desfazer` | Admin | sim | 204 |
| `GET /regras-de-reconhecimento` | todos | não | 200 |
| `POST /regras-de-reconhecimento`, `/{id}/ativacao`, `/{id}/desativacao` | Admin | sim | 201 / 200 |
| `GET /pendencias` | todos | não | 200 |
| `GET /situacao?conta&de&ate[&tenant]` | todos (BPO com `tenant`) | não | 200 |
| `GET /transacoes/{id}/historico`, `/lancamentos/{id}/historico` | todos | não | 200 |
| `GET /transacoes/{id}/baixas-compativeis` (M15) | todos | não | 200 |

O Operador cria **propostas** pelos mesmos endpoints de escrita (a resposta traz o vínculo `PROPOSTO`). Códigos de erro estáveis, entre outros: `alocacao_acima_da_transacao`, `alocacao_acima_do_em_aberto` (422), `lancamento_quitado`, `vinculo_nao_pendente`, `vinculo_nao_ativo`, `transacao_ignorada`, `transacao_com_vinculo`, `transacao_parcialmente_alocada`, `regra_duplicada`, `par_ja_conciliado` (409), `sod_violation` (403), `categoria_obrigatoria`, `categoria_inativa`, `categoria_nao_permitida_para_regra`, `contas_iguais`, `par_incompativel`, `periodo_invalido`, `lote_grande_demais` (422).

## 5. Dados e persistência

- **Hoje (Fase A)**: repositórios em memória, isolados por tenant, que guardam e devolvem cópias, conferem versão e recusam escrita de outro tenant dentro da operação. A conclusão da importação usa um monitor único sobre contas, importações e transações para ser atômica.
- **Fase B**: PostgreSQL com `tenant_id` e RLS, tabelas `con_*`; `ledger_conciliacao` sem `UPDATE` nem `DELETE`; estado da transação **não é coluna** (derivado dos vínculos ativos); teto de alocação por trava da transação.
- **O que a Fase B troca na U3b**: a trava por transação (hoje monitor em memória) vira trava de linha ou advisory lock; `noPeriodo` e a situação do período viram agregação SQL com índices `(tenant, conta, data)`; o índice de grupo por transferência vira índice por `transferencia_id`; a entrega dos eventos passa a ser depois do commit. As metas de desempenho (pendências e situação em até 500 ms com 50 mil transações) foram medidas em memória e precisam do teste de carga contra o PostgreSQL (risco RK-U3b-1).
- **Pendências para a Fase B**: fronteira transacional do recebimento (porta de unidade de trabalho), consultas entre tenants do job de expiração e do indicador sob RLS, entrega dos eventos depois do commit, ciclo de vida de dados pessoais ([ADR 011](adr/011-retencao-e-eliminacao-de-dados-pessoais.md)).

## 6. Observabilidade

- **Métricas da U3b** (Micrometer, sem tenant, usuário, valor, descrição nem documento): `conciliacao.operacao` por operação e resultado (uma por caso de uso: alocar, confirmar, desfazer, ignorar, consultas...), `conciliacao.motor` por camada e resultado, `conciliacao.fila` por motivo, `conciliacao.automacao` por resultado, `conciliacao.motor.duracao`. Os nomes diferem dos rascunhos do BR-AP-03 (`conciliacao.vinculo`, `conciliacao.consulta`): a operação virou rótulo de `conciliacao.operacao`.
- **Métricas** (Micrometer): `conciliacao.importacao` por resultado, `conciliacao.importacao.processamento`, `...espera`, `...linhas`, `conciliacao.importacao.saldo` por situação, `conciliacao.importacao.descartada` e `conciliacao.processamento.ignorado` por motivo, `conciliacao.etapa_pos_importacao.falhou`. Tags de baixa cardinalidade, nunca tenant, conta ou usuário.
- **Alertas**: importação parada (gauge da idade da mais antiga em andamento), saldo divergente, arquivo corrompido, importação falhou ou expirou.
- **Spans**: `conciliacao.receber_extrato`, `conciliacao.processar_importacao`, `conciliacao.expirar_importacoes` e, na U3b, um por caso de uso (`conciliacao.alocar`, `conciliacao.confirmar`, `conciliacao.desfazer`, `conciliacao.listar_pendencias`...).
- **Logs** sem PII: nunca número de conta, descrição, valor, nome de arquivo ou trecho do OFX; só contagens, códigos e identificadores técnicos.

## 7. Onde está o código

| O quê | Onde |
|---|---|
| Domínio: agregados, deduplicação, conferência de saldo | `application/conciliacao/src/main/java/.../conciliacao/domain` |
| Casos de uso e portas | `.../conciliacao/application` |
| Parser OFX (SGML 1.x e StAX 2.x), repositórios, web | `.../conciliacao/infrastructure` |
| Contratos publicados (ids e eventos) | `.../conciliacao/api` |
| Motor puro, vínculo, ledger, estado derivado, extração de documento | `.../conciliacao/domain` (`MotorDeConciliacao`, `Vinculo`, `EventoDoLedger`, `CalculadoraDeAlocacao`, `ExtratorDeDocumentoEChaves`) |
| Execução do motor, alocar, confirmar, desfazer, transferência, ignorar, regras, consultas | `.../conciliacao/application` (`ExecutorDoMotor`, `ServicoDeVinculos`, `ServicoDeConfirmacoes`, `ServicoDeDesfazimento`, `ServicoDeTransferenciasEPares`, `ServicoDeIgnoradas`, `ServicoDeRegras`, `ServicoDeConsultaDeConciliacao`) |
| Regras de negócio da U3a (RC-nn) | `aidlc-docs/construction/u3a-conciliacao-importacao/functional-design/business-rules.md` |
| Regras de negócio da U3b (BR-xx-nn) e resumo da construção | `aidlc-docs/construction/u3b-conciliacao-motor-vinculos/` (`functional-design/business-rules.md`, `code/code-summary.md`) |
| Decisão de arquitetura da U3b | [ADR 013](adr/013-vinculo-ledger-insert-only-e-identificador-deterministico.md) |
| Regras oficiais v2 (CB-nn) | `docs/sdd/rules/REGRAS-CONCILIACAO-BANCARIA.md` |
| Desenho da U3b | `aidlc-docs/inception/application-design/` |
