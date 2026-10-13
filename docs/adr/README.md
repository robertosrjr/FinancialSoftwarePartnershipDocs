# Architecture Decision Records

| # | Decisão | Status | Data |
|---|---|---|---|
| [001](001-monolito-modular-hexagonal.md) | Monolito modular com Spring Modulith e arquitetura hexagonal por módulo | Aceita | 2026-10-01 |
| [002](002-auditoria-cadeia-hmac-crypto-shredding.md) | Trilha de auditoria com cadeia HMAC por tenant e crypto-shredding | Aceita | 2026-10-01 |
| [003](003-persistencia-em-fases.md) | Persistência em fases: memória, PostgreSQL local e AWS | Aceita | 2026-10-01 |
| [004](004-maker-checker-revisor-unico.md) | Segregação de funções com modo estrito e exceção de revisor único | Aceita | 2026-10-01 |
| [005](005-indice-cego-numero-de-conta-nao-armazenado.md) | Índice cego: o número da conta bancária não é armazenado | Aceita | 2026-10-02 |
| [006](006-deduplicacao-por-fitid-com-conferencia-de-conteudo.md) | Deduplicação por FITID com conferência de conteúdo e ordem de ocorrência | Aceita | 2026-10-02 |
| [008](008-idempotencia-de-escrita.md) | Idempotência de escrita por `Idempotency-Key`, na plataforma | Aceita | 2026-10-03 |
| [011](011-retencao-e-eliminacao-de-dados-pessoais.md) | Retenção e eliminação de dados pessoais da conciliação (prazos a validar) | Proposta | 2026-10-03 |
| [012](012-baixas-imutaveis-situacao-derivada-e-travas.md) | Baixas imutáveis, situação derivada e travas por lançamento e por conta | Aceita | 2026-10-03 |
| [013](013-vinculo-ledger-insert-only-e-identificador-deterministico.md) | Vínculo, ledger insert-only e identificador determinístico na conciliação | Aceita | 2026-10-04 |
| [014](014-vinculo-a-baixa-existente.md) | Vínculo a uma baixa que já existe: a transação do banco não duplica o pagamento | Aceita | 2026-10-06 |
| [015](015-ofx-permanece-adaptador-de-entrada-da-conciliacao.md) | OFX permanece adaptador de entrada da conciliação; revisão pela origem só no agregador | Aceita | 2026-10-09 |
