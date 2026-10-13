# ADR 002 — Trilha de auditoria com cadeia HMAC por tenant e crypto-shredding

- **Status**: Aceita · **Data**: 2026-10-01
- **Origem**: NFR-10/11, financial-compliance-skill (ITGC OP-1), Functional Design U1 Q1 = A

## Contexto
O produto precisa provar que registros financeiros não foram adulterados (ITGC, BCB 130). Ao mesmo tempo, a LGPD garante o direito de eliminação, com exceção para o cumprimento de obrigação legal (art. 16, I). Um log imutável comum resolve uma exigência e quebra a outra.

## Decisão
- Cada tenant tem uma **cadeia própria**: sequencial contínuo e `hash = HMAC-SHA256(chave, conteúdo canônico + hash anterior)`. A chave HMAC fica no cofre e é versionada.
- O conteúdo dos campos `anterior`, `novo` e `motivo` é **cifrado com AES-256-GCM usando uma chave por tenant**. A cadeia assina o texto cifrado.
- Durante a retenção legal (5 anos, por padrão), a eliminação é negada. Depois dela, a chave do tenant é destruída (**crypto-shredding**): o conteúdo fica ilegível e a cadeia continua verificável.
- A verificação é incremental (com checkpoints) e completa, roda em job agendado e também sob demanda, e qualquer quebra gera alerta e auditoria.

## Alternativas descartadas
- **Guardar só hashes do conteúdo**: perde o valor de evidência, porque o auditor não vê o que mudou.
- **Conteúdo em claro**: deixa dados financeiros expostos em backups por anos (LC 105, LGPD art. 46).
- **SHA-256 sem chave**: quem tem acesso ao banco pode recalcular a cadeia inteira.

## Consequências
- (+) A adulteração é detectável mesmo por quem tem acesso direto ao banco.
- (−) A gravação é serializada por tenant (lock). No porte de uma PME isso não é gargalo, e o desempenho está medido nos requisitos (U1-P2).
- (−) A destruição de chave exige um procedimento operacional com dupla aprovação, que será definido na Fase C.
