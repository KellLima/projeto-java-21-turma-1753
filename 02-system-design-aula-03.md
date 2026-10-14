# System Design — Banco de Dados em Larga Escala
## Checkout na Black Friday

## 1. Cenário e objetivo

A API de Checkout já foi preparada para atender alta concorrência, mas o volume de requisições passou a sobrecarregar uma única instância do PostgreSQL. As transações ficam aguardando locks, o uso de CPU chega a 100% e a latência p99 alcança 4,8 segundos. O pool HikariCP também se esgota devido a consultas lentas de pedidos sem índice adequado.

O objetivo é definir uma estratégia de armazenamento e concorrência que suporte o crescimento do tráfego sem comprometer a consistência dos pagamentos.

## 2. Estratégia de chave primária: Sequence, UUIDv4 ou UUIDv7?

### Decisão

Adotar **UUIDv7 para identificadores de pedidos e pagamentos**, após validar a implementação utilizada e medir o impacto nos índices do PostgreSQL.

### Justificativa

- **Sequence / BIGINT:** tem tamanho menor e costuma favorecer índices B-tree, mas a geração depende de uma sequência centralizada no banco. Pode continuar sendo uma boa opção quando o banco é a autoridade de geração dos IDs.
- **UUIDv4:** pode ser gerado em diferentes instâncias sem consultar um gerador central, mas sua distribuição aleatória pode aumentar a fragmentação e reduzir a localidade de inserção nos índices B-tree.
- **UUIDv7:** pode ser gerado por diferentes instâncias e inclui um componente temporal, favorecendo a localidade dos índices em comparação com UUIDv4.

UUIDv7 não garante a ordem exata de eventos concorrentes e não deve ser tratado como segredo ou mecanismo de autorização. A decisão deve ser validada por testes de inserção e consulta.

## 3. Estratégia de armazenamento: PostgreSQL, NoSQL e persistência poliglota

### 3.1. PostgreSQL para o núcleo transacional

O PostgreSQL será a fonte oficial dos dados que exigem integridade transacional, incluindo:

- Pedidos e seus estados.
- Pagamentos e referências do provedor.
- Registros de idempotência.
- Reservas de estoque, quando precisarem de transações e restrições relacionais.
- Histórico das transições importantes de estado.

Esses dados se beneficiam de transações ACID, chaves estrangeiras, restrições de unicidade e mecanismos de concorrência do banco.

### 3.2. Quando otimizar o PostgreSQL?

Antes de introduzir outra tecnologia, investigar o plano de execução das consultas, os índices existentes, a seletividade dos filtros, o tempo de execução, as transações longas, os locks e o uso do pool de conexões.

O incidente descrito — latência p99 de 4,8 segundos e esgotamento do HikariCP devido a buscas de pedidos sem índice — deve ser investigado primeiro com métricas e `EXPLAIN ANALYZE`. Um índice apropriado pode reduzir leituras e o tempo de ocupação das conexões, mas deve ser escolhido a partir do plano real.

### 3.3. Quando considerar NoSQL?

A escolha depende do padrão de acesso e dos requisitos dos dados:

- **Documental:** dados semiestruturados e documentos acessados como unidades, quando o modelo documental trouxer benefício.
- **Key-value:** cache, sessões ou consultas por chave com baixa latência. Redis pode atender a esse tipo de uso, mas não deve ser a única fonte de verdade de um pagamento.
- **Colunar / wide-column:** grandes volumes distribuídos com padrões de consulta previsíveis e necessidade de alta vazão.
- **Grafos:** consultas centradas em relacionamentos complexos, como redes de relacionamento e recomendações.

NoSQL não é automaticamente mais escalável para todos os casos. A tecnologia deve ser escolhida conforme o volume, o padrão de leitura e escrita, a consistência necessária e o custo operacional.

### 3.4. Persistência poliglota

Persistência poliglota consiste em combinar diferentes tecnologias de armazenamento quando cada uma resolve uma necessidade concreta.

Para este cenário, PostgreSQL será o núcleo transacional. Um armazenamento key-value, como Redis, pode ser introduzido para cache; outros bancos podem atender catálogo, pesquisa ou análises se os requisitos justificarem. A introdução de múltiplos bancos aumenta a complexidade de operação e consistência, portanto não deve ser feita apenas por antecipação.

## 4. Fronteira de consistência: ACID e consistência eventual

### 4.1. Onde ACID é inegociável?

A consistência transacional é essencial para:

- Registrar pedidos e pagamentos de forma coerente.
- Impedir duplicidade por meio de restrições de unicidade e controle de idempotência.
- Garantir transições válidas de estado.
- Controlar a reserva e a baixa de estoque para que a quantidade disponível não fique negativa.
- Persistir juntas as alterações locais que precisam ser atômicas.

Uma transação local no PostgreSQL não torna atômica uma operação que também envolve um provedor externo de pagamento. Quando o resultado externo é desconhecido, o sistema deve manter a operação identificável e reconciliar seu estado antes de repetir uma cobrança.

### 4.2. Onde a consistência eventual é aceitável?

Pode ser utilizada em informações que toleram atraso na atualização, como:

- Dashboards e relatórios.
- Projeções de leitura.
- Índices de pesquisa derivados.
- Métricas e indicadores analíticos.
- Cache de dados cuja defasagem seja aceitável.

O estado financeiro oficial e as decisões críticas de reserva de estoque não devem depender exclusivamente de uma projeção eventualmente consistente se isso permitir pagamentos duplicados ou venda de estoque indisponível.

## 5. Estratégia de concorrência: status do pagamento versus estoque

### 5.1. Status do pedido ou pagamento: lock otimista

Para atualizações de status com baixa contenção, utilizar inicialmente **lock otimista**, por exemplo com `@Version` no JPA.

```java
@Entity
public class Pagamento {

    @Id
    private UUID id;

    @Version
    private Long versao;

    private String status;
}
```

O campo de versão permite detectar se outra transação alterou o registro desde sua leitura. Em caso de conflito, a aplicação deve reler o estado e decidir se a operação ainda é válida, em vez de sobrescrever a alteração concorrente.

**Risco:** podem ocorrer conflitos e `OptimisticLockException`, exigindo tratamento e, quando seguro, nova tentativa limitada.

**Mitigação:** manter transações curtas, tratar conflitos explicitamente e não repetir efeitos financeiros externos sem verificar o estado da operação.

### 5.2. Reserva de estoque: contenção alta

Na reserva de estoque, muitos clientes podem disputar simultaneamente as últimas unidades do mesmo SKU. A estratégia deve impedir que o estoque fique negativo.

Uma opção é utilizar uma atualização atômica condicional no banco:

```sql
UPDATE estoque
SET quantidade = quantidade - 1
WHERE sku = :sku
  AND quantidade > 0;
```

A aplicação deve verificar o número de linhas afetadas: uma linha atualizada indica que a reserva foi efetuada; zero linhas indica que não havia estoque disponível ou que o SKU não foi encontrado.

Quando a operação exigir leitura e várias alterações relacionadas, um lock pessimista (`SELECT ... FOR UPDATE`, equivalente no JPA a `PESSIMISTIC_WRITE`) pode ser apropriado.

**Riscos do lock pessimista:** filas de espera, menor vazão e deadlocks, especialmente quando muitas transações disputam o mesmo SKU.

**Mitigação:** manter transações curtas, adquirir locks em ordem consistente, definir timeouts adequados e monitorar contenção. Não manter locks enquanto a aplicação aguarda uma chamada HTTP externa.

### Resumo da decisão

| Situação | Estratégia inicial | Motivo |
|---|---|---|
| Atualização de status com baixa contenção | Lock otimista (`@Version`) | Evita bloqueios desnecessários e detecta conflitos |
| Reserva de estoque muito disputada | Atualização atômica condicional ou lock pessimista quando necessário | Protege a quantidade disponível sob concorrência |
| Evitar duplicidade de identificadores de operação | Restrição `UNIQUE` | A integridade é aplicada pelo próprio banco |

A escolha final deve ser validada com testes de concorrência e métricas reais.

## 6. Diagnóstico de queries lentas e estratégia de índices

### 6.1. Investigação do incidente

Diante da latência p99 de 4,8 segundos e do esgotamento do HikariCP, seguir este processo:

1. Identificar as consultas com maior latência e frequência usando métricas e logs do banco.
2. Verificar o pool HikariCP: conexões ativas, conexões em espera, tempo de aquisição e duração das operações.
3. Executar `EXPLAIN (ANALYZE, BUFFERS)` para uma consulta representativa em ambiente controlado.
4. Examinar se ocorre `Seq Scan` em uma tabela grande, se o filtro é seletivo e se as linhas estimadas diferem muito das linhas reais.
5. Verificar leituras de buffers, ordenações, joins e possíveis esperas por locks.
6. Criar ou ajustar somente os índices justificados pelo padrão de consulta.
7. Repetir o teste e comparar plano, latência, leituras e vazão de escrita.

Exemplo de diagnóstico:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, status, created_at
FROM pedido
WHERE cliente_id = :clienteId
ORDER BY created_at DESC
LIMIT 20;
```

Se a consulta for frequente e o plano demonstrar benefício, um índice composto em `(cliente_id, created_at DESC)` pode ser adequado. A decisão depende do esquema, da seletividade e do plano real.

`EXPLAIN ANALYZE` executa a consulta. Para comandos de escrita, usar ambiente controlado ou uma transação que possa ser revertida com segurança.

### 6.2. Indexar todas as colunas seria uma boa decisão?

**Não.** Criar índices indiscriminadamente pode:

- Aumentar o custo de `INSERT`, `UPDATE` e `DELETE`, pois os índices também precisam ser mantidos.
- Consumir mais armazenamento e memória.
- Aumentar o trabalho do banco durante alterações.
- Criar índices redundantes ou pouco seletivos que não ajudam nas consultas relevantes.

Os índices devem ser orientados pelas consultas reais, pelos filtros, joins e ordenações mais importantes. Cada índice precisa justificar seu custo de escrita e armazenamento.

## 7. Validação

A arquitetura deve ser validada por:

- Testes concorrentes de atualização de status e reserva de estoque.
- Testes que comprovem que o estoque não fica negativo.
- Comparação de inserções com Sequence, UUIDv4 e UUIDv7.
- Análise de planos com `EXPLAIN ANALYZE`.
- Medição de latência p95/p99, uso de CPU, locks e conexões HikariCP.
- Testes de carga representando o tráfego esperado da Black Friday.

## 8. Conclusão

Para o Checkout na Black Friday, a proposta é manter o PostgreSQL como fonte oficial dos dados transacionais, avaliar UUIDv7 para IDs distribuídos, preservar ACID nas operações financeiras e utilizar consistência eventual em projeções e análises.

O lock otimista é a primeira opção para atualizações de status com baixa contenção. Para estoque, a atualização condicional atômica ou o lock pessimista pode ser necessário devido à disputa simultânea pelas mesmas unidades.

O incidente de latência deve ser investigado com métricas e `EXPLAIN ANALYZE`, corrigindo índices com base em evidências em vez de indexar todas as colunas. NoSQL e persistência poliglota devem ser adotados apenas quando um padrão de acesso concreto justificar a complexidade adicional.
