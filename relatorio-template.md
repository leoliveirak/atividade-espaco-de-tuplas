# Relatório — Atividade Prática: Espaço de Tuplas com Apache River

**Disciplina:** Computação Distribuída
**Nome:** Leonardo de Oliveira Klitzke
**Data:** 28/10/2026

---

## Nível 0 — Rodar e observar

### Observação inicial

1. Qual serviço aparece nos logs primeiro? Por que ele precisa existir antes dos outros?

> _Resposta:_ `reggie-1`, por que ele atua como um "explorador", sem ele ativo, o servidor do `javaspaces` não consegue registrar a instância na rede e os clientes não tem commo não conseguem se conectar

2. O produtor menciona o nome ou o endereço do consumidor em algum momento? O consumidor menciona o produtor?

> _Resposta:_ Não, no espaço de tuplas, os processos não precisam conhecer a identidade, o endereço ip, ele cooperam por meio de dados compartilhados

3. O consumidor começa a processar tarefas antes que o produtor termine de depositar todas? O que isso diz sobre como os dois se coordenam?

> _Resposta:_ Sim, isso diz que os dois de coordenam de maneira assíncrona, caso o consumidor não encontre elementos, ele fica em estado de espera

---

### Experimento de desacoplamento temporal

4. O consumidor encontrou as tarefas mesmo sendo iniciado depois que o produtor já havia encerrado? O que isso demonstra?

> _Resposta:_ Sim, isso mostra o desacoplamento temporal, os dados são armazenados em um repositório que independe da existência de outros processos, assim, então é possivel consimir tarefas de um protudor que já encerrou.

5. Em comunicação direta via socket, seria possível esse comportamento? Por quê não?

> _Resposta:_ Não, pois necessita de acoplamento temporal. Como a troca de mensagens é direta, é necessário que ambos os processos estejam em execução.

---

## Nível 1 — Inspecionar

### 1.1 As três operações

Preencha a tabela com base no que você observou nos logs:

| Operação River | Equivalente Linda | O que ela faz? | Bloqueia quando não encontra correspondência? | Altera o estado do espaço? |
|---------------|-------------------|----------------|----------------------------------------------|---------------------------|
| `write(entry)` | `OUT` | escreve uma nova tupla | Não | Sim |
| `take(template)` | `IN` | busca uma tupla com o `template` e remove do espaco | Sim | Sim |
| `read(template)` | `RD` | Busca uma tupla com o `template` e retorna uma cópia | Sim | Não (pois apenas copia, mantendo a tupla lá) |

---

### 1.2 O papel do `reggie`

1. Quando o `reggie` caiu, os serviços que já estavam conectados ao espaço continuaram funcionando? Por quê?

> _Resposta:_ Sim. Porque eles já possuíam a referência direta

2. O que aconteceria com um produtor ou consumidor que tentasse iniciar enquanto o `reggie` estivesse fora do ar?

> _Resposta:_ Não conseguiria se conectar ao espaço

3. Qual sistema moderno cumpre papel equivalente ao `reggie` em uma arquitetura de microsserviços?

> _Resposta:_ De acordo com as minhas pesquisas, ferramentas de Service Discovery (ex.: Consul, Eureka ou Kubernetes Service Discovery).

---

### 1.3 Desacoplamento espacial

1. O produtor tem qualquer informação sobre quantos consumidores existem?

> _Resposta:_ Não, ele apenas insere tuplas no espaço.

2. O consumidor tem qualquer informação sobre quem produziu a tarefa que ele retirou?

> _Resposta:_ Não, ele apenas busca tuplas correspondentes ao seu template no espaço.

3. Como produtor e consumidor se coordenam se não se conhecem?

> _Resposta:_ Através do espaço de tuplas (que é compartilhado), e o match de templates.

---

### 1.4 Comportamento de bloqueio

1. O que o consumidor fez enquanto o espaço estava vazio?

> _Observado:_ Ficou bloqueado aguardando uma tarefa no espaço.

2. Quando o produtor depositou a primeira tarefa, o que aconteceu imediatamente?

> _Observado:_ O consumidor foi desbloqueado.

3. Esse comportamento tem nome no modelo Linda. Qual é e por que ele é útil em sistemas distribuídos reais?

> _Resposta:_ Leitura bloqueante, é útil pois elimina a espera ativa, economizando recursos, acordando o processo apenas quando necessário

---

### 1.5 Escalabilidade horizontal

1. Uma mesma tarefa foi processada por dois consumidores ao mesmo tempo?

> _Observado:_ Não. Cada tarefa foi retirada por apenas um consumidor.

2. O produtor precisou ser modificado para suportar dois consumidores?

> _Resposta:_ Não, pois o produtor escreve no espaço sem conhecer os consumidores.

3. Esse comportamento tem um nome em arquitetura de sistemas. Qual é?

> _Resposta:_ Competing Consumers Pattern

---

## Nível 2 — Modificar

### 2.1 Modificação A — Prioridade de tarefas

1. As tarefas de prioridade alta foram processadas antes das de prioridade baixa? Cole um trecho dos logs que evidencie isso:

Sim, o log abaixo mostra isso acontecendo

```
produtor-1    | [PRODUTOR] Espaço encontrado via lookup.
consumidor-1  | [CONSUMIDOR-1] Espaço encontrado. Aguardando tarefas...
produtor-1    | [PRODUTOR] write: TaskEntry{id=1, tipo="calcular", prioridade=2}
produtor-1    | [PRODUTOR] write: TaskEntry{id=2, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=2, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] write: TaskEntry{id=3, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 2
produtor-1    | [PRODUTOR] write: TaskEntry{id=4, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=4, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] write: TaskEntry{id=5, tipo="calcular", prioridade=1}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 4
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=5, tipo="calcular", prioridade=1}
produtor-1    | [PRODUTOR] Todas as tarefas depositadas. Encerrando.
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 5
produtor-1 exited with code 0
consumidor-1  | [CONSUMIDOR-1] Nenhuma tarefa de alta prioridade disponível. Buscando qualquer tarefa...
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=1, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 1
consumidor-1  | [CONSUMIDOR-1] Nenhuma tarefa de alta prioridade disponível. Buscando qualquer tarefa...
consumidor-1  | [CONSUMIDOR-1] take: TaskEntry{id=3, tipo="calcular", prioridade=2}
consumidor-1  | [CONSUMIDOR-1] Processamento concluído: tarefa 3
consumidor-1  | [CONSUMIDOR-1] Nenhuma tarefa de alta prioridade disponível. Buscando qualquer tarefa...
```

2. O produtor precisou ser modificado para que isso funcionasse?

> _Resposta:_ Não, ele apenas escreve a tupla informando a prioridade, a lógica de escolha é do consumidor

3. Como o consumidor consegue selecionar apenas tarefas de uma prioridade específica? Qual mecanismo do espaço de tuplas torna isso possível?

> _Resposta:_ Através do match de templates, o consumidor passa um `take()` num objeto template com o campo `prioridade` preenchido 

---

### 2.2 Modificação B — Serviço monitor

1. Qual operação você usou no monitor — `read()` ou `take()`? Por quê a outra seria problemática?

> _Resposta:_ Operação `read()` pois ela apenas lê do espaço (sem remover), já a take remove, e não é isso que queremos

2. Como você contou as tarefas pendentes usando apenas `read()`? Que limitação isso revela?

> _Resposta:_ Executanto um loop de read() e registrando os ids num set para evitar duplicações, e interrompe após um limite de tentativas sem um id novo. uma desvantagem é o custo O(N), outra é a imprecisão que pode acontecer devido a concorrencia.

3. Por que um espaço de tuplas puro não tem operação `count()`? O que seria necessário adicionar ao modelo para suportá-la?

> _Resposta:_ Por que o espaço de tuplas funciona com um pool de dados soltos. Para suportar o count, precisaria de um contador interno que atualiza a cada insersão / remoção (sofreria bloqueio para atualizar a informação)

Cole o trecho do `Monitor.java` que você completou:

```java
    private static int contarTarefas(JavaSpace space) throws Exception {
        TaskEntry template = new TaskEntry(null, null, null);
        int count = 0;

        // a operação read() com timeout=0 pode retornar a mesma entrada mais de uma vez,
        // então precisamos rastrear quais ids já foram vistos para contar corretamente.
        java.util.Set<Integer> idsVistos = new java.util.HashSet<>();
        int tentativas = 0; 

        TaskEntry encontrada = null;

        // loop de read() para contar entradas únicas
        while ((encontrada = (TaskEntry) space.read(template, null, 0)) != null && tentativas < 20) {
            if (!idsVistos.contains(encontrada.id) && encontrada.id != null) {
                idsVistos.add(encontrada.id);
                count++; // incrementa a contagem apenas se o id for único
                tentativas = 0; // resetar tentativas se encontrar uma nova entrada
            } else {
                tentativas++; // incrementa tentativas se não encontrar uma nova entrada
            }
            
        }
        return count;
    }
```

---

## Observações livres

_(Comportamentos inesperados, erros encontrados, dificuldades técnicas — descreva o que aconteceu e como você resolveu)_

>

---

## Dúvida para a próxima aula

_(Formule uma pergunta substantiva que surgiu durante a atividade)_

>
