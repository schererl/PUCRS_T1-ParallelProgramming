# -PUCRS-T1-ParallelProgramming

# Problema-Proposto

Relaxação do algoritmo *Sequential Halving*, um algoritmo de Inteligência Artificial para jogos que tem como objetivo fazer uma estimativa do espaço de ações de um jogo, dado um estado, como por exemplo uma configuração atual de um tabuleiro. A estimativa é composta por dois momentos, as simulações de monte-carlo e a diminuição iterativa do espaço-de-ações cortando de metade em metade. 

Normalmente é necessário que exista um modelo do domínio do problema para que seja possível gerar as configurações do tabuleiro, buscar por quais ações são aplicáveis ao estado do tabuleiro e uma função de transformação de um par estado-ação em um novo estado. Como o propósito deste trabalho é avaliar o paralelismo da aplicação, não nos preocuparemos em implementar tal modelo. Para explorarmos o funcionamento do algoritmo, a etapa de simulação de monte-carlo que é o local do código onde ocorre o processamento pesado, usaremos um método que define aleatóriamente um tempo de processamento entre dois intervalos. O propósito de realizar as simulações é estimar a recompensa esperada da ação a ser avaliada, dada a soma das recompensas obtidas a partir das simulações realizadas. Reproduziremos este processo de duas formas, primeiro é gerado um tempo de processamento aleatório entre um intervalo, neste período o programa entrará em *loop* executando instruções dispensáveis que servem apenas para gerar trabalho. No segundo momento é gerado um valor aleatório de recompensa no intervalo [0,1] para servir como estimativa da recompensa obtida da ação. A outra etapa do algoritmo, calcula o *budget* destinado para cada ação, ordena as mesmas de acordo com as recompensas obtidas na etapa de simulação e dispensa metade do espaço de ações a cada iteração do menor para o maior.

## Código

1- Sequential Halving

```
Budget (budget de simulações)
N (número de ações)
A (espaço-de-ações)
n_layer (tamanho do espaço-de-ações do layer)
Bn (budget virtual)
layer = N

enquanto(n_layer > 1):
  Bn = calcula_budget(Budget) 
  Para cada ação 'a' em A[0:Vn]:
    simula(a, Bn)
  ordena(A[0:n_layer])
  n_layer = ceil(n_layer/2)
```
2- Simulação Monte-Carlo (a, Bn)

```
budget_alocado = (Bn)
tempo_simulacao = rand(limite_inferior, limite_superior)
enquanto(tempo < tempo_simulacao):
  trabalho()
recompensa = random(0,1)
retorna recompensa
```
# Discussão Modelos de Paralelismo

Para o problema proposto, existem três etapas do processo que podem ser paralelizadas:

  * **Paralelismo de Raiz:** Cada thread executa o *sequential halving* de forma independente, divindo o budget definido igualmente e depois juntam-se os resultados. Aqui não existe distribuição de carga, cada core executará o algoritmo independente da diferença de trabalho que cada um enfrenta. Como o problema faz uso de simulações de monte-carlo, a quantidade de trabalho que cada thread executará deve variar bastante. A vantagem deste modelo é a baixa necessidade de comunicação entre as threads abertas, possivelmente um modelo Mestre-Escravo poderia ser bem aplicado.

  * **Paralelismo de Ação** Cada thread executar um certo número de ações de forma independente. Aqui, possivelmente, existirão problemas em situações aonde existe mais threads do que ações disponíveis, onde, possivelmente não haverá trabalho para algumas threads. Apesar de existir a independência das ações, existirá uma necessidade moderada de comunicação nos diferentes *layers* do *sequential halving*.

  * **Paralelismo de Simulação** Cada thread executa uma parte do budget destinado para cada ação de forma independente. Neste modelo existirá a melhor distribuição de carga possível. A cada ação observada no algoritmo, o budget destinado à aquela ação é divido entre as *n* threads. Quanto maior o grau de paralelismo da aplicação, maior o budget que pode ser usado porque mais threads poderão dividí-lo para executar em paralelo. A única situação que acreditamos ser a mais crítica é o alto grau de comunicação exigido: para cada ação do layer atual as *n* threds devem ser abertas e fechadas, este processo se repete para cada *layer* executado no programa.

## Paralelismo de Ação

O paralelismo de ação consiste em dividir as ações, destinando uma ou um conjunto delas para as threads disponíveis. Avaliaremos o ganho de desempenho da aplicação aumentando gradativamente o número de threads. 

A tabela abaixo mostra a execução do programa variando o número de threads no conjunto {1,2,4,6,8,16}. Os testes foram executados no LAD por uma máquina com 8/16 núcleos.

BUDGET|THREADS|TEMPO EXECUÇÃO|NÚMERO AÇÕES|
--- | --- | --- | --- |
40000|1|164.50 s||10|
40000|2|87.85 s||10|
40000|4|65.83 s|10|
40000|6|49.52 s|10|
40000|8|49.64 s|10|
40000|16|44.27 s|10|

Perceba que o ganho de desempenho ocorre numa escala bem abaixo do esperado. A partir dos 4 cores já temos um desempenho, aproximadamente de 63% de eficiência. O tempo de execução ideal estaria em torno de 41 segundos, ao invés de 65.83s. A partir das 6 threads a variação se torna muito pequena quando comparado ao que seria uma speed-up ideal.

A explicação que acreditamos que esteja ocorrendo é devido a granularidade do problema ser muito grande. O paralelismo de ação divide as tasks de acordo com o número de ações a cada iteração do algoritmo. Como estamos fazendo um experimento estático, a primeira iteração sempre começará com um espaço de 10 ações, seguido de 5, 3, 2 e sai do laço quando sobrar apenas uma única melhor ação. O problema nesta forma de distribuir trabalho é que existe um número muito restrito de ações para se aplicar as simulações em paralelo e ainda a cada *layer* esse número cai pela metade. Pegando o caso das 16 threads, já 6 recursos ficam sem trabalho, afinal o mínimo que o problema pode ser quebrado é em 10 ações, ao passo que, a medida que o espaço de ações diminui, o aproveitamento do paralelismo reduz ainda mais.

A execução em 2 threads ser a que mais se aproxima do speed-up ideal antes de ter uma queda abrupta não é por acaso: perceba que no caso do nosso espaço de 10 ações, as duas threads conseguem executar todo o workload em paralelio. No momento que as duas últimas ações terminam suas simulações, a execução termina. A execução com 4 threads ainda consegue aproveitar todos os recurso até chegar no espaço com 3 ações, a partir dai, terá menos trabalho do que threads disponíveis.

**alteração na carga**

* grão 2 (ação):
  - 4 threads 98.68 s
  - 8 threads 87.96 s
  - 16 threads 87.99 s
  
Realizando outro teste a respeito da glanularidade do problema, aumentamos o workload das tasks, agrupando mais de uma ação para cada. Os resultados como é possível ver, foram piores. O curioso nesta etapa foi perceber que aumentando a granularidade, o desempenho com 4,8 e 16 threads foi inferior ao de 2 threads com uma ação por task. Não ficou claro o porquê deste fenomeno, intuitivamente o que pensávamos seria que de fato o programa escalaria ainda menos com o número de threads, mas que pelo menos tivesse um desempenho superiro à versão paralela com 2 threads. Talvez, aumentar o grão fez com que aumentassem o número de *misses* nas memórias cache, o que decai o desempenho da aplicação. 

## Pralelismo de Simulação

Os testes feitos com o paralelismo de Simulação serão apresentados abaixo:

BUDGET|THREADS|TEMPO EXECUÇÃO|NÚMERO AÇÕES|
--- | --- | --- | --- |
40000|1|164.50 s|10
40000|2|82.45 s|10
40000|3|54.89 s|10
40000|4|41.27 s|10
40000|6|27.53 s|10
40000|8|20.66 s|10
40000|16|10.38 s|10

Comparado aos resultados feitos com o paralelismo de ação, fica evidente o ganho de desempenho da aplicação que mantém o speed-up muito próximo do ideal, inclusive com o hyperthreading no teste com 16 threads, o que pode se explicar por um ganho de desempenho graças a diminuição do grão, fazendo um melhor aproveitamento das memórias cache. 

Conforme mencionamos na explicação deste modelo, a distribuição de carga entre as threads divide o budget destinado para uma ação entre todas as threads. Como estamos explorando o paralelismo, quanto maior o número de threads, maior o budget que poderíamos sem alterar o tempo de execução. Neste experimento nós não aumentamos o budget geral distribuído, portanto o tempo de execução que diminuiu.


**alterações na carga:**

Algumas variações no tamanho das tasks foram testadas, agrupando um número de simulações por tarefa, aumentando a carga de trabalho. A intuição por traz é restringir a comunicação do modelo que aparenta ser a mais elevada entre os 3 métodos de paralelização.

* grão 2 (folha):
  - 4 threads 41.28s
  - 8 threads 20.71s  
  - 16 threads 10.43s
  
* grão 10 (folha):
  - 4 threads 41.59s
  - 8 threads 21.04s
  - 16 threads 10.80s

* grão 100 (folha):
  - 4 threads 44.68s
  - 8 threads 26.83s
  - 16 threads 15.61s
  
Percebe-se que mesmo aumentando o grão não existe um impacto significativo na execução do programa, inclusive tem uma pequena perda de desempenho. Para testarmos um caso mais extremo, também fizemos uma versão com grão 100, ou seja, 100 simulações por task. Aqui existe um nítida perda no desempenho para 4, 8 e 16 threads. O motivo é semelhante ao problema apresentado no paralelismo de ação, onde existem situações ao qual o budget passado não é divisível pelo tamanho de task definida e podendo levar a situações onde algumas threads fiquem subutilizadas.

  
# Conclusão

  
