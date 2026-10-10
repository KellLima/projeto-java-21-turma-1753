# Cenários de Risco — API de Checkout

## 1. Pagamento aprovado sem confirmação

**Pergunta:** E se o provedor de pagamento aprovar a cobrança, mas a resposta se perder e o cliente repetir a compra em outra instância da API?

**Risco:** O sistema pode processar a mesma operação mais de uma vez, causando cobranças duplicadas ou divergência entre o estado do pagamento e o estado do pedido.

**Solução:** utilizar idempotência persistente, controle de concorrência e a chave de idempotência do provedor, quando suportada. Se o resultado for desconhecido, consultar 
ou reconciliar a transação antes de repetir a cobrança.

**Resultado esperado:** a repetição da requisição não cria uma segunda cobrança e o estado do pagamento pode ser recuperado.


## 2. Banco de dados indisponível

**Pergunta:** E se o provedor aprovar o pagamento, mas o banco de dados ficar indisponível antes de registrarmos a confirmação?

**Risco:** O pagamento pode ter sido efetivado externamente sem que o Checkout consiga persistir o resultado, deixando a operação em um estado inconsistente.

**Solução:** não interpretar a falha de persistência como falha definitiva do pagamento. Manter meios de identificar a operação pendente, recuperar seu estado junto ao 
provedor e reconciliar os dados após a recuperação do banco.

Se não for possível registrar a confirmação, a API não deve afirmar que a cobrança falhou. O resultado deve permanecer incerto até que seja possível confirmá-lo.

**Resultado esperado:** o sistema recupera o estado correto da operação sem cobrar novamente de forma indevida.


## 3. Sobrecarga durante a recuperação

**Pergunta:** E se o serviço de pagamento voltar após uma indisponibilidade e receber simultaneamente milhares de novas requisições e retentativas acumuladas?

**Risco:** O volume repentino pode sobrecarregar o provedor e provocar novas falhas, mesmo após a recuperação inicial do serviço.

**Solução:** combinar Circuit Breaker, Retry com backoff exponencial e jitter, Bulkhead e RateLimiter.

O sistema deve limitar a concorrência e a taxa de chamadas, evitando liberar todas as tentativas simultaneamente.

**Resultado esperado:** o provedor recebe uma carga controlada durante a recuperação, reduzindo a possibilidade de uma nova sobrecarga.
