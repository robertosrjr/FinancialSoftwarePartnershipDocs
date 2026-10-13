# ADR 003 — Persistência em fases: memória, PostgreSQL local e AWS

- **Status**: Aceita · **Data**: 2026-10-01
- **Origem**: NFR Requirements U1, C1 = X e C2 = A

## Contexto
A estação de desenvolvimento não tem runtime de containers, e o PostgreSQL ainda não está instalado. A decisão do usuário foi começar com tudo em memória. O produto, porém, é de mercado e não pode ir para produção sem persistência durável, RLS e triggers de imutabilidade.

## Decisão
- **Fase A (atual)**: todas as portas de saída têm adaptadores em memória, marcados com `AdaptadorEmMemoria`. Domínio, casos de uso, portas e API já são os definitivos.
- **Fase B**: adaptadores JPA, Flyway, RLS e triggers no PostgreSQL local. Os testes de integração com banco rodam no CI, com Testcontainers.
- **Fase C**: RDS PostgreSQL Multi-AZ, S3, KMS e Secrets Manager, com imagem OCI em EC2.
- **Gate de produção**: no perfil `prod`, a aplicação não inicia se houver algum `AdaptadorEmMemoria`. Um teste automatizado garante isso.
- Dependências da Fase B (JPA, Flyway, Testcontainers) **não entram no POM antes da hora** (SECURITY-10).

## Alternativa descartada
- **H2 como banco em memória**: daria uma falsa cobertura de RLS e triggers, que o H2 não reproduz.

## Consequências
- (+) O desenvolvimento avança sem infraestrutura, e a troca de fase mexe só em `infrastructure`.
- (−) O RLS e o append-only no banco ficam adiados. Até lá, o controle compensatório é o isolamento na aplicação, coberto por testes, e o repositório sem métodos de alteração, verificado com ArchUnit.
- (−) Na Fase A o estado vive dentro do processo, então só é possível rodar uma instância.
