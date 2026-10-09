# Cenários de Risco — API de Checkout

## 1. Pagamento aprovado sem confirmação

**Pergunta:** E se o provedor de pagamento aprovar a cobrança, mas a resposta se perder e o cliente repetir a compra em outra instância da API?

**Risco:** O sistema pode processar a mesma operação mais de uma vez, causando cobranças duplicadas ou divergência entre o estado do pagamento e o estado do pedido.


## 2. Banco de dados indisponível

**Pergunta:** E se o provedor aprovar o pagamento, mas o banco de dados ficar indisponível antes de registrarmos a confirmação?

**Risco:** O pagamento pode ter sido efetivado externamente sem que o Checkout consiga persistir o resultado, deixando a operação em um estado inconsistente.


## 3. Sobrecarga durante a recuperação

**Pergunta:** E se o serviço de pagamento voltar após uma indisponibilidade e receber simultaneamente milhares de novas requisições e retentativas acumuladas?

**Risco:** O volume repentino pode sobrecarregar o provedor e provocar novas falhas, mesmo após a recuperação inicial do serviço.
