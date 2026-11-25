# Relatório — Atividade Prática: Espaço de Tuplas com Apache River

**Disciplina:** INE5418
**Aluno:** André Cáceres
**Data:** 24/11/2026

---

## Nível 0 — Rodar e observar

### Observação inicial

1. Qual serviço aparece nos logs primeiro? Por que ele precisa existir antes dos outros?

> O Reggie. Sem ele, produtor e consumidor não conheceriam o endereço do espaço de tuplas.

2. O produtor menciona o nome ou o endereço do consumidor em algum momento? O consumidor menciona o produtor?

> Não para ambos. Mencionam apenas as tuplas e operações.

3. O consumidor começa a processar tarefas antes que o produtor termine de depositar todas? O que isso diz sobre como os dois se coordenam?

> Sim! Isso demonstra que eles podem atuar de forma independente e assíncrona.

---

### Experimento de desacoplamento temporal

4. O consumidor encontrou as tarefas mesmo sendo iniciado depois que o produtor já havia encerrado? O que isso demonstra?

> Sim! Demonstra o desacoplamento temporal.

5. Em comunicação direta via socket, seria possível esse comportamento? Por quê não?

> Não seria possível porque ambos os processos precisariam estar ativos e conectados ao mesmo tempo.

---

## Nível 1 — Inspecionar

### 1.1 As três operações

Preencha a tabela com base no que você observou nos logs:

| Operação River | Equivalente Linda | O que ela faz? | Bloqueia quando não encontra correspondência? | Altera o estado do espaço? |
|---------------|-------------------|----------------|----------------------------------------------|---------------------------|
| `write(entry)` | `OUT` | Adiciona uma tupla ao espaço | Não se aplica. Não é necessário correspondência para executar a operação | Sim |
| `take(template)` | `IN` | Remove uma tupla que case com o padrão e retorna seu valor | Sim | Sim |
| `read(template)` | `RD` | Retorna o valor de uma tupla que case com o padrão sem removê-la do espaço | Sim | Não |

---

### 1.2 O papel do `reggie`

1. Quando o `reggie` caiu, os serviços que já estavam conectados ao espaço continuaram funcionando? Por quê?

> Continuaram funcionando pois já conheciam o espaço.

2. O que aconteceria com um produtor ou consumidor que tentasse iniciar enquanto o `reggie` estivesse fora do ar?

> Não conseguiria realizar operações por não ter conhecimento do espaço

3. Qual sistema moderno cumpre papel equivalente ao `reggie` em uma arquitetura de microsserviços?

> Sistemas de service registry/discovery, como Consul, Kubernetes DNS, Eureka ou etcd

---

### 1.3 Desacoplamento espacial

1. O produtor tem qualquer informação sobre quantos consumidores existem?

> Não.

2. O consumidor tem qualquer informação sobre quem produziu a tarefa que ele retirou?

> Não.

3. Como produtor e consumidor se coordenam se não se conhecem?

> Ambos direcionam suas operações para o espaço de tuplas. Ao consumidor, não importa quem ou como produziu as entradas, desde que estejam lá, e de forma similar, não importa para o produtor quem irá consumi-lás.

---

### 1.4 Comportamento de bloqueio

1. O que o consumidor fez enquanto o espaço estava vazio?

> Ficou bloquado esperando tarefas.

2. Quando o produtor depositou a primeira tarefa, o que aconteceu imediatamente?

> O consumidor foi desbloqueado e passou a consumi-la.

3. Esse comportamento tem nome no modelo Linda. Qual é e por que ele é útil em sistemas distribuídos reais?

> Semântica bloqueante. É útil porque evita espera ocupada e sincroniza naturalmente produtores e consumidores: o consumidor aguarda ocioso e acorda imediatamente quando uma tupla compatível chega ao espaço.

---

### 1.5 Escalabilidade horizontal

1. Uma mesma tarefa foi processada por dois consumidores ao mesmo tempo?

> Não, pois uma vez que é feito o take a tarefa sai do espaço.

2. O produtor precisou ser modificado para suportar dois consumidores?

> Não, conforme explicado na questão 1.3.3.

3. Esse comportamento tem um nome em arquitetura de sistemas. Qual é?

> Escalabilidade horizontal.

---

## Nível 2 — Modificar

### 2.1 Modificação A — Prioridade de tarefas

1. As tarefas de prioridade alta foram processadas antes das de prioridade baixa? Cole um trecho dos logs que evidencie isso:

> Sim.

```
consumidor-1  | [CONSUMIDOR-1] Espaço encontrado. Aguardando tarefas...
produtor-1    | [PRODUTOR] Espaço encontrado via lookup.
produtor-1    | [PRODUTOR] write: TaskEntry{id=1, tipo="calcular", prioridade=2}
produtor-1    | [PRODUTOR] write: TaskEntry{id=2, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=2, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] write: TaskEntry{id=3, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 2
produtor-1    | [PRODUTOR] write: TaskEntry{id=4, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=4, tipo="calcular", prioridade=1}
```

2. O produtor precisou ser modificado para que isso funcionasse?

> Não.

3. Como o consumidor consegue selecionar apenas tarefas de uma prioridade específica? Qual mecanismo do espaço de tuplas torna isso possível?

> Casamento de padrões através de um template (null, null, 1).

---

### 2.2 Modificação B — Serviço monitor

1. Qual operação você usou no monitor — `read()` ou `take()`? Por quê a outra seria problemática?

> `read()`. `take()` produziria o resultado desejado para o monitor, porém, destruiria as tuplas e elas não seriam lidas pelo consumidor.

2. Como você contou as tarefas pendentes usando apenas `read()`? Que limitação isso revela?

> Foi utilizado um hashmap para guardar os ids das tarefas já contadas. A cada leitura, era necessário verificar se a tupla já havia sido contada. Isso mostra a ausência de uma forma de iterar pelos elementos do espaço.

3. Por que um espaço de tuplas puro não tem operação `count()`? O que seria necessário adicionar ao modelo para suportá-la?

> Porque o modelo Linda puro é minimalista (apenas `OUT`, `IN` e `RD`) e trata o espaço como um multiconjunto não-ordenado, acessado exclusivamente por casamento de padrões de uma tupla por vez. Uma operação `count()` exigiria uma visão global instantânea do espaço, o que conflita com a concorrência de múltiplos produtores e consumidores alterando tuplas simultaneamente. Para suportá-la, seria necessário adicionar ao modelo contadores/índices mantidos pelo servidor ou primitivas de iteração com cursor.

Cole o trecho do `Monitor.java` que você completou:

```java
    private static int contarTarefas(JavaSpace space) throws Exception {
        TaskEntry template = new TaskEntry(null, null, null);
        Map<Integer, Boolean> idsVistos = new HashMap<>();
        int repeticoesSeguidas = 0;
        final int maxRepeticoes = 10;

        // Usa read() (leitura não-destrutiva) com timeout = 0 (JavaSpace.NO_WAIT)
        // para não remover tarefas do espaço nem bloquear quando não houver mais entradas.
        while (repeticoesSeguidas < maxRepeticoes) {
            TaskEntry encontrada = (TaskEntry) space.read(template, null, JavaSpace.NO_WAIT);
            if (encontrada == null) {
                break;
            }
            if (encontrada.id != null && !idsVistos.containsKey(encontrada.id)) {
                idsVistos.put(encontrada.id, Boolean.TRUE);
                repeticoesSeguidas = 0;
            } else {
                repeticoesSeguidas++;
            }
        }

        return idsVistos.size();
    }
```

---

