# ADR 004 — Segregação de funções com modo estrito e exceção de revisor único

- **Status**: Aceita · **Data**: 2026-10-01
- **Origem**: FR-120, FR-122, FR-123, US-048, risco R-01; Functional Design U1 Q2 = A

## Contexto
As normas de compliance pedem maker-checker: quem origina não aprova. O público principal, porém, são MEIs com um único usuário. Aplicar um maker-checker estrito em todo tenant travaria o produto para esses clientes.

## Decisão
- A **política é centralizada** na `plataforma` (`PoliticaDeSegregacaoApi`). O estado da proposta fica no agregado de cada módulo.
- **Modo estrito**: liga **automaticamente** quando o tenant tem 2 ou mais Admins ativos. A partir daí, o originador nunca confirma o próprio item, e uma tentativa responde 403 `sod_violation`, com auditoria.
- **Modo revisor único**: é a exceção documentada para quando não existe revisor independente. A auto-aprovação é marcada no ledger como `AUTO_APROVACAO_SEM_REVISOR` e entra no relatório de controle compensatório.
- **O controle nunca é afrouxado automaticamente**: voltar para revisor único exige um Admin com MFA, um motivo de pelo menos 20 caracteres, auditoria e alerta. Desativar um Admin não altera o modo.

## Consequências
- (+) O controle é auditável e explicável para auditores, e não trava o MEI.
- (−) No modo revisor único, um erro do Admin só é detectado depois (pelo relatório) e corrigido pelo desfazer auditado. Esse é o risco residual aceito em R-01.
- (−) A retirada do papel ADMIN pelo provedor de identidade pode deixar o tenant sem Admin. A aplicação não bloqueia isso, porque o IdP é a fonte da verdade sobre papéis. Esse comportamento foi detectado pelo teste de propriedade e registrado.
