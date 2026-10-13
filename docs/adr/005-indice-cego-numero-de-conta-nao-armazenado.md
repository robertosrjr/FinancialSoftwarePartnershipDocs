# ADR 005 — Índice cego: o número da conta bancária não é armazenado

- **Status**: Aceita · **Data**: 2026-10-02
- **Origem**: LGPD, LC 105/2001, NFR-26, US-010, US-024; Functional Design U3a Q1 = B

## Contexto
O sistema precisa reconhecer de qual conta é um extrato OFX (a identificação vem no arquivo) e impedir o cadastro da mesma conta duas vezes. O número da agência e da conta, porém, é dado financeiro sigiloso, e o produto não precisa mostrá-lo nem exportá-lo.

## Decisão
- **Nenhum lugar guarda o número completo** da agência e da conta: nem o banco de dados, nem o log, nem a auditoria, nem a resposta da API.
- A conta guarda **dois índices cegos** e os **4 últimos dígitos** (só para exibir `****1234`):
  - `indiceCompleto` = HMAC-SHA256(chave de pseudonimização, finalidade + tenant + banco + agência + conta);
  - `indiceSemAgencia` = o mesmo sem a agência, para os bancos que não informam `BRANCHID` no OFX.
- Antes do cálculo, banco, agência e conta são **normalizados** (maiúsculas, só letras e dígitos, sem zeros à esquerda). A mesma função vale no cadastro e na leitura do arquivo.
- O tenant e a finalidade entram na mensagem do HMAC: o mesmo número em tenants diferentes gera índices diferentes, e um índice não serve para outra finalidade.
- A capacidade é da `plataforma` (`ProtecaoDeDadosApi.indiceCego`), com a chave atrás da porta de chaves (local na Fase A, KMS na Fase C).
- Quando o arquivo é de uma conta não cadastrada, a resposta traz só o **banco e os 4 últimos dígitos** do próprio arquivo enviado.

## Consequências
- (+) Um vazamento do banco de dados não expõe números de conta utilizáveis.
- (+) A unicidade e o reconhecimento funcionam por comparação de igualdade, sem ler o número.
- (−) **Rotacionar a chave de pseudonimização exige reindexar as contas**, e isso só é possível com o número em mãos. Procedimento a definir na Infrastructure Design: pedir ao Admin que reconfirme as contas, ou recalcular a partir do próximo arquivo recebido de cada conta.
- (−) Dois índices significam que, sem agência no arquivo, mais de uma conta pode casar (`conta_ambigua`); a API aceita o `contaId` para resolver.
- (−) Um HMAC de um número curto (poucos dígitos) é vulnerável a força bruta por quem tenha a chave. Por isso a chave é tratada como segredo de alto valor (SECURITY-12).

## Nota (2026-10-10): CNPJ alfanumérico (D-49)
O mesmo princípio vale para o documento da contraparte (CPF e CNPJ), guardado como índice cego mais os 4 últimos caracteres. Com o CNPJ alfanumérico (IN RFB 2.229/2024), a normalização passa a ser: maiúsculas, só letras e dígitos, dígito verificador pelo módulo 11 com o valor de cada caractere igual ao código ASCII − 48. É a mesma função no fluxo de caixa (`DocumentoProtegido`) e na conciliação (`ExtratorDeDocumentoEChaves`), porque o motor compara os índices dos dois lados. Documentos numéricos continuam com o mesmo índice de antes (teste de regressão), então nada já gravado deixa de ser encontrado.
