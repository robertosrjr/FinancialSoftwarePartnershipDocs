# ADR 001 — Monolito modular com Spring Modulith e arquitetura hexagonal por módulo

- **Status**: Aceita · **Data**: 2026-10-01 · **Decisores**: Roberto (arquiteto)
- **Origem**: requisitos Q7 e Q8; Application Design Q1 = A

## Contexto
O produto tem seis serviços de negócio previstos: identidade, fluxo de caixa, DRE, conciliação, CRM e IA. Na primeira entrega, o time é pequeno, a carga é moderada (até 1.000 tenants) e o deploy é um só. Microsserviços trariam custo operacional sem benefício nesse momento. Já um monolito sem fronteiras explícitas tende a virar um acoplamento difícil de desfazer.

## Decisão
- O projeto é **Maven multi-módulo** (`shared-kernel`, `plataforma`, um módulo por contexto de negócio, `app`), validado pelo **Spring Modulith**.
- Cada módulo publica só um pacote `api`. O resto é interno, e `ApplicationModules.verify()` faz o build falhar se essa fronteira for violada.
- Dentro de cada módulo, a organização é **hexagonal por pacote**: `domain` em Java puro, `application` com portas e casos de uso, `infrastructure` com adaptadores e Spring. O ArchUnit verifica a regra de dependência.
- A comunicação entre módulos acontece por API síncrona, quando a consistência é imediata, e por eventos de domínio para efeitos colaterais.

## Consequências
- (+) Um módulo pode virar serviço depois, trocando só os adaptadores e a forma de comunicação.
- (+) O domínio fica testável sem Spring: unitários e PBT rodam em milissegundos.
- (−) Há mais código de mapeamento (DTO, domínio e, na Fase B, entidade JPA) e mais POMs.
- (−) A aplicação escala como um bloco só. Isso é aceitável para a carga do primeiro ano.
