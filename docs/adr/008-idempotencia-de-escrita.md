# ADR 008 — Idempotência de escrita por `Idempotency-Key`, na plataforma

- **Status**: Aceita · **Data**: 2026-10-03
- **Origem**: US-003, FR-143, FC-04, CB-04, NFR-65, V2-Q13, decisão D11; Functional Design e NFR Requirements da U1b

## Contexto
Um cliente que não recebe a resposta de uma escrita (queda de rede, timeout, clique duplo) reenvia. Sem proteção, o reenvio cria um segundo lançamento, uma segunda baixa ou uma segunda importação. Num produto financeiro, um registro duplicado só se corrige por estorno (ITG 2000 R1), então evitar a duplicidade é melhor que corrigi-la. Os três módulos (plataforma, fluxo de caixa, conciliação) precisam do mesmo comportamento.

## Decisão
Toda escrita da API (`POST`, `PUT`, `PATCH` em `/api/v1`) exige o cabeçalho `Idempotency-Key`, tratado por **um filtro único na `plataforma`**, dentro da cadeia de segurança, depois de o usuário e o tenant estarem no contexto.

- **Chave**: de 8 a 128 caracteres entre letras, dígitos e `. _ : -`. Sem o cabeçalho, 400; fora do formato, 400.
- **Identidade**: (tenant, usuário, chave). A mesma chave de outro usuário ou tenant é outra chave. O armazém indexa por um **HMAC** da identidade; a chave bruta nunca é guardada nem logada.
- **Impressão da operação**: SHA-256 de método, caminho, consulta ordenada, tipo de conteúdo e corpo **normalizado** (JSON com chaves ordenadas; multipart por nome, tipo e SHA-256 do conteúdo, sem o separador nem o nome do arquivo). A mesma chave com outra impressão é 409 `idempotency_key_reutilizada`.
- **Reserva atômica**. Quem chega com a mesma chave e a mesma impressão enquanto a primeira executa **espera** (até 30 s) e recebe a mesma resposta; passado o prazo, 409 `requisicao_em_andamento` com `Retry-After`. A escrita acontece uma única vez.
- **Só respostas 2xx ficam guardadas**; qualquer outra resposta ou exceção **libera** a chave. Uma escrita que falhou não mudou o estado e pode ser repetida; guardar um `403 mfa_required` ou um `429` prenderia o usuário num erro que ele já corrigiu.
- **Reenvio**: devolve o mesmo status, corpo, `Content-Type` e `Location`, com `Idempotent-Replayed: true`, **sem** executar o caso de uso, gravar, auditar nem publicar evento. Não repete papel e MFA (não produz efeito novo). Conta no limite de taxa.
- **Resposta guardada cifrada** com a chave do tenant (a mesma da auditoria), até 1 MB, por 24 horas. Destruída a chave, fica ilegível.
- **Reserva abandonada** (nenhuma conclusão em 5 minutos) conta como ausente; o token da reserva antiga não conclui mais nada.
- **Falha fechada**: armazém cheio ou indisponível recusa a escrita com 503. Nunca se executa uma escrita que não pode ser registrada.
- **Chaves naturais continuam**: hash do arquivo na importação, identificador da alocação entre módulos, unicidade do identificador externo. A idempotência cobre o resto; é a segunda barreira.

## Consequências
- (+) Um comportamento só, testado uma vez, para os três módulos; um oráculo de cobertura quebra o build se um endpoint de escrita novo escapar.
- (+) O reenvio do upload do extrato funciona mesmo com outro separador multipart, que as bibliotecas de HTTP geram a cada tentativa.
- (+) Sem dado pessoal novo guardado: chave irreversível, resposta cifrada, vida de 24 horas.
- (−) **Janela na Fase A**: o armazém e a escrita não compartilham transação. Com persistência em memória é inofensivo (tudo some junto). Na **Fase B**, a reserva e a resposta passam a ser gravadas **na mesma transação da escrita**, com a chave única do banco arbitrando a corrida; isso é requisito da NFR Design.
- (−) O armazém em memória tem tetos (registros, bytes, esperas); acima deles chaves novas são recusadas, por design.
- (−) Todos os clientes e testes de integração passam a enviar a chave (um auxiliar comum de teste cuida disso; nenhuma regra de negócio mudou).
- (−) O reenvio devolve ao mesmo usuário uma resposta antiga mesmo que o papel dele tenha sido retirado nas últimas 24 horas; não há efeito novo. Decisão aceita.

## Alternativas rejeitadas
- **Biblioteca de terceiros**: não cobre multipart normalizado, identidade por tenant e usuário, cifra por tenant nem o contrato de erros do produto.
- **Impressão pelos bytes exatos do corpo**: falso 409 no reenvio de upload e em JSON reformatado.
- **Guardar também erros**: prende o usuário em `mfa_required`, `429` e `503`.
- **Guardar a chave bruta ou a resposta em claro**: dado pessoal em armazém de vida curta sem necessidade.
- **Idempotência dentro de cada módulo**: três implementações e três comportamentos.
