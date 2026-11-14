# Prática 06: replicação, partição e quórum

**Disciplina:** Tópicos Especiais IV (Computação Distribuída)
**Curso:** Bacharelado em Sistemas de Informação, UNIPAM

A Prática 05 terminou com a eleição de líder: quando o líder cai, o algoritmo do valentão escolhe
o processo de maior número entre os ativos, e todos passam a reconhecê-lo. Esta prática dá a esse
líder um papel concreto. Ele passa a ser o **primário** de um sistema de reservas replicado, e os
demais processos guardam cópias das reservas.

O programa desta prática, `replicas.py`, é a continuação do `reservas.py`. O relógio de Lamport, os
registros, a eleição do valentão e os comandos `verificar`, `eleicao`, `lider` e `contagem` são os
mesmos. A exclusão mútua de Ricart e Agrawala sai do programa, porque o primário passa a ordenar
todas as escritas, como o coordenador do algoritmo centralizado da Unidade 4. O que entra é a
replicação: cada reserva vai do primário para as cópias, e a versão de cada reserva é o par formado
pelo valor do relógio de Lamport e pelo número do processo.

A prática tem quatro partes. Na primeira, o primário cai e uma reserva é promovida sem perder dados.
Na segunda, a rede se divide sem que nenhum processo perceba, cada lado elege o seu primário e a
mesma vaga é reservada para dois clientes. Na terceira, a mesma partição é repetida com quórum de
escrita e de leitura, e o lado menor deixa de aceitar reservas. Na quarta, a partição é feita pelo
cabo, sem nenhuma ajuda do programa.

A prática não pede entrega. No fim de cada parte, um quadro destaca os **valores para analisar**,
que entram no relatório final da sequência de práticas, descrito na seção 7.

---

## Sumário

| Seção | Assunto |
|---|---|
| Quadro de referência | máquinas, processos, portas e comandos, para consulta rápida |
| 1 | Fundamentação |
| 2 | Preparação do ambiente |
| 3 | Primeira parte: cópia primária e promoção da reserva |
| 4 | Segunda parte: partição silenciosa e cérebro dividido |
| 5 | Terceira parte: quórum de escrita e de leitura |
| 6 | Quarta parte: a partição pelo cabo |
| 7 | O relatório final |
| 8 | Síntese |
| 9 | Questões para estudo |
| 10 | Referência de comandos |
| 11 | Diagnóstico de falhas |
| 12 | Apêndice com o conteúdo integral do programa |

---

## Objetivos de aprendizagem

Ao final desta prática o aluno deve ser capaz de:

- Descrever as três etapas de uma escrita na cópia primária e contar as mensagens de cada uma
- Explicar por que a promoção de uma reserva preserva as reservas já efetivadas
- Reconhecer uma partição de rede pelo efeito que ela produz, e explicar por que ela se confunde
  com a queda de um processo
- Descrever o cérebro dividido e o modo como a sincronização pela maior versão perde uma escrita
  confirmada
- Aplicar a condição W + R > N e prever qual lado de uma partição continua aceitando escritas
- Relacionar os resultados das partes 2 e 3 às duas escolhas do teorema CAP

---

## Quadro de referência

**As máquinas, as mesmas das práticas anteriores**

| Papel | Nome | Endereço na rede exclusiva de hospedeiro |
|---|---|---|
| processos P2 e P4 | `banco` | 192.168.56.101 |
| processos P1, P3 e P5 | `aplicacao` | 192.168.56.102 |

**Os processos e as portas**

| Processo | Máquina | Porta UDP | Papel no início |
|---|---|---|---|
| P1 | `aplicacao` | 9501 | reserva |
| P2 | `banco` | 9502 | reserva |
| P3 | `aplicacao` | 9503 | reserva |
| P4 | `banco` | 9504 | reserva |
| P5 | `aplicacao` | 9505 | primário |

Os cinco processos participam de todas as partes.

**O diretório de trabalho, nas duas máquinas**

```
/home/SEU_USUARIO/pratica06
```

**Os comandos aceitos por um processo em execução**

| Comando | Efeito |
|---|---|
| `reservar 12 Ana` | no primário, reserva a vaga 12 para Ana e replica a reserva |
| `ler 12` | mostra a vaga 12, pela cópia local ou com quórum de leitura |
| `reservas` | mostra todas as reservas que este processo tem |
| `particao P2 P4` | separa P2 e P4 dos demais, sem aviso a ninguém |
| `unir` | desfaz a partição |
| `sincronizar` | troca as reservas com todos e mantém a maior versão de cada vaga |
| `verificar`, `eleicao`, `lider` | os mesmos da Prática 05 |
| `contagem` e `relogio` | os mesmos da Prática 05 |
| `sair` | encerra o processo |

---

## 1. Fundamentação

### 1.1 Por que replicar

Um sistema de reservas guardado numa só máquina para de funcionar quando ela cai, e as reservas se
perdem se o disco falha. Manter cópias em várias máquinas resolve as duas coisas: as cópias que
restam continuam atendendo, e os dados sobrevivem à perda de uma delas. As leituras também podem se
dividir entre as cópias.

O preço é manter as cópias iguais. Se duas cópias aceitam, ao mesmo tempo, a vaga 12 para clientes
diferentes, cada uma fica com um dono para a vaga, e o sistema deixa de ter uma resposta certa. Os
algoritmos de replicação definem quem aceita a escrita e em que ordem as cópias a aplicam.

### 1.2 A cópia primária

Na cópia primária, um processo é o **primário** e só ele aceita escritas. Os demais são as
**reservas**. No programa, o primário é o líder da eleição do valentão, que no início é P5. Cada
escrita passa por três etapas:

1. o primário envia ESCRITA a todas as reservas, com a vaga, o cliente e a versão
2. cada reserva guarda a escrita como pendente e responde CONFIRMA
3. o primário conta as confirmações, efetiva a reserva e envia EFETIVA a todas, e só então as
   reservas aplicam a escrita

Com n processos, uma escrita custa n − 1 mensagens em cada etapa, ou 3(n − 1) no total. Com os
cinco processos desta prática, são 12.

A **versão** de uma reserva é o par (relógio de Lamport, número do processo) do envio da ESCRITA.
É a mesma ordem total da Prática 04, e ela serve para decidir qual cópia é a mais recente quando
duas cópias discordam.

Quando o primário cai, a eleição do valentão promove a reserva de maior número. Como cada reserva já
aplicou as escritas efetivadas, o novo primário continua de onde o anterior parou.

### 1.3 A partição de rede

Uma **partição** divide os processos em grupos que não se comunicam. Ninguém é avisado. As
mensagens que atravessam o corte simplesmente não chegam, e cada lado percebe apenas que o outro
deixou de responder. Pelo tempo limite, uma partição e a queda de um processo produzem a mesma
observação, como na Unidade 4.

O comando `particao P2 P4` simula esse corte dentro do programa. Ele é digitado nos cinco terminais
e, a partir dele, cada processo descarta, em silêncio, o que enviaria ao outro lado e o que
receberia de lá. As mensagens continuam sendo contadas como enviadas, porque quem envia não tem como
saber que elas se perderam. A quarta parte repete o corte sem o comando, desconectando o cabo.

### 1.4 O quórum

Com quórum, uma escrita só vale se o primário reunir **W** confirmações, contando a própria, e uma
leitura só vale se reunir **R** respostas, contando a própria. Se W + R > N, todo grupo de leitura
tem pelo menos um processo que recebeu a última escrita efetivada, e a leitura escolhe a cópia de
maior versão. Se W > N/2, duas escritas também têm sempre um processo em comum, e dois lados de uma
partição não podem efetivar escritas ao mesmo tempo.

O programa começa com W = 1 e R = 1, o que significa nenhum quórum. As opções `--w 3 --r 3` dão o
quórum da terceira parte, com N = 5.

### 1.5 O tempo limite

As confirmações e as respostas de leitura são esperadas por até 2 segundos. Quando todos respondem
antes, o processo segue na hora. Quando alguém não responde, porque caiu ou porque está do outro
lado de uma partição, o processo espera os 2 segundos e decide com as respostas que recebeu. A
mesma espera vale para o PING de `verificar` e para a eleição, como na Prática 05.

---

## 2. Preparação do ambiente

As duas máquinas virtuais das práticas anteriores, ligadas. Nada é instalado.

São necessários **cinco** terminais na máquina hospedeira: dois com acesso à máquina `banco` e três
com acesso à máquina `aplicacao`.

```bash
ssh SEU_USUARIO@192.168.56.101       # terminais de P2 e de P4
```

```bash
ssh SEU_USUARIO@192.168.56.102       # terminais de P1, P3 e P5
```

Em cada sessão, criar o diretório de trabalho e entrar nele:

```bash
mkdir -p ~/pratica06
cd ~/pratica06
```

Copiar o arquivo `replicas.py` para a máquina `banco` pelo mesmo procedimento da Prática 05, abrindo
o editor com `nano -I replicas.py` e colando o conteúdo. Em seguida, na máquina `banco`, copiar o
arquivo para a máquina `aplicacao`:

```bash
scp replicas.py SEU_USUARIO@192.168.56.102:~/pratica06/
```

O conteúdo integral do programa está no apêndice, na seção 12.

**Atenção.** Os comandos são digitados na ordem indicada, um terminal de cada vez. Depois de cada
comando, esperar as linhas de recebimento aparecerem nos outros terminais antes de seguir. Os valores
de versão dos quadros supõem que cada parte começa com os cinco processos recém-iniciados.

---

## 3. Primeira parte: cópia primária e promoção da reserva

### 3.1 Iniciar os processos

```bash
python3 replicas.py processo P1      # terminal P1, maquina aplicacao
python3 replicas.py processo P2      # terminal P2, maquina banco
python3 replicas.py processo P3      # terminal P3, maquina aplicacao
python3 replicas.py processo P4      # terminal P4, maquina banco
python3 replicas.py processo P5      # terminal P5, maquina aplicacao
```

Cada um em um terminal. Cada processo mostra o grupo, o primário inicial e os quóruns:

```
Processo P1, porta UDP 9501, grupo P1 P2 P3 P4 P5, primario inicial P5.
Quorum de escrita W = 1, quorum de leitura R = 1, N = 5.
```

### 3.2 Uma escrita replicada

No terminal P5:

```
reservar 12 Ana
```

P5 envia a ESCRITA às quatro reservas, recebe as quatro confirmações e efetiva a reserva:

```
L=1 P5 envia ESCRITA vaga 12 Ana versao (1, P5) para P1, P2, P3, P4
L=4 P5 recebe CONFIRMA de P1 (mensagem com 3)
L=5 P5 recebe CONFIRMA de P2 (mensagem com 3)
L=6 P5 recebe CONFIRMA de P3 (mensagem com 3)
L=7 P5 recebe CONFIRMA de P4 (mensagem com 3)
  confirmacoes: 5 de 5, contando o primario. Quorum de escrita: 1
L=8 P5 efetiva a reserva da vaga 12 para Ana
L=9 P5 envia EFETIVA versao (1, P5) para P1, P2, P3, P4
```

A ordem das confirmações pode variar. Em cada reserva, a escrita fica pendente até o EFETIVA:

```
L=2 P2 recebe ESCRITA de P5 (mensagem com 1)
L=3 P2 confirma versao (1, P5) para P5
L=10 P2 recebe EFETIVA de P5 (mensagem com 9)
  vaga 12 reservada para Ana, versao (1, P5)
```

Digitar `reservas` e `contagem` nos cinco terminais. Todos têm a vaga 12 para Ana, com a versão
(1, P5). P5 enviou ESCRITA 4 e EFETIVA 4, e cada reserva enviou CONFIRMA 1. São 12 mensagens, o
valor de 3(n − 1) com n = 5.

### 3.3 A queda do primário

| Passo | Terminal | Comando |
|---|---|---|
| 1 | P5 | `sair`, que simula a queda do primário |
| 2 | P2 | `verificar` |
| 3 | todos | esperar até que nenhuma linha nova apareça |

A eleição é a mesma da quarta parte da Prática 05, e P4 assume:

```
  nenhum processo maior respondeu em 2 s
L=18 P4 anuncia COORDENADOR para P1, P2, P3
  P4 e o novo lider e o primario
```

### 3.4 O novo primário continua

No terminal P4:

| Passo | Comando | O que aparece |
|---|---|---|
| 1 | `reservas` | `vaga 12: Ana, versao (1, P5)` |
| 2 | `reservar 12 Bia` | `vaga 12 ja reservada para Ana, pedido recusado` |
| 3 | `reservar 13 Bia` | a ESCRITA vai a P1, P2, P3 e P5 |

No passo 3, P5 não responde, porque caiu. P4 espera os 2 segundos e efetiva com as respostas que
recebeu:

```
L=19 P4 envia ESCRITA vaga 13 Bia versao (19, P4) para P1, P2, P3, P5
L=22 P4 recebe CONFIRMA de P2 (mensagem com 21)
L=23 P4 recebe CONFIRMA de P1 (mensagem com 21)
L=24 P4 recebe CONFIRMA de P3 (mensagem com 21)
  confirmacoes: 4 de 5, contando o primario. Quorum de escrita: 1
L=25 P4 efetiva a reserva da vaga 13 para Bia
L=26 P4 envia EFETIVA versao (19, P4) para P1, P2, P3, P5
```

O valor do relógio na versão de Bia pode variar, porque P3 e P4 conduzem eleições ao mesmo tempo. O
número do processo, P4, não varia.

### 3.5 A volta do antigo primário

No terminal P5, iniciar o processo de novo e consultar as reservas:

```bash
python3 replicas.py processo P5
```

```
reservas
```

```
Nenhuma reserva.
```

P5 volta sem nenhum dado e, pelo valor inicial do programa, acredita ser o primário. Antes de
qualquer reserva, ele precisa receber o estado das cópias. No terminal P5:

```
sincronizar
```

```
L=1 P5 pede o ESTADO para P1, P2, P3, P4
L=30 P5 recebe ESTADO de P1 (mensagem com 29)
  vaga 12 reservada para Ana, versao (1, P5)
  vaga 13 reservada para Bia, versao (19, P4)
  estados recebidos: 4
L=34 P5 envia o ESTADO combinado para P1, P2, P3, P4
```

Só então, no terminal P5:

```
eleicao
```

P5 não tem processo maior, assume e anuncia COORDENADOR aos outros quatro. Digitar `lider` e
`reservas` nos cinco terminais: todos reconhecem P5 e têm as duas reservas.

**Valores para analisar na primeira parte**

| O que conferir | Resultado |
|---|---|
| versão da reserva de Ana | (1, P5) |
| mensagens de uma escrita sem falhas | 12: ESCRITA 4, CONFIRMA 4, EFETIVA 4 |
| valor de 3(n − 1) com n = 5 | 12 |
| primário depois da queda de P5 | P4 |
| reserva de Ana no novo primário | presente, versão (1, P5) |
| `reservar 12 Bia` em P4 | recusado, vaga já reservada |
| confirmações da vaga 13 | 4 de 5, depois de esperar 2 segundos |
| reservas de P5 ao voltar, antes de `sincronizar` | nenhuma |
| mensagens de `sincronizar` enviadas por P5 | SINCRONIA 4 e ESTADO 4 |
| primário depois da eleição de P5 | P5, com as vagas 12 e 13 |

Encerrar os cinco processos com `sair`.

---

## 4. Segunda parte: partição silenciosa e cérebro dividido

### 4.1 O corte

Iniciar os cinco processos de novo, com os mesmos comandos da seção 3.1. Em seguida, digitar nos
cinco terminais:

```
particao P2 P4
```

Os processos da máquina `banco` mostram:

```
Particao: este lado tem P2 P4. Nenhuma mensagem chega a P1 P3 P5, nem vem de la.
```

E os da máquina `aplicacao`:

```
Particao: este lado tem P1 P3 P5. Nenhuma mensagem chega a P2 P4, nem vem de la.
```

O corte coincide com a divisão entre as duas máquinas. É o que aconteceria se a rede entre elas
deixasse de funcionar.

### 4.2 Os dois lados continuam

| Passo | Terminal | Comando | O que acontece |
|---|---|---|---|
| 1 | P5 | `reservar 12 Ana` | P5 espera 2 segundos e efetiva com 3 de 5 |
| 2 | P2 | `ler 12` | `vaga 12: livre (copia local)` |
| 3 | P2 | `verificar` | P5 não responde, e P4 vence a eleição do lado menor |
| 4 | P4 | `reservar 12 Bia` | P4 espera 2 segundos e efetiva com 2 de 5 |

No passo 1, só P1 e P3 confirmam. Com W = 1, as três confirmações bastam:

```
L=1 P5 envia ESCRITA vaga 12 Ana versao (1, P5) para P1, P2, P3, P4
L=4 P5 recebe CONFIRMA de P1 (mensagem com 3)
L=5 P5 recebe CONFIRMA de P3 (mensagem com 3)
  confirmacoes: 3 de 5, contando o primario. Quorum de escrita: 1
L=6 P5 efetiva a reserva da vaga 12 para Ana
```

No passo 2, P2 responde que a vaga está livre. A reserva de Ana já foi confirmada ao cliente, mas a
cópia de P2 não a recebeu.

No passo 3, a eleição acontece só entre P2 e P4, e P4 anuncia COORDENADOR a P1, P2 e P3, mas só P2
recebe. No passo 4:

```
L=7 P4 envia ESCRITA vaga 12 Bia versao (7, P4) para P1, P2, P3, P5
L=10 P4 recebe CONFIRMA de P2 (mensagem com 9)
  confirmacoes: 2 de 5, contando o primario. Quorum de escrita: 1
L=11 P4 efetiva a reserva da vaga 12 para Bia
```

Digitar `lider` e `reservas` nos cinco terminais:

| Processo | Líder | Vaga 12 |
|---|---|---|
| P1, P3, P5 | P5 | Ana, versão (1, P5) |
| P2, P4 | P4 | Bia, versão (7, P4) |

Há dois primários, e a mesma vaga foi confirmada a dois clientes. É o **cérebro dividido** visto na
Unidade 4, agora com dados.

### 4.3 A rede volta

Digitar `unir` nos cinco terminais. Nada muda nas cópias, porque nenhum processo sabe que houve
partição. Em seguida, no terminal P5:

```
sincronizar
```

Cada vaga fica com a maior versão. (7, P4) é maior que (1, P5), e nas cópias de P1, P3 e P5 aparece:

```
  vaga 12: Ana (1, P5) substituida por Bia (7, P4)
```

Por fim, no terminal P4, `eleicao`. P5 responde, toma a liderança e anuncia COORDENADOR a todos.
Digitar `lider` e `reservas` nos cinco terminais: todos reconhecem P5, e todos têm a vaga 12 para
Bia.

As cópias voltaram a concordar, mas a reserva de Ana, que tinha sido confirmada ao cliente, se
perdeu. A regra da maior versão produz convergência, não correção.

**Valores para analisar na segunda parte**

| O que conferir | Resultado |
|---|---|
| confirmações da reserva de Ana | 3 de 5 |
| `ler 12` em P2 depois da reserva de Ana | livre |
| primário de cada lado durante a partição | P5 e P4 |
| confirmações da reserva de Bia | 2 de 5 |
| versões em disputa | (1, P5) para Ana e (7, P4) para Bia |
| vaga 12 depois de `sincronizar` | Bia, em todas as cópias |
| reserva confirmada que se perdeu | a de Ana |
| primário depois da eleição de P4 | P5 |

Encerrar os cinco processos com `sair`.

---

## 5. Terceira parte: quórum de escrita e de leitura

### 5.1 Os processos com quórum

Iniciar os cinco processos com as opções de quórum:

```bash
python3 replicas.py processo P1 --w 3 --r 3      # terminal P1, maquina aplicacao
python3 replicas.py processo P2 --w 3 --r 3      # terminal P2, maquina banco
python3 replicas.py processo P3 --w 3 --r 3      # terminal P3, maquina aplicacao
python3 replicas.py processo P4 --w 3 --r 3      # terminal P4, maquina banco
python3 replicas.py processo P5 --w 3 --r 3      # terminal P5, maquina aplicacao
```

```
Quorum de escrita W = 3, quorum de leitura R = 3, N = 5.
```

Com N = 5, W = 3 e R = 3, vale 3 + 3 > 5. Digitar `particao P2 P4` nos cinco terminais, como na
seção 4.1.

### 5.2 A mesma sequência

| Passo | Terminal | Comando | O que acontece |
|---|---|---|---|
| 1 | P5 | `reservar 12 Ana` | 3 de 5, efetiva |
| 2 | P2 | `ler 12` | 2 de 5, leitura recusada |
| 3 | P2 | `verificar` | P4 vence a eleição do lado menor |
| 4 | P4 | `reservar 12 Bia` | 2 de 5, escrita recusada |

No passo 1, P5 reúne exatamente o quórum:

```
  confirmacoes: 3 de 5, contando o primario. Quorum de escrita: 3
L=6 P5 efetiva a reserva da vaga 12 para Ana
```

No passo 2, a leitura de P2 só obtém a resposta de P4:

```
L=1 P2 envia LEITURA vaga 12 para P1, P3, P4, P5
L=4 P2 recebe VALOR de P4 (mensagem com 3)
  respostas: 2 de 5, contando este processo. Quorum de leitura: 3
  leitura recusada: sem quorum
```

No passo 4, P4 é primário, mas só reúne a própria confirmação e a de P2:

```
  confirmacoes: 2 de 5, contando o primario. Quorum de escrita: 3
L=15 P4 escrita recusada: vaga 12 Bia, sem quorum
```

O lado menor continua elegendo líder, mas não consegue reservar nem ler. Digitar `reservas` nos
cinco terminais: P1, P3 e P5 têm a vaga 12 para Ana, e P2 e P4 não têm nenhuma reserva.

### 5.3 A rede volta

Digitar `unir` nos cinco terminais. Em seguida, no terminal P2:

```
ler 12
```

```
  respostas: 5 de 5, contando este processo. Quorum de leitura: 3
  vaga 12: Ana, versao (1, P5) (maior versao entre 5 copias)
```

A cópia local de P2 continua vazia, porque o EFETIVA de Ana se perdeu na partição. Mesmo assim, a
leitura com quórum encontra a reserva, porque pelo menos uma das respostas veio de um processo que a
recebeu. Para terminar, no terminal P4, `eleicao`, e P5 volta a ser o primário de todos.

**Valores para analisar na terceira parte**

| O que conferir | Resultado |
|---|---|
| W, R e N | 3, 3 e 5, com 3 + 3 > 5 |
| confirmações da reserva de Ana | 3 de 5, efetivada |
| leitura em P2 durante a partição | 2 de 5, recusada |
| confirmações da reserva de Bia em P4 | 2 de 5, recusada |
| reservas de P2 e P4 durante a partição | nenhuma |
| leitura em P2 depois de `unir` | Ana, versão (1, P5), com 5 respostas |
| reserva confirmada que se perdeu | nenhuma |

Encerrar os cinco processos com `sair`.

---

## 6. Quarta parte: a partição pelo cabo

### 6.1 O que muda

Nas partes 2 e 3, quem descartava as mensagens era o próprio programa. Nesta parte, o corte é
físico: o cabo de rede é desconectado, e nenhuma linha do programa sabe disso. As sequências das
seções 4.2 e 5.2 são repetidas sem os comandos `particao` e `unir`. No lugar de `particao P2 P4`,
desconecta-se o cabo. No lugar de `unir`, reconecta-se o cabo.

### 6.2 Nas máquinas virtuais

O cabo é o da placa de rede exclusiva de hospedeiro da máquina `banco`. Desconectá-lo derruba também
as sessões SSH dessa máquina, e por isso os terminais de P2 e de P4 são abertos na própria janela da
máquina virtual `banco`, e não na hospedeira.

| Passo | Onde | Ação |
|---|---|---|
| 1 | janela da máquina `banco` | entrar com o usuário e abrir dois terminais, com `cd ~/pratica06` em cada um |
| 2 | esses dois terminais | iniciar P2 e P4 |
| 3 | hospedeira, por SSH | iniciar P1, P3 e P5 na máquina `aplicacao` |
| 4 | janela da máquina `banco` | no menu **Dispositivos**, opção **Rede**, desmarcar a conexão do cabo do adaptador |
| 5 | terminais | a sequência da seção 4.2 |
| 6 | janela da máquina `banco` | marcar de novo a conexão do cabo |
| 7 | terminais | a seção 4.3, a partir de `sincronizar` |

Em seguida, repetir com `--w 3 --r 3` nos cinco processos e a sequência da seção 5.2, reconectando
o cabo antes da seção 5.3.

### 6.3 No laboratório técnico

No laboratório técnico, os dois lados são dois computadores ligados pelo cabo de rede montado na
Unidade 2. Num deles executam P2 e P4, e no outro P1, P3 e P5. Os endereços são os do plano de
endereçamento da bancada, informados na linha de comando de todos os processos:

```bash
python3 replicas.py processo P2 --banco 10.0.0.1 --aplicacao 10.0.0.2
```

Os endereços acima são um exemplo. A partição é feita retirando o cabo de um dos computadores, e a
rede volta quando ele é recolocado.

### 6.4 O que comparar

Os resultados esperados são os mesmos das partes 2 e 3, com uma diferença no registro: as linhas
de envio continuam aparecendo, e nenhuma mensagem de erro indica que elas não chegaram.

**Valores para analisar na quarta parte**

| O que conferir | Resultado |
|---|---|
| reserva de Ana, sem quórum, com o cabo desconectado | 3 de 5, efetivada |
| reserva de Bia, sem quórum, em P4 | 2 de 5, efetivada |
| vaga 12 depois de reconectar e sincronizar, sem quórum | Bia, a maior versão |
| reserva de Bia, com quórum | 2 de 5, recusada |
| leitura em P2 depois de reconectar, com quórum | Ana |
| aviso de erro no envio durante o corte | nenhum |

---

## 7. O relatório final

Esta prática é a terceira etapa do mesmo sistema. Para o relatório final, ela é executada de novo,
e os quatro quadros de valores para analisar indicam o que deve ser obtido e comparado. Nada é
entregue agora.

As comparações que o relatório vai pedir partem destes quadros: o número de mensagens de uma escrita
contra a fórmula 3(n − 1), a reserva que se perde sem quórum contra a que se preserva com quórum, e
a partição simulada pelo programa contra a partição pelo cabo.

---

## 8. Síntese

| Pergunta | Sem quórum, W = 1 e R = 1 | Com quórum, W = 3 e R = 3 |
|---|---|---|
| Quem aceita escritas na partição | os dois lados, cada um com seu primário | só o lado com três processos |
| O que o lado menor responde | a cópia local, que pode estar desatualizada | recusa a escrita e a leitura |
| O que acontece quando a rede volta | a maior versão vence, e uma reserva confirmada se perde | a leitura com quórum encontra a reserva |
| Escolha no teorema CAP | disponibilidade | consistência |
| Ligação com as práticas anteriores | a versão é o par da ordem total da Prática 04 | o primário é o líder do valentão da Prática 05 |

---

## 9. Questões para estudo

1. Na primeira parte, a escrita de P5 custou 12 mensagens. Quantas custaria com 3 processos e com
   10 processos?
2. Por que as reservas guardam a escrita como pendente e só a aplicam depois do EFETIVA? O que
   aconteceria, na terceira parte, se P2 aplicasse a escrita de Bia ao recebê-la?
3. Na seção 3.5, P5 voltou acreditando ser o primário. O que aconteceria se alguém digitasse
   `reservar 12 Carlos` em P5 antes do `sincronizar`?
4. Na segunda parte, a leitura de P2 devolveu a vaga livre depois que Ana recebeu a confirmação. Por
   que essa leitura não é linearizável?
5. Na segunda parte, qual seria o resultado do `sincronizar` se a reserva de Ana tivesse sido feita
   depois da de Bia, com um valor maior de relógio?
6. Com N = 5, quais pares de W e R satisfazem W + R > N com W > N/2? Qual deles mantém o lado de
   três processos aceitando escritas e leituras?
7. Com W = 5 e R = 1, o que aconteceria na terceira parte com a reserva de Ana?
8. Na terceira parte, o lado menor recusou as operações. Que propriedade do teorema CAP foi
   sacrificada, e para quais clientes?
9. Explicar por que, nas partes 2 e 4, nenhum processo mostrou mensagem de erro, embora metade das
   mensagens tenha se perdido.

---

## 10. Referência de comandos

**Iniciar um processo**

```bash
python3 replicas.py processo P1                      # sem quorum, partes 1, 2 e 4
python3 replicas.py processo P1 --w 3 --r 3          # com quorum, partes 3 e 4
python3 replicas.py processo P1 --banco IP --aplicacao IP   # laboratorio tecnico
```

**Dentro de um processo**

```
reservar 12 Ana        no primario, reserva e replica
ler 12                 consulta a vaga, local ou com quorum
reservas               reservas desta copia
particao P2 P4         separa P2 e P4 dos demais
unir                   desfaz a particao
sincronizar            troca as reservas e mantem a maior versao
verificar              envia PING ao lider
eleicao                inicia uma eleicao
lider                  mostra o lider conhecido
contagem               mensagens enviadas, por tipo
relogio                valor do relogio de Lamport
sair                   encerra o processo
```

**Ordem total dos registros, como na Prática 04**

```bash
python3 replicas.py ordenar P1.log P2.log P3.log P4.log P5.log
```

**Plano alternativo, com todos os processos numa só máquina**

Se uma das máquinas virtuais não estiver disponível, os cinco processos podem executar na mesma
máquina, em terminais diferentes, com a opção `--mesma-maquina` em todos eles. As partes 1, 2 e 3
dão os mesmos resultados. A quarta parte exige as duas máquinas.

```bash
python3 replicas.py processo P1 --mesma-maquina
```

---

## 11. Diagnóstico de falhas

| Sintoma | Causa provável | Verificação |
|---|---|---|
| `OSError: [Errno 98] Address already in use` | o mesmo processo já está em execução em outro terminal | encerrar a outra instância, conferindo com `ss -lunp` |
| `Este processo nao e o primario` | a reserva foi digitada fora do primário | digitar `lider` e usar o terminal indicado |
| na segunda parte, a reserva de Ana teve 5 confirmações | `particao P2 P4` não foi digitado em algum terminal | digitar o comando nos cinco terminais |
| as versões diferem das dos quadros | os processos não foram reiniciados no começo da parte | reiniciar os cinco e repetir a sequência |
| na terceira parte, a escrita de Ana foi recusada | algum processo do lado maior foi iniciado sem `--w 3 --r 3` | reiniciar os cinco com as mesmas opções |
| na quarta parte, o terminal de P2 parou de responder | P2 foi iniciado por SSH, e a sessão caiu com o cabo | abrir os terminais de P2 e P4 na janela da máquina `banco` |
| Erro de indentação ao executar | espaços perdidos na colagem | abrir de novo com `nano -I` e colar o conteúdo inteiro |

---

## 12. Apêndice com o conteúdo integral do programa

### 12.1 `replicas.py`, nas máquinas `banco` e `aplicacao`

```python
import json
import re
import socket
import sys
import threading
import time

# Os processos da pratica: P2 e P4 na maquina banco, P1, P3 e P5 na maquina aplicacao.
PROCESSOS = {
    "P1": ("192.168.56.102", 9501),
    "P2": ("192.168.56.101", 9502),
    "P3": ("192.168.56.102", 9503),
    "P4": ("192.168.56.101", 9504),
    "P5": ("192.168.56.102", 9505),
}
TEMPO_LIMITE = 2.0


def numero(nome):
    return int(nome[1:])


def versao_txt(v):
    return "(%d, P%d)" % (v[0], v[1])


class Processo:
    def __init__(self, nome, grupo, w, r):
        self.nome = nome
        self.grupo = ["P%d" % i for i in range(1, grupo + 1)]
        self.outros = [p for p in self.grupo if p != nome]
        self.w = w
        self.r = r
        self.relogio = 0
        self.trava = threading.RLock()
        self.enviadas = {}
        # Particao simulada: processos com que este processo deixou de se comunicar, sem saber
        self.cortados = set()
        # Replicacao: cada vaga guarda o cliente e a versao (relogio de Lamport, numero do processo)
        self.reservas = {}
        self.pendentes = {}
        self.confirmacoes = set()
        self.respostas = {}
        self.esperando = None
        # Eleicao pelo algoritmo do valentao, como na Pratica 05
        self.lider = self.grupo[-1]
        self.em_eleicao = False
        self.recebeu_resposta = False
        self.pong = False
        self.registro = open(nome + ".log", "w")
        self.soquete = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.soquete.bind(("0.0.0.0", PROCESSOS[nome][1]))

    def anotar(self, texto):
        linha = "L=%d %s %s" % (self.relogio, self.nome, texto)
        print(linha)
        self.registro.write(linha + "\n")
        self.registro.flush()

    # O relogio de Lamport da Pratica 04, com o ajuste no recebimento.
    def avancar(self):
        self.relogio += 1

    def local(self, texto="evento local"):
        with self.trava:
            self.avancar()
            self.anotar(texto)

    def enviar(self, destinos, tipo, texto, extra=None):
        with self.trava:
            self.avancar()
            mensagem = {"de": self.nome, "relogio": self.relogio, "tipo": tipo}
            mensagem.update(extra or {})
            dados = json.dumps(mensagem).encode("utf-8")
            for destino in destinos:
                self.enviadas[tipo] = self.enviadas.get(tipo, 0) + 1
                # Na particao, a mensagem sai normalmente e se perde no caminho.
                if destino in self.cortados:
                    continue
                try:
                    self.soquete.sendto(dados, PROCESSOS[destino])
                except OSError:
                    pass
            self.anotar("%s para %s" % (texto, ", ".join(destinos)))

    def receber(self, m):
        with self.trava:
            self.avancar()
            if self.relogio <= m["relogio"]:
                self.relogio = m["relogio"] + 1
            print()
            self.anotar("recebe %s de %s (mensagem com %d)" % (m["tipo"], m["de"], m["relogio"]))
            tratar = getattr(self, "ao_receber_" + m["tipo"].lower())
            tratar(m)
            print("%s> " % self.nome, end="", flush=True)

    def escutar(self):
        while True:
            try:
                dados, _ = self.soquete.recvfrom(65536)
            except OSError:
                continue
            m = json.loads(dados.decode("utf-8"))
            with self.trava:
                perdida = m["de"] in self.cortados
            if not perdida:
                self.receber(m)

    def ao_receber_msg(self, m):
        pass

    # Particao silenciosa: o processo continua enviando, mas nada atravessa o corte.
    def particao(self, lado):
        with self.trava:
            if self.nome in lado:
                self.cortados = set(p for p in self.grupo if p not in lado)
            else:
                self.cortados = set(p for p in self.grupo if p in lado)
            lado_a = sorted(set(self.grupo) - self.cortados, key=numero)
            print("Particao: este lado tem %s. Nenhuma mensagem chega a %s, nem vem de la."
                  % (" ".join(lado_a), " ".join(sorted(self.cortados, key=numero))))

    def unir(self):
        with self.trava:
            self.cortados = set()
            print("A rede voltou a ligar todos os processos.")

    # Espera as respostas de todos os outros processos, ou o tempo limite.
    def aguardar(self, conjunto):
        fim = time.time() + TEMPO_LIMITE
        while time.time() < fim:
            with self.trava:
                if len(conjunto) == len(self.outros):
                    return
            time.sleep(0.02)

    # Copia primaria: so o primario aceita a reserva e a repassa as replicas.
    def reservar(self, vaga, cliente):
        with self.trava:
            if self.lider != self.nome:
                print("Este processo nao e o primario. O primario e %s." % self.lider)
                return
            if vaga in self.reservas:
                print("  vaga %s ja reservada para %s, pedido recusado" % (vaga, self.reservas[vaga][0]))
                return
            # A versao e o par do proprio envio da ESCRITA: relogio de Lamport e numero do processo.
            self.confirmacoes = set()
            versao = (self.relogio + 1, numero(self.nome))
            self.esperando = versao
            self.enviar(self.outros, "ESCRITA", "envia ESCRITA vaga %s %s versao %s"
                        % (vaga, cliente, versao_txt(versao)),
                        {"vaga": vaga, "cliente": cliente, "versao": list(versao)})
        self.aguardar(self.confirmacoes)
        with self.trava:
            self.esperando = None
            total = len(self.confirmacoes) + 1
            print("  confirmacoes: %d de %d, contando o primario. Quorum de escrita: %d"
                  % (total, len(self.grupo), self.w))
            if total < self.w:
                self.local("escrita recusada: vaga %s %s, sem quorum" % (vaga, cliente))
                print("%s> " % self.nome, end="", flush=True)
                return
            self.reservas[vaga] = (cliente, versao)
            self.local("efetiva a reserva da vaga %s para %s" % (vaga, cliente))
            self.enviar(self.outros, "EFETIVA", "envia EFETIVA versao %s" % versao_txt(versao),
                        {"versao": list(versao)})
            print("%s> " % self.nome, end="", flush=True)

    def ao_receber_escrita(self, m):
        versao = tuple(m["versao"])
        self.pendentes[versao] = (m["vaga"], m["cliente"])
        self.enviar([m["de"]], "CONFIRMA", "confirma versao %s" % versao_txt(versao),
                    {"versao": list(versao)})

    def ao_receber_confirma(self, m):
        if self.esperando == tuple(m["versao"]):
            self.confirmacoes.add(m["de"])

    def ao_receber_efetiva(self, m):
        versao = tuple(m["versao"])
        if versao in self.pendentes:
            vaga, cliente = self.pendentes.pop(versao)
            self.aplicar(vaga, cliente, versao)

    def aplicar(self, vaga, cliente, versao):
        atual = self.reservas.get(vaga)
        if atual is None or versao > atual[1]:
            if atual is not None:
                print("  vaga %s: %s %s substituida por %s %s"
                      % (vaga, atual[0], versao_txt(atual[1]), cliente, versao_txt(versao)))
            self.reservas[vaga] = (cliente, versao)
            print("  vaga %s reservada para %s, versao %s" % (vaga, cliente, versao_txt(versao)))

    # Leitura com quorum R: com R = 1, cada processo responde com o que tem.
    def ler(self, vaga):
        with self.trava:
            proprio = self.reservas.get(vaga)
            if self.r <= 1:
                self.mostrar_vaga(vaga, proprio, "copia local")
                return
            self.respostas = {self.nome: proprio}
            self.enviar(self.outros, "LEITURA", "envia LEITURA vaga %s" % vaga, {"vaga": vaga})
        self.aguardar_leitura()
        with self.trava:
            total = len(self.respostas)
            print("  respostas: %d de %d, contando este processo. Quorum de leitura: %d"
                  % (total, len(self.grupo), self.r))
            if total < self.r:
                print("  leitura recusada: sem quorum")
                return
            valores = [v for v in self.respostas.values() if v is not None]
            maior = max(valores, key=lambda v: v[1]) if valores else None
            self.mostrar_vaga(vaga, maior, "maior versao entre %d copias" % total)

    def aguardar_leitura(self):
        fim = time.time() + TEMPO_LIMITE
        while time.time() < fim:
            with self.trava:
                if len(self.respostas) == len(self.grupo):
                    return
            time.sleep(0.02)

    def mostrar_vaga(self, vaga, valor, origem):
        if valor is None:
            print("  vaga %s: livre (%s)" % (vaga, origem))
        else:
            print("  vaga %s: %s, versao %s (%s)" % (vaga, valor[0], versao_txt(valor[1]), origem))

    def ao_receber_leitura(self, m):
        valor = self.reservas.get(m["vaga"])
        extra = {"vaga": m["vaga"], "valor": None if valor is None else [valor[0], list(valor[1])]}
        self.enviar([m["de"]], "VALOR", "responde VALOR vaga %s" % m["vaga"], extra)

    def ao_receber_valor(self, m):
        v = m["valor"]
        self.respostas[m["de"]] = None if v is None else (v[0], tuple(v[1]))

    # Sincronizacao depois da particao: vence a maior versao de cada vaga.
    def estado(self):
        return {vaga: [c, list(v)] for vaga, (c, v) in self.reservas.items()}

    def sincronizar(self):
        with self.trava:
            self.respostas = {self.nome: None}
            self.enviar(self.outros, "SINCRONIA", "pede o ESTADO", {"reservas": self.estado()})
        self.aguardar_leitura()
        with self.trava:
            print("  estados recebidos: %d" % (len(self.respostas) - 1))
            self.enviar(self.outros, "ESTADO", "envia o ESTADO combinado", {"reservas": self.estado()})
            print("%s> " % self.nome, end="", flush=True)

    def combinar(self, reservas):
        for vaga, (cliente, versao) in reservas.items():
            self.aplicar(vaga, cliente, tuple(versao))

    def ao_receber_sincronia(self, m):
        self.combinar(m["reservas"])
        self.enviar([m["de"]], "ESTADO", "responde ESTADO", {"reservas": self.estado()})

    def ao_receber_estado(self, m):
        self.combinar(m["reservas"])
        if m["de"] not in self.respostas:
            self.respostas[m["de"]] = None

    # Eleicao pelo algoritmo do valentao, igual a da Pratica 05.
    def verificar(self):
        with self.trava:
            lider = self.lider
            if lider == self.nome:
                print("Este processo e o lider.")
                return
            self.pong = False
            self.enviar([lider], "PING", "envia PING ao lider")
        time.sleep(TEMPO_LIMITE)
        with self.trava:
            respondeu = self.pong
        if respondeu:
            print("  o lider %s respondeu" % lider)
        else:
            print("  o lider %s nao respondeu em %g s" % (lider, TEMPO_LIMITE))
            self.iniciar_eleicao()

    def ao_receber_ping(self, m):
        self.enviar([m["de"]], "PONG", "responde PONG")

    def ao_receber_pong(self, m):
        self.pong = True

    def iniciar_eleicao(self):
        with self.trava:
            if self.em_eleicao:
                return
            self.em_eleicao = True
            self.recebeu_resposta = False
            maiores = [p for p in self.grupo if numero(p) > numero(self.nome)]
            if maiores:
                self.enviar(maiores, "ELEICAO", "envia ELEICAO")
        if maiores:
            threading.Thread(target=self.aguardar_respostas, daemon=True).start()
        else:
            self.assumir()

    def aguardar_respostas(self):
        time.sleep(TEMPO_LIMITE)
        with self.trava:
            if not self.recebeu_resposta:
                print("\n  nenhum processo maior respondeu em %g s" % TEMPO_LIMITE)
        if not self.recebeu_resposta:
            self.assumir()
            return
        print("\n  um processo maior respondeu, %s sai da disputa e espera o COORDENADOR" % self.nome)
        time.sleep(3 * TEMPO_LIMITE)
        with self.trava:
            sem_anuncio = self.em_eleicao
            self.em_eleicao = False
        if sem_anuncio:
            print("\n  nenhum COORDENADOR chegou, nova eleicao")
            self.iniciar_eleicao()

    def assumir(self):
        with self.trava:
            self.lider = self.nome
            self.em_eleicao = False
            menores = [p for p in self.grupo if numero(p) < numero(self.nome)]
            if menores:
                self.enviar(menores, "COORDENADOR", "anuncia COORDENADOR")
            print("  %s e o novo lider e o primario" % self.nome)
            print("%s> " % self.nome, end="", flush=True)

    def ao_receber_eleicao(self, m):
        self.enviar([m["de"]], "RESPOSTA", "responde OK a ELEICAO")
        threading.Thread(target=self.iniciar_eleicao, daemon=True).start()

    def ao_receber_resposta(self, m):
        self.recebeu_resposta = True

    def ao_receber_coordenador(self, m):
        self.lider = m["de"]
        self.em_eleicao = False
        print("  novo lider e primario: %s" % m["de"])


AJUDA = ("Comandos: reservar VAGA CLIENTE | ler VAGA | reservas | particao P.. | unir | sincronizar |\n"
         "          verificar | eleicao | lider | contagem | relogio | sair")


def cmd_processo(args):
    nome = args[0].upper()
    if nome not in PROCESSOS:
        print("Processo desconhecido. Use P1 a P5.")
        sys.exit(1)
    grupo = int(args[args.index("--grupo") + 1]) if "--grupo" in args else 5
    w = int(args[args.index("--w") + 1]) if "--w" in args else 1
    r = int(args[args.index("--r") + 1]) if "--r" in args else 1
    if numero(nome) > grupo:
        print("O processo %s nao faz parte de um grupo de %d processos." % (nome, grupo))
        sys.exit(1)
    p = Processo(nome, grupo, w, r)
    print("Processo %s, porta UDP %d, grupo %s, primario inicial %s."
          % (nome, PROCESSOS[nome][1], " ".join(p.grupo), p.lider))
    print("Quorum de escrita W = %d, quorum de leitura R = %d, N = %d." % (w, r, grupo))
    print(AJUDA)
    print("L=0 %s inicio" % nome)
    threading.Thread(target=p.escutar, daemon=True).start()
    while True:
        try:
            partes = input("%s> " % nome).split()
        except EOFError:
            break
        if not partes:
            continue
        if partes[0] == "reservar" and len(partes) == 3:
            p.reservar(partes[1], partes[2])
        elif partes[0] == "ler" and len(partes) == 2:
            p.ler(partes[1])
        elif partes[0] == "reservas":
            with p.trava:
                if not p.reservas:
                    print("Nenhuma reserva.")
                for vaga, (cliente, versao) in sorted(p.reservas.items()):
                    print("  vaga %s: %s, versao %s" % (vaga, cliente, versao_txt(versao)))
        elif partes[0] == "particao" and len(partes) >= 2:
            lado = set(x.upper() for x in partes[1:])
            if not lado <= set(p.grupo):
                print("Use nomes do grupo: %s" % " ".join(p.grupo))
            else:
                p.particao(lado)
        elif partes[0] == "unir":
            p.unir()
        elif partes[0] == "sincronizar":
            p.sincronizar()
        elif partes[0] == "verificar":
            p.verificar()
        elif partes[0] == "eleicao":
            p.iniciar_eleicao()
        elif partes[0] == "lider":
            print("Lider e primario: %s" % p.lider)
        elif partes[0] == "contagem":
            with p.trava:
                total = sum(p.enviadas.values())
                print("Mensagens enviadas por %s: %s, total %d" % (p.nome, ", ".join(
                    "%s %d" % (t, n) for t, n in sorted(p.enviadas.items())) or "nenhuma", total))
        elif partes[0] == "relogio":
            print("L=%d" % p.relogio)
        elif partes[0] == "sair":
            break
        else:
            print(AJUDA)


# Ordem total: pelo valor do relogio e, no empate, pelo numero do processo.
def cmd_ordenar(arquivos):
    eventos = []
    for arquivo in arquivos:
        with open(arquivo) as f:
            for linha in f:
                m = re.match(r"L=(\d+) (P\d) (.*)", linha.strip())
                if m:
                    eventos.append((int(m.group(1)), int(m.group(2)[1:]), m.group(2), m.group(3)))
    eventos.sort()
    for n, (valor, _, processo, texto) in enumerate(eventos, 1):
        print("%3d  L=%-4d %s %s" % (n, valor, processo, texto))


if __name__ == "__main__":
    if "--mesma-maquina" in sys.argv:
        PROCESSOS = {nome: ("127.0.0.1", porta) for nome, (_, porta) in PROCESSOS.items()}
    # No laboratorio tecnico, cada lado roda num computador, com o endereco dado na linha de comando.
    for opcao, nomes in (("--banco", ("P2", "P4")), ("--aplicacao", ("P1", "P3", "P5"))):
        if opcao in sys.argv:
            ip = sys.argv[sys.argv.index(opcao) + 1]
            for n in nomes:
                PROCESSOS[n] = (ip, PROCESSOS[n][1])
    if len(sys.argv) >= 3 and sys.argv[1] == "processo":
        cmd_processo(sys.argv[2:])
    elif len(sys.argv) >= 3 and sys.argv[1] == "ordenar":
        cmd_ordenar(sys.argv[2:])
    else:
        print("Uso: python3 replicas.py processo P1..P5 [--grupo N] [--w W] [--r R] [--mesma-maquina]")
        print("                                       [--banco IP] [--aplicacao IP]")
        print("     python3 replicas.py ordenar P1.log P2.log P3.log P4.log P5.log")
        sys.exit(1)
```

---

## Referência bibliográfica

BREWER, Eric. CAP twelve years later: how the "rules" have changed. **Computer**, v. 45, n. 2,
p. 23-29, 2012.

COULOURIS, George et al. **Sistemas distribuídos: conceitos e projetos**. 5. ed. Porto Alegre:
Bookman, 2013. Capítulo 18, replicação.

GIFFORD, David K. Weighted voting for replicated data. In: ACM SYMPOSIUM ON OPERATING SYSTEMS
PRINCIPLES, 7., 1979, Pacific Grove. **Proceedings** [...]. New York: ACM, 1979. p. 150-162.

KLEPPMANN, Martin. **Designing Data-Intensive Applications**. Sebastopol: O'Reilly, 2017.
Capítulos 5 e 9, replicação, consistência e consenso.

VAN STEEN, Maarten; TANENBAUM, Andrew S. **Distributed Systems**. 4. ed. 2023. Capítulo 7,
consistência e replicação.
