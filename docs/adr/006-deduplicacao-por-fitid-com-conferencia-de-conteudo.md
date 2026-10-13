# ADR 006 — Deduplicação por FITID com conferência de conteúdo e ordem de ocorrência

- **Status**: Aceita (revisada em 2026-10-03: chave versionada, FITID como hash, ordinal por ocorrência de linhas idênticas) · **Data**: 2026-10-02
- **Origem**: US-021, RC-26, RC-27, CB-16, V2-Q5; Functional Design U3a Q2 = A; revisão de arquitetura da U3a (M4)

## Contexto
Reimportar extratos que se sobrepõem não pode duplicar transações, mas também não pode descartar transações legítimas idênticas (duas retiradas de R$ 100,00 no mesmo dia). O `FITID` do OFX deveria ser único por transação, mas alguns bancos o reutilizam, mudam o formato entre exportações ou nem o informam.

## Decisão
Cada linha recebe uma **chave de deduplicação**, única por conta e **versionada** (`v1`):
- Sem FITID: `C:v1:<hash>`, em que o hash é o SHA-256 de `v1 | conta | data | direção | centavos | descrição normalizada | ordinal`. O **ordinal** é o número de ocorrência da linha idêntica (mesma data, direção, valor e descrição normalizada) no mesmo arquivo. Ele conta **ocorrências de linhas idênticas**, não a posição da transação no dia: assim, uma transação nova num dia já exportado não desloca as antigas, e a reimportação do período inteiro reconhece o que já existe. Repetições legítimas são preservadas.
- Com FITID: `F:v1:<sha256 do fitid>`. O FITID entra como hash porque vem cru do arquivo: com `:` no texto, um FITID forjado poderia reproduzir a chave composta de outra linha e fazer uma transação ser descartada em silêncio. Se a chave já existe com **o mesmo conteúdo** (data, direção e valor), a linha é duplicada. Se existe com conteúdo diferente, o FITID foi reutilizado: a linha entra com a chave `FC:v1:<sha256 do fitid>:<hash>` e o aviso `fitid_inconsistente`.
- Uma linha com FITID inédito cujo conteúdo (incluindo a descrição normalizada) já existe em **importações anteriores** da conta entra normalmente com o aviso `possivel_duplicata`. Nada é descartado em silêncio: o usuário decide na revisão.
- O material do hash é montado por **serialização explícita** (`conta.valor()`, `direcao.name()`, data ISO), sem depender de nenhum `toString`.
- As consultas ao repositório são **em lote**: no máximo duas por importação, nunca uma por linha.
- A unicidade (conta, chave) é garantida também pelo repositório, como última barreira, mesmo que a trava por conta falhe.

### Versão da chave
A versão faz parte do material do hash e do formato da chave. **Mudar qualquer um dos dois exige nova versão e migração**: com a mesma versão, todas as transações já gravadas deixariam de ser reconhecidas e o extrato inteiro entraria em duplicidade. Testes com valor congelado (`DeduplicadorTest`) quebram se o formato mudar sem versão nova.

## Consequências
- (+) Reimportar o mesmo arquivo, ou arquivos com períodos sobrepostos, não grava nada de novo (propriedade testada com geração aleatória).
- (+) Transações idênticas legítimas são preservadas.
- (+) Uma mudança acidental de `toString` ou de formato é detectada pelo build.
- (−) Se um banco mudar o texto da descrição entre exportações de uma transação sem FITID, ela entra duas vezes. O aviso `possivel_duplicata` só cobre linhas com FITID; arquivos reais dirão se isso exige ajuste.
- (−) Janelas que cortam um dia ao meio (metade das transações idênticas de um mesmo dia em cada arquivo) podem duplicar uma delas, porque o ordinal é por arquivo. Extratos de bancos fecham o dia inteiro, então o risco é baixo.
- (−) A CB-16 das regras descreve o ordinal como "sequência de ocorrência no mesmo dia" e não prevê a conferência de conteúdo do FITID; a emenda está prevista na Etapa 7 do plano de atualização.
