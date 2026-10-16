# System Design: Aula 04 — Microsserviços & Padrão Saga

> **Contexto:** compra distribuída na Black Friday. O pagamento Pix de uma Smart TV de R$ 3.000 foi aprovado, mas o serviço de Estoque rejeitou a reserva porque o SKU esgotou. Checkout, Pagamento e Estoque têm bancos independentes, sem uma única transação `@Transactional`. O cenário considera 5.000 TPS no pico e milhões de consultas a “Meus Pedidos”.

## 1. Transações Distribuídas: Coordenação e Trade-offs

### 1.1 Volumetria e pragmatismo: 2PC vs. Saga

Uma transação ACID local é mais simples quando todas as alterações pertencem ao mesmo banco e podem ser confirmadas ou revertidas em uma única transação. O Two-Phase Commit (2PC) pode ser adequado quando a atomicidade entre poucos participantes é indispensável e a carga é compatível com o custo de coordenação.

No cenário da Black Friday, Checkout, Pagamento e Estoque são serviços autônomos, cada um com seu banco, e o pico chega a 5.000 TPS. O 2PC acrescentaria comunicação coordenada, espera pela decisão de commit e potencial retenção de locks ou outros recursos durante a preparação. Falhas de rede ou do coordenador também podem deixar participantes aguardando a decisão. Isso pode elevar a latência, aumentar a contenção e reduzir a disponibilidade durante o pico.

Não existe um limite universal de TPS que torne o 2PC inviável: isso depende da duração das transações, quantidade de participantes, latência de rede, implementação e capacidade da infraestrutura. Neste cenário, escolheria uma Saga porque os serviços são autônomos e o processo precisa se recuperar de falhas parciais. Cada serviço confirma sua transação local e, se uma etapa posterior falhar, o fluxo executa uma ação compensatória. Isso oferece consistência eventual, não uma transação ACID global.

### 1.2 Modelo de coordenação da Saga

- [ ] Coreografia (orientada a eventos)
- [x] **Orquestração (Process Manager / máquina de estados)**

O fluxo possui uma sequência crítica: criar pedido, reservar estoque, processar pagamento e confirmar o pedido. Também exige compensação automática quando o pagamento é aprovado, mas a reserva falha. Um orquestrador explícito facilita acompanhar o estado, identificar a etapa que falhou e decidir qual compensação executar.

O **Orquestrador da Saga de Checkout** mantém o estado durável do processo e envia comandos aos serviços participantes por mensageria. Os serviços respondem por eventos. O orquestrador não executa a lógica interna dos serviços: Estoque continua responsável por reservar/liberar unidades e Pagamento por cobrar/estornar.

Para evitar que o orquestrador vire um “serviço Deus”:
- manter nele somente a coordenação e a máquina de estados;
- deixar regras específicas dentro dos serviços donos do domínio;
- usar contratos de eventos versionados e explícitos;
- persistir o estado da Saga e permitir retomada após reinicialização;
- monitorar Sagas em andamento, falhas e compensações pendentes;
- garantir idempotência nos comandos e consumidores.

A comunicação será predominantemente assíncrona. Isso evita manter uma requisição HTTP aberta durante todo o fluxo distribuído, mas exige lidar com mensagens duplicadas, estados intermediários e consistência eventual.

## 2. Máquina de Estados, Compensação e Confiabilidade

### 2.1 Máquina de estados e fluxos ponta a ponta

#### Estados do pedido

- `RECEIVED`: pedido recebido.
- `RESERVING_STOCK`: reserva de estoque em andamento.
- `STOCK_RESERVED`: estoque reservado.
- `PAYMENT_PENDING`: pagamento em andamento.
- `PAYMENT_APPROVED`: pagamento aprovado pelo provedor.
- `CONFIRMED`: pedido confirmado; fluxo concluído.
- `COMPENSATION_PENDING`: compensação necessária e ainda em andamento.
- `REFUND_PENDING`: estorno aguardando confirmação.
- `CANCELLED_REFUNDED`: pedido cancelado e estorno confirmado.
- `CANCELLED_NO_CHARGE`: pedido cancelado sem cobrança efetivada.
- `MANUAL_REVIEW`: estado excepcional que exige análise operacional quando a recuperação automática não resolve o caso.

As transições devem ser explícitas e validadas. `CONFIRMED` só deve ser alcançado quando o estoque estiver reservado e o pagamento aprovado. O pedido não deve ser marcado como `CANCELLED_REFUNDED` antes da confirmação do estorno. Se o resultado de uma chamada externa for desconhecido, a operação permanece pendente de consulta ou reconciliação.

#### Caminho feliz

1. Checkout cria o pedido em `RECEIVED` e persiste o estado inicial da Saga.
2. O orquestrador solicita a reserva ao Estoque.
3. Estoque valida a disponibilidade, reserva a unidade em sua transação local e publica `StockReserved`.
4. O orquestrador solicita o pagamento.
5. Pagamento processa o Pix e publica `PaymentApproved` após confirmar o resultado com o provedor.
6. O orquestrador conclui a Saga em `CONFIRMED`.
7. O modelo de leitura de “Meus Pedidos” é atualizado por eventos.

Cada serviço confirma suas alterações localmente; não há uma transação ACID única envolvendo todos os bancos.

#### Caminho de falha e compensação

Para reduzir a possibilidade de cobrar por um produto indisponível, a sequência preferencial é reservar o estoque antes de solicitar o pagamento. Se Estoque rejeitar a reserva, o pedido pode terminar em `CANCELLED_NO_CHARGE`, sem iniciar a cobrança.

No incidente descrito, porém, o pagamento já foi aprovado. Nesse caso:

1. O orquestrador registra `COMPENSATION_PENDING`.
2. Envia uma solicitação de estorno ao serviço de Pagamento com um identificador único de compensação.
3. Pagamento solicita o estorno ao provedor e registra o resultado.
4. Se o estorno for confirmado, publica `RefundConfirmed`; o orquestrador marca o pedido como `CANCELLED_REFUNDED`.
5. Se o estorno falhar ou ficar incerto, o pedido permanece em `REFUND_PENDING` ou `COMPENSATION_PENDING`, com retentativas controladas e reconciliação.
6. A Saga só termina como compensada depois da confirmação do estorno.

### 2.2 Resiliência da compensação e consistência

#### E se a compensação falhar ou oscilar?

Um timeout não prova que o estorno falhou: o provedor pode ter processado a solicitação sem que a resposta chegasse à aplicação.

A estratégia será:
- atribuir um identificador único à operação compensatória e persistir seu estado;
- reutilizar o mesmo identificador lógico nas retentativas, quando suportado pelo provedor;
- aplicar retentativas limitadas com backoff exponencial e jitter para falhas transitórias;
- usar timeout e Circuit Breaker para limitar espera e evitar insistência contínua em uma dependência instável;
- consultar o estado do estorno ou reconciliar a operação antes de enviar outra solicitação quando o resultado for desconhecido;
- manter a Saga pendente e emitir alertas se o prazo operacional for excedido;
- encaminhar para `MANUAL_REVIEW` casos que não possam ser resolvidos automaticamente.

A idempotência impede que a mesma mensagem cause múltiplos efeitos de negócio. O estado do estorno deve ser durável e observável. Assim, o cliente não fica silenciosamente sem produto e sem acompanhamento do dinheiro.

#### Atomicidade entre estado e mensageria: Transactional Outbox

Publicar um evento diretamente depois de gravar no banco cria o problema de Dual Write: o banco pode confirmar a alteração e a publicação falhar, ou o evento pode ser publicado e a transação local falhar.

Para evitar a perda silenciosa de eventos, cada serviço utiliza **Transactional Outbox**:

1. Na mesma transação local, atualiza o estado de negócio e insere o evento em uma tabela `outbox`.
2. Se a transação confirmar, ambos os registros ficam persistidos; se falhar, ambos são revertidos.
3. Um publicador separado ou mecanismo CDC, como Debezium, lê a Outbox e publica os eventos no broker.
4. Após falha de rede ou broker, o evento permanece disponível para nova tentativa.
5. Os consumidores registram os eventos processados e aplicam efeitos de forma idempotente.

A Outbox evita a janela de perda entre a gravação local e o registro do evento, mas não garante entrega exatamente uma vez de ponta a ponta. Uma publicação pode ser repetida; os consumidores precisam tolerar duplicatas.

## 3. Desacoplamento com CQRS e Estratégia de Testes

### 3.1 Consultas em alta vazão com CQRS

Milhões de clientes consultando “Meus Pedidos” podem competir por CPU, conexões e I/O com as transações que registram pagamentos e reservas.

Adotaria **CQRS** para separar o modelo de escrita do modelo de leitura:

- **Command / escrita:** os serviços validam comandos e persistem alterações transacionais em seus bancos.
- **Publicação de eventos:** alterações relevantes geram eventos por meio da Transactional Outbox.
- **Query / leitura:** consumidores atualizam uma projeção otimizada para consultar pedidos por cliente e exibir o status.
- **Escala independente:** a camada de consulta pode ser replicada ou usar armazenamento apropriado para leitura, sem aumentar na mesma proporção a carga do banco transacional.

A projeção pode ficar temporariamente atrasada porque sua atualização é assíncrona. A API deve exibir estados como `PROCESSING`, `PAYMENT_PENDING` ou `REFUND_PENDING`, em vez de sugerir que a operação terminou. Para uma consulta imediatamente após o comando, pode devolver o estado confirmado pela escrita ou um identificador para acompanhar o processamento.

A consistência eventual é aceitável para a projeção, mas não deve ser usada para autorizar uma cobrança ou confirmar estoque com base em informação desatualizada. Monitoraria o atraso da projeção, a taxa de eventos processados, erros de consumo e a latência p95/p99 das consultas.

### 3.2 Estratégia de testes da transação distribuída

O objetivo é comprovar que a Saga executa a compensação quando o pagamento é aprovado e o estoque falha, inclusive diante de mensagens duplicadas e indisponibilidades.

#### Teste automatizado principal

1. Subir os componentes necessários em ambiente de integração, com bancos isolados por serviço e broker de teste.
2. Configurar o provedor de pagamento simulado para aprovar a cobrança.
3. Configurar Estoque para rejeitar a reserva com `SKU_OUT_OF_STOCK`.
4. Iniciar um pedido e acompanhar os eventos e estados da Saga.
5. Verificar que o orquestrador registra `COMPENSATION_PENDING` e envia a solicitação de estorno.
6. Simular a confirmação do estorno pelo provedor.
7. Verificar que o pedido termina em `CANCELLED_REFUNDED`, que o estorno foi solicitado apenas uma vez por operação lógica e que os eventos foram persistidos e publicados.
8. Reenviar o evento de falha de estoque e confirmar que a compensação não é executada novamente de forma indevida.

#### Testes adicionais de falha

- **Broker indisponível após o commit local:** confirmar que o evento continua na Outbox e é publicado após a recuperação.
- **Mensagem duplicada:** publicar o mesmo evento mais de uma vez e validar a idempotência do consumidor.
- **Timeout no estorno:** simular que o provedor processa o estorno, mas a resposta se perde; verificar que o sistema consulta ou reconcilia o resultado antes de repetir.
- **Falha transitória do estorno:** verificar retentativas limitadas, backoff e manutenção do estado pendente.
- **Indisponibilidade prolongada:** validar alertas e encaminhamento para revisão operacional.
- **Reinicialização do orquestrador:** interromper o processo no meio da Saga e comprovar que ele retoma a partir do estado persistido.

Esses testes devem rodar na pipeline de CI. As asserções precisam verificar estados persistidos, efeitos de negócio e eventos, não apenas códigos HTTP. Métricas úteis incluem tempo de conclusão da Saga, compensações pendentes, falhas de publicação da Outbox, retentativas e idade das operações em estado intermediário.

## Conclusão

Para a compra distribuída na Black Friday, a proposta é usar uma Saga orquestrada com estado durável, compensação automática e Transactional Outbox. O fluxo reserva o estoque antes de cobrar sempre que possível; se a cobrança já tiver sido aprovada e a reserva falhar, inicia o estorno e mantém o pedido em estado intermediário até confirmar o resultado.

CQRS separa a carga massiva de consultas das transações críticas. A consistência eventual é aceita no modelo de leitura, enquanto cada serviço mantém integridade ACID em suas transações locais. Testes de falha na pipeline comprovam recuperação, idempotência e ausência de perda silenciosa de eventos.
