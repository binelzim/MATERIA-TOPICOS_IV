# Prática 05: exclusão mútua e eleição de líder

**Disciplina:** Tópicos Especiais IV (Computação Distribuída)
**Curso:** Bacharelado em Sistemas de Informação, UNIPAM

A Prática 04 terminou com a ordem total: cada evento recebe o par formado pelo valor do relógio de
Lamport e pelo número do processo, e todos os processos chegam à mesma sequência sem nenhum relógio
comum. Esta prática usa esse par para resolver dois problemas de coordenação vistos nos slides da
Unidade 4.

O primeiro é a exclusão mútua. Três processos disputam a reserva da mesma vaga, e só um pode
reservá-la de cada vez. O algoritmo de Ricart e Agrawala decide quem entra primeiro comparando os
pares dos pedidos, exatamente como a ordem total da prática anterior. O segundo é a eleição de
líder. Cinco processos dependem de um líder, o líder cai, e o algoritmo do valentão escolhe outro.

O programa desta prática, `reservas.py`, é a continuação do `lamport.py`. O relógio, as regras de
envio e recebimento, os registros e o comando `ordenar` são os mesmos. O que muda é que as
mensagens passam a ter tipos, como pedido, resposta e eleição, e cada tipo provoca uma reação no
processo que a recebe.

A prática não pede entrega. No fim de cada parte, um quadro destaca os **valores para analisar**,
que entram no relatório final da sequência de práticas, descrito na seção 7.

---

## Sumário

| Seção | Assunto |
|---|---|
| Quadro de referência | máquinas, processos, portas e comandos, para consulta rápida |
| 1 | Fundamentação |
| 2 | Preparação do ambiente |
| 3 | Primeira parte: uma reserva sem disputa |
| 4 | Segunda parte: um pedido durante a seção crítica |
| 5 | Terceira parte: dois pedidos empatados |
| 6 | Quarta parte: a queda do líder e a eleição |
| 7 | O relatório final |
| 8 | Síntese |
| 9 | Questões para estudo |
| 10 | Referência de comandos |
| 11 | Diagnóstico de falhas |
| 12 | Apêndice com o conteúdo integral do programa |

---

## Objetivos de aprendizagem

Ao final desta prática o aluno deve ser capaz de:

- Explicar por que a exclusão mútua entre máquinas diferentes só pode ser obtida por troca de
  mensagens
- Aplicar a regra de resposta do algoritmo de Ricart e Agrawala a partir do estado do processo e da
  comparação entre pedidos
- Contar as mensagens necessárias para uma entrada na seção crítica e relacionar o resultado à
  fórmula 2(n − 1)
- Explicar como o par (relógio de Lamport, número do processo) resolve dois pedidos simultâneos
- Descrever a sequência de mensagens do algoritmo do valentão a partir da queda do líder, e contar
  as mensagens de cada tipo
- Identificar o papel do tempo limite na detecção da queda e a hipótese que ele exige

---

## Quadro de referência

**As máquinas, as mesmas das práticas anteriores**

| Papel | Nome | Endereço na rede exclusiva de hospedeiro |
|---|---|---|
| processos P2 e P4 | `banco` | 192.168.56.101 |
| processos P1, P3 e P5 | `aplicacao` | 192.168.56.102 |

**Os processos e as portas**

| Processo | Máquina | Porta UDP | Partes em que participa |
|---|---|---|---|
| P1 | `aplicacao` | 9501 | todas |
| P2 | `banco` | 9502 | todas |
| P3 | `aplicacao` | 9503 | todas |
| P4 | `banco` | 9504 | quarta |
| P5 | `aplicacao` | 9505 | quarta |

**O diretório de trabalho, nas duas máquinas**

```
/home/SEU_USUARIO/pratica05
```

**Os comandos aceitos por um processo em execução**

| Comando | Efeito |
|---|---|
| `reservar 12` | pede a seção crítica para reservar a vaga 12 |
| `verificar` | envia PING ao líder e espera a resposta por 2 segundos |
| `eleicao` | inicia uma eleição |
| `lider` | mostra qual processo este considera o líder |
| `contagem` | mostra quantas mensagens de cada tipo este processo enviou |
| `relogio` | mostra o valor atual do relógio |
| `local` e `envia` | os mesmos da Prática 04 |
| `sair` | encerra o processo |

---

## 1. Fundamentação

### 1.1 A seção crítica sem memória compartilhada

Num único computador, dois processos que disputam um recurso são coordenados pelo sistema
operacional, com um semáforo ou uma trava na memória que ambos enxergam. Entre máquinas diferentes,
não existe memória comum. A única forma de um processo saber que outro está usando o recurso é
receber uma mensagem dele, e por isso a exclusão mútua distribuída é um protocolo de mensagens.

O algoritmo precisa garantir três propriedades. A **segurança** exige que no máximo um processo
esteja na seção crítica. A **vivacidade** exige que todo pedido acabe atendido, sem impasse e sem
inanição. A **ordenação** exige que um pedido feito antes, pela relação aconteceu-antes, seja
atendido antes.

### 1.2 O algoritmo de Ricart e Agrawala

Cada processo está em um de três estados: **liberado**, quando não quer o recurso, **querendo**,
quando pediu e espera, e **dentro**, quando está na seção crítica.

Para entrar, o processo passa a querendo e envia um PEDIDO a todos os outros, com o valor do
relógio de Lamport. O pedido fica identificado pelo par (relógio, número do processo). O processo
entra quando recebe OK de todos os outros.

Quem recebe um PEDIDO decide entre responder e adiar:

- se está dentro da seção crítica, adia a resposta
- se está querendo e o próprio pedido é menor que o recebido, adia a resposta
- em qualquer outro caso, responde OK na hora

Ao sair da seção crítica, o processo envia OK a todos os pedidos que adiou. Com n processos, cada
entrada custa n − 1 pedidos e n − 1 respostas, ou 2(n − 1) mensagens.

O par é comparado primeiro pelo valor do relógio e, no empate, pelo número do processo. É a ordem
total da Prática 04. Como todos os processos comparam da mesma forma, nunca acontece de dois deles
se considerarem, ao mesmo tempo, com o pedido menor.

### 1.3 A eleição pelo algoritmo do valentão

Cada processo tem um número, e o líder é o de maior número entre os que estão ativos. Quem percebe
que o líder não responde inicia uma eleição: envia ELEIÇÃO a todos os processos de número maior.
Quem recebe ELEIÇÃO responde OK, o que tira o remetente da disputa, e inicia a própria eleição. Se
nenhum processo maior responde, quem iniciou vence e anuncia COORDENADOR a todos os de número
menor. Quando um processo de número maior volta a funcionar, ele inicia uma eleição e toma a
liderança, porque nenhum processo maior vai responder a ele.

### 1.4 O tempo limite

A queda do líder e a ausência de resposta numa eleição são percebidas da mesma forma: uma mensagem
é enviada, e a resposta não chega dentro de um tempo limite. O programa usa 2 segundos.

O tempo limite só distingue um processo caído de um processo lento quando existe um limite conhecido
para o atraso das mensagens, que é a hipótese do sistema síncrono vista na aula de modelos de
sistema e de falha. Na rede das
máquinas virtuais, as mensagens levam frações de milissegundo, e 2 segundos são uma margem folgada.
Numa rede com atrasos imprevisíveis, um líder lento seria tomado por caído, e dois processos
poderiam se considerar líderes.

---

## 2. Preparação do ambiente

As duas máquinas virtuais das práticas anteriores, ligadas. Nada é instalado.

Da primeira à terceira parte, participam três processos, e são necessários **três** terminais na
máquina hospedeira: um com acesso à máquina `banco` e dois com acesso à máquina `aplicacao`. A quarta
parte acrescenta P4 e P5, e com eles mais dois terminais, um em cada máquina.

```bash
ssh SEU_USUARIO@192.168.56.101       # terminais de P2 e de P4
```

```bash
ssh SEU_USUARIO@192.168.56.102       # terminais de P1, P3 e P5
```

Em cada sessão, criar o diretório de trabalho e entrar nele:

```bash
mkdir -p ~/pratica05
cd ~/pratica05
```

Copiar o arquivo `reservas.py` para a máquina `banco` pelo mesmo procedimento da Prática 04, abrindo
o editor com `nano -I reservas.py` e colando o conteúdo. Em seguida, na máquina `banco`, copiar o
arquivo para a máquina `aplicacao`:

```bash
scp reservas.py SEU_USUARIO@192.168.56.102:~/pratica05/
```

O conteúdo integral do programa está no apêndice, na seção 12.

**Atenção.** Os comandos são digitados na ordem indicada, um terminal de cada vez. Depois de cada
comando, esperar as linhas de recebimento aparecerem nos outros terminais antes de seguir.

---

## 3. Primeira parte: uma reserva sem disputa

### 3.1 Iniciar os processos

```bash
python3 reservas.py processo P1      # terminal P1, maquina aplicacao
```

```bash
python3 reservas.py processo P2      # terminal P2, maquina banco
```

```bash
python3 reservas.py processo P3      # terminal P3, maquina aplicacao
```

Cada processo mostra o grupo de que faz parte e o líder inicial, que é o de maior número:

```
Processo P1, porta UDP 9501, grupo P1 P2 P3, lider inicial P3.
```

### 3.2 O pedido

No terminal P1:

```
reservar 12
```

P1 envia o pedido aos outros dois, recebe as duas respostas e entra na seção crítica. Depois de 10
segundos, sai:

```
L=1 P1 pede a vaga 12 para P2, P3
  pedido (1, P1)
L=4 P1 recebe OK de P2 (mensagem com 3)
L=5 P1 recebe OK de P3 (mensagem com 3)
L=6 P1 entra na secao critica: reserva a vaga 12
  sai da secao critica em 10 s
L=7 P1 sai da secao critica
```

A ordem das duas respostas pode se inverter, conforme a que chega primeiro, e os valores continuam
os mesmos. Nos terminais P2 e P3, cada processo recebe o pedido e responde OK na hora, porque está
liberado:

```
L=2 P2 recebe PEDIDO de P1 (mensagem com 1)
L=3 P2 responde OK para P1
```

### 3.3 A contagem

Digitar `contagem` nos três terminais:

```
Mensagens enviadas por P1: PEDIDO 2, total 2
Mensagens enviadas por P2: OK 1, total 1
Mensagens enviadas por P3: OK 1, total 1
```

Foram quatro mensagens para uma entrada: dois pedidos e duas respostas. Com n = 3, a fórmula
2(n − 1) dá 4.

**Valores para analisar na primeira parte**

| O que conferir | Resultado |
|---|---|
| pedido de P1 | (1, P1) |
| respostas recebidas por P1 | OK de P2 e OK de P3, ambas com 3 |
| valor do relógio na entrada | 6 |
| mensagens para uma entrada | 4: PEDIDO 2, OK 2 |
| valor de 2(n − 1) com n = 3 | 4 |

**Não encerrar os processos.** A segunda parte continua com eles.

---

## 4. Segunda parte: um pedido durante a seção crítica

### 4.1 A sequência

| Passo | Terminal | Comando ou ação |
|---|---|---|
| 1 | P1 | `reservar 12` |
| 2 | P1 | esperar a linha `entra na secao critica` |
| 3 | P3 | `reservar 12`, antes que P1 saia, dentro dos 10 segundos |
| 4 | todos | esperar P1 sair e P3 entrar e sair |

### 4.2 O que acontece

No terminal P1, o pedido de P3 chega enquanto P1 está dentro, e a resposta é adiada:

```
L=13 P1 entra na secao critica: reserva a vaga 12
  sai da secao critica em 10 s
L=14 P1 recebe PEDIDO de P3 (mensagem com 11)
  adia a resposta a P3: esta na secao critica
L=15 P1 sai da secao critica
L=16 P1 responde OK adiado para P3
```

No terminal P3, a resposta de P2 chega logo, e a de P1 só depois que P1 sai:

```
L=11 P3 pede a vaga 12 para P1, P2
  pedido (11, P3)
L=14 P3 recebe OK de P2 (mensagem com 13)
L=17 P3 recebe OK de P1 (mensagem com 16)
L=18 P3 entra na secao critica: reserva a vaga 12
```

É a propriedade de segurança em funcionamento. P3 não tem como saber que P1 está usando a vaga,
porque não há memória comum. O que o impede de entrar é a falta de uma resposta. A espera de P3
termina com a mensagem de P1, e não com a passagem do tempo.

**Valores para analisar na segunda parte**

| O que conferir | Resultado |
|---|---|
| pedido de P1 | (8, P1) |
| pedido de P3 | (11, P3) |
| reação de P1 ao pedido de P3 | adia, porque está na seção crítica |
| reação de P2 ao pedido de P3 | responde OK na hora |
| quando P3 recebe o OK de P1 | só depois de P1 sair |
| ordem de entrada | P1, depois P3 |

Encerrar os três processos com `sair`.

---

## 5. Terceira parte: dois pedidos empatados

### 5.1 Pedidos simultâneos

Nas partes anteriores, um pedido sempre chegou depois do outro, e o relógio de quem pediu depois já
tinha avançado. Para que dois pedidos sejam de fato simultâneos, cada um tem de sair antes de o
outro chegar. A opção `--atraso 5` faz cada mensagem levar 5 segundos para sair, o que dá tempo de
digitar o segundo pedido antes de o primeiro chegar.

Iniciar os três processos com a opção:

```bash
python3 reservas.py processo P1 --atraso 5      # terminal P1
```

```bash
python3 reservas.py processo P2 --atraso 5      # terminal P2
```

```bash
python3 reservas.py processo P3 --atraso 5      # terminal P3
```

### 5.2 A sequência

| Passo | Terminal | Comando |
|---|---|---|
| 1 | P1 | `reservar 12` |
| 2 | P2 | `reservar 12`, logo em seguida, dentro dos 5 segundos |

Os dois processos começam com o relógio em zero, e os dois pedidos saem com valor 1:

```
L=1 P1 pede a vaga 12 para P2, P3
  pedido (1, P1)
```

```
L=1 P2 pede a vaga 12 para P1, P3
  pedido (1, P2)
```

### 5.3 O desempate

Quando o pedido de P2 chega a P1, P1 está querendo e o próprio pedido é menor:

```
L=2 P1 recebe PEDIDO de P2 (mensagem com 1)
  adia a resposta a P2: o proprio pedido (1, P1) e menor que (1, P2)
```

Quando o pedido de P1 chega a P2, P2 também está querendo, mas o próprio pedido é maior, e P2
responde OK:

```
L=2 P2 recebe PEDIDO de P1 (mensagem com 1)
L=3 P2 responde OK para P1
```

P1 recebe OK de P2 e de P3 e entra. Ao sair, envia o OK adiado, e P2 entra em seguida. Os dois
processos tomaram decisões opostas a partir da mesma comparação, e por isso nunca entram juntos. É
a situação dos slides, com os valores 41 trocados por 1.

**Valores para analisar na terceira parte**

| O que conferir | Resultado |
|---|---|
| pedido de P1 | (1, P1) |
| pedido de P2 | (1, P2) |
| critério que decide | empate no relógio, desfeito pelo número do processo |
| reação de P1 ao pedido de P2 | adia |
| reação de P2 ao pedido de P1 | responde OK |
| ordem de entrada | P1, depois P2 |
| mensagens enviadas por P1, P2 e P3 | 3, 3 e 2, total de 8 para duas entradas |

Encerrar os três processos com `sair`.

---

## 6. Quarta parte: a queda do líder e a eleição

### 6.1 Cinco processos

Esta parte usa os cinco processos, com a opção `--grupo 5`, e a mesma situação dos slides: o líder
P5 cai, P2 percebe a queda, e P4 assume.

```bash
python3 reservas.py processo P1 --grupo 5      # terminal P1, maquina aplicacao
python3 reservas.py processo P2 --grupo 5      # terminal P2, maquina banco
python3 reservas.py processo P3 --grupo 5      # terminal P3, maquina aplicacao
python3 reservas.py processo P4 --grupo 5      # terminal P4, maquina banco
python3 reservas.py processo P5 --grupo 5      # terminal P5, maquina aplicacao
```

Cada um em um terminal. Digitar `lider` em todos: os cinco respondem P5.

### 6.2 A queda

| Passo | Terminal | Comando |
|---|---|---|
| 1 | P5 | `sair`, que simula a queda do líder |
| 2 | P2 | `verificar` |
| 3 | todos | esperar até que nenhuma linha nova apareça |

P2 envia PING a P5, não recebe resposta e inicia a eleição:

```
  o lider P5 nao respondeu em 2 s
L=2 P2 envia ELEICAO para P3, P4, P5
L=5 P2 recebe RESPOSTA de P3 (mensagem com 4)
L=6 P2 recebe RESPOSTA de P4 (mensagem com 4)
  um processo maior respondeu, P2 sai da disputa e espera o COORDENADOR
L=10 P2 recebe COORDENADOR de P4 (mensagem com 9)
  novo lider: P4
```

No terminal P4, a eleição termina sem resposta de P5:

```
L=8 P4 envia ELEICAO para P5
  nenhum processo maior respondeu em 2 s
L=9 P4 anuncia COORDENADOR para P1, P2, P3
  P4 e o novo lider
```

Nesta parte, os valores do relógio podem variar de uma execução para outra, porque P3 e P4 conduzem
eleições ao mesmo tempo, e a ordem em que as mensagens se cruzam muda. O que não varia é quem envia
o quê, e por isso a análise usa as contagens.

### 6.3 A contagem

Digitar `lider` e `contagem` nos quatro terminais ativos. Todos respondem P4 como líder, e as
contagens são:

| Processo | Mensagens enviadas |
|---|---|
| P1 | nenhuma |
| P2 | PING 1, ELEICAO 3 |
| P3 | ELEICAO 2, RESPOSTA 1 |
| P4 | ELEICAO 1, RESPOSTA 2, COORDENADOR 3 |

### 6.4 A volta do processo de maior número

No terminal P5, iniciar o processo de novo e pedir uma eleição:

```bash
python3 reservas.py processo P5 --grupo 5
```

```
eleicao
```

Não há processo maior que P5, e ele assume na hora:

```
L=1 P5 anuncia COORDENADOR para P1, P2, P3, P4
  P5 e o novo lider
```

Digitar `lider` nos cinco terminais: todos respondem P5. É o comportamento que dá nome ao
algoritmo. O processo de maior número toma a liderança assim que volta, mesmo que o líder atual
esteja funcionando.

**Valores para analisar na quarta parte**

| O que conferir | Resultado |
|---|---|
| tempo limite para perceber a queda | 2 segundos |
| quem percebe a queda | P2 |
| mensagens ELEICAO | 6: P2 envia 3, P3 envia 2, P4 envia 1 |
| mensagens RESPOSTA | 3: P3 envia 1, P4 envia 2 |
| mensagens COORDENADOR | 3, de P4 para P1, P2 e P3 |
| total da eleição, sem o PING | 12 |
| novo líder | P4 |
| mensagens na volta de P5 | 4 COORDENADOR, nenhuma ELEICAO |
| líder depois da volta | P5 |

Encerrar os cinco processos com `sair`.

---

## 7. O relatório final

Esta prática é a segunda etapa do mesmo sistema. Para o relatório final, ela é executada de novo,
e os quatro quadros de valores para analisar indicam o que deve ser obtido e comparado. Nada é
entregue agora.

As comparações que o relatório vai pedir partem destes quadros: o número de mensagens de uma
entrada na seção crítica contra a fórmula 2(n − 1), a decisão dos dois processos empatados contra a
ordem total da Prática 04, e o número de mensagens da eleição contra a volta do processo de maior
número.

---

## 8. Síntese

| Pergunta | Ricart e Agrawala | Valentão |
|---|---|---|
| Que problema resolve | quem usa o recurso agora | quem coordena depois da queda do líder |
| O que decide | o menor par (relógio, número do processo) | o maior número entre os ativos |
| Mensagens por execução, nesta prática | 4 por entrada, com 3 processos | 12 na eleição, com 5 processos |
| O que pode bloquear | a queda de qualquer processo, cuja resposta nunca chega | um tempo limite curto demais, que toma lento por caído |
| Ligação com a Prática 04 | o par é a ordem total | o relógio continua ajustado em toda mensagem |

---

## 9. Questões para estudo

1. Na segunda parte, P2 respondeu OK ao pedido de P3 na hora, e P1 adiou. O que, no estado de cada
   um, explica a diferença?
2. Na terceira parte, P2 também queria a vaga e mesmo assim respondeu OK a P1. Explicar por que essa
   resposta não fere a vontade de P2 e por que ela é necessária para a segurança.
3. Se P1 desfizesse os empates pelo menor número do processo e P2 pelo maior, o que aconteceria na
   terceira parte?
4. Suponha que P3 caia enquanto P1 espera as respostas, na primeira parte. O que acontece com P1, e
   por que esse é o ponto fraco do algoritmo apontado nos slides?
5. Calcular o número de mensagens de uma entrada na seção crítica com 5 processos e com 10
   processos. Comparar com o algoritmo centralizado.
6. Na quarta parte, P1 não enviou nenhuma mensagem. Explicar por quê, a partir da regra do
   algoritmo.
7. Repetir mentalmente a quarta parte com P1, e não P2, percebendo a queda. Quantas mensagens
   ELEICAO seriam enviadas?
8. O tempo limite do programa é de 2 segundos. Descrever o que aconteceria se P5 estivesse apenas
   lento, respondendo em 3 segundos.

---

## 10. Referência de comandos

**Iniciar um processo**

```bash
python3 reservas.py processo P1                  # grupo de 3, partes 1 e 2
python3 reservas.py processo P1 --atraso 5       # mensagens saem 5 s depois, parte 3
python3 reservas.py processo P1 --grupo 5        # grupo de 5, parte 4
python3 reservas.py processo P1 --duracao 20     # 20 s na secao critica
```

**Dentro de um processo**

```
reservar 12            pede a secao critica para a vaga 12
verificar              envia PING ao lider
eleicao                inicia uma eleicao
lider                  mostra o lider conhecido
contagem               mensagens enviadas, por tipo
relogio                valor do relogio de Lamport
sair                   encerra o processo
```

**Ordem total dos registros, como na Prática 04**

```bash
python3 reservas.py ordenar P1.log P2.log P3.log
```

**Plano alternativo, com todos os processos numa só máquina**

Se uma das máquinas virtuais não estiver disponível, os processos podem executar na mesma máquina,
em terminais diferentes, com a opção `--mesma-maquina` em todos eles. Os resultados são os mesmos.

```bash
python3 reservas.py processo P1 --mesma-maquina
```

---

## 11. Diagnóstico de falhas

| Sintoma | Causa provável | Verificação |
|---|---|---|
| `OSError: [Errno 98] Address already in use` | o mesmo processo já está em execução em outro terminal | encerrar a outra instância, conferindo com `ss -lunp` |
| `O processo P4 nao faz parte de um grupo de 3 processos` | P4 ou P5 iniciados sem `--grupo 5` | iniciar com a opção, como na seção 6.1 |
| o pedido fica sem entrar na seção crítica | um dos processos do grupo não foi iniciado, ou foi iniciado com outro grupo | conferir que todos foram iniciados com as mesmas opções |
| na terceira parte, não houve empate | o segundo pedido foi digitado depois que o primeiro chegou | reiniciar os três com `--atraso 5` e digitar o segundo pedido logo após o primeiro |
| na quarta parte, os líderes divergem | um processo foi iniciado sem `--grupo 5` | reiniciar todos com a opção |
| `Ja existe um pedido em andamento` | o pedido anterior ainda não terminou | esperar a linha `sai da secao critica` |
| Erro de indentação ao executar | espaços perdidos na colagem | abrir de novo com `nano -I` e colar o conteúdo inteiro |

---

## 12. Apêndice com o conteúdo integral do programa

### 12.1 `reservas.py`, nas máquinas `banco` e `aplicacao`

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


class Processo:
    def __init__(self, nome, grupo, duracao, atraso):
        self.nome = nome
        self.grupo = ["P%d" % i for i in range(1, grupo + 1)]
        self.outros = [p for p in self.grupo if p != nome]
        self.duracao = duracao
        self.atraso = atraso
        self.relogio = 0
        self.trava = threading.RLock()
        self.enviadas = {}
        # Exclusao mutua de Ricart e Agrawala
        self.estado = "LIBERADO"
        self.meu_pedido = None
        self.vaga = None
        self.oks = set()
        self.adiados = []
        # Eleicao pelo algoritmo do valentao: no inicio, o lider e o de maior numero
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
                if self.atraso:
                    threading.Timer(self.atraso, self.soquete.sendto, (dados, PROCESSOS[destino])).start()
                else:
                    self.soquete.sendto(dados, PROCESSOS[destino])
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
                dados, _ = self.soquete.recvfrom(4096)
            except OSError:
                continue
            self.receber(json.loads(dados.decode("utf-8")))

    def ao_receber_msg(self, m):
        pass

    # Exclusao mutua: pedido com o par (relogio de Lamport, numero do processo).
    def reservar(self, vaga):
        with self.trava:
            if self.estado != "LIBERADO":
                print("Ja existe um pedido em andamento.")
                return
            self.estado = "QUERENDO"
            self.vaga = vaga
            self.oks = set()
            self.enviar(self.outros, "PEDIDO", "pede a vaga %s" % vaga, {"vaga": vaga})
            self.meu_pedido = (self.relogio, numero(self.nome))
            print("  pedido (%d, %s)" % (self.relogio, self.nome))

    def ao_receber_pedido(self, m):
        pedido = (m["relogio"], numero(m["de"]))
        if self.estado == "DENTRO":
            self.adiados.append(m["de"])
            print("  adia a resposta a %s: esta na secao critica" % m["de"])
        elif self.estado == "QUERENDO" and self.meu_pedido < pedido:
            self.adiados.append(m["de"])
            print("  adia a resposta a %s: o proprio pedido (%d, %s) e menor que (%d, %s)"
                  % (m["de"], self.meu_pedido[0], self.nome, pedido[0], m["de"]))
        else:
            self.enviar([m["de"]], "OK", "responde OK")

    def ao_receber_ok(self, m):
        self.oks.add(m["de"])
        if self.estado == "QUERENDO" and len(self.oks) == len(self.outros):
            self.estado = "DENTRO"
            self.local("entra na secao critica: reserva a vaga %s" % self.vaga)
            print("  sai da secao critica em %g s" % self.duracao)
            threading.Timer(self.duracao, self.sair_da_secao).start()

    def sair_da_secao(self):
        with self.trava:
            print()
            self.estado = "LIBERADO"
            self.local("sai da secao critica")
            for destino in self.adiados:
                self.enviar([destino], "OK", "responde OK adiado")
            self.adiados = []
            print("%s> " % self.nome, end="", flush=True)

    # Eleicao pelo algoritmo do valentao.
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
            print("  %s e o novo lider" % self.nome)
            print("%s> " % self.nome, end="", flush=True)

    def ao_receber_eleicao(self, m):
        self.enviar([m["de"]], "RESPOSTA", "responde OK a ELEICAO")
        threading.Thread(target=self.iniciar_eleicao, daemon=True).start()

    def ao_receber_resposta(self, m):
        self.recebeu_resposta = True

    def ao_receber_coordenador(self, m):
        self.lider = m["de"]
        self.em_eleicao = False
        print("  novo lider: %s" % m["de"])


AJUDA = ("Comandos: local [N ou NOME] | envia DESTINO TEXTO | reservar VAGA | verificar | eleicao |\n"
         "          lider | contagem | relogio | sair")


def cmd_processo(args):
    nome = args[0].upper()
    if nome not in PROCESSOS:
        print("Processo desconhecido. Use P1 a P5.")
        sys.exit(1)
    grupo = int(args[args.index("--grupo") + 1]) if "--grupo" in args else 3
    duracao = float(args[args.index("--duracao") + 1]) if "--duracao" in args else 10.0
    atraso = float(args[args.index("--atraso") + 1]) if "--atraso" in args else 0.0
    if numero(nome) > grupo:
        print("O processo %s nao faz parte de um grupo de %d processos." % (nome, grupo))
        sys.exit(1)
    p = Processo(nome, grupo, duracao, atraso)
    print("Processo %s, porta UDP %d, grupo %s, lider inicial %s."
          % (nome, PROCESSOS[nome][1], " ".join(p.grupo), p.lider))
    if atraso:
        print("Cada mensagem enviada leva %g s para sair." % atraso)
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
        if partes[0] == "local" and len(partes) > 1 and not partes[1].isdigit():
            p.local("evento local " + partes[1])
        elif partes[0] == "local":
            for _ in range(int(partes[1]) if len(partes) > 1 else 1):
                p.local()
        elif partes[0] == "envia" and len(partes) >= 3 and partes[1].upper() in p.outros:
            p.enviar([partes[1].upper()], "MSG", "envia %s" % " ".join(partes[2:]))
        elif partes[0] == "reservar" and len(partes) == 2:
            p.reservar(partes[1])
        elif partes[0] == "verificar":
            p.verificar()
        elif partes[0] == "eleicao":
            p.iniciar_eleicao()
        elif partes[0] == "lider":
            print("Lider: %s" % p.lider)
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
    anterior = None
    for n, (valor, _, processo, texto) in enumerate(eventos, 1):
        empate = "   empate com o anterior, desfeito pelo numero do processo" if valor == anterior else ""
        print("%3d  L=%-4d %s %s%s" % (n, valor, processo, texto, empate))
        anterior = valor


if __name__ == "__main__":
    if "--mesma-maquina" in sys.argv:
        PROCESSOS = {nome: ("127.0.0.1", porta) for nome, (_, porta) in PROCESSOS.items()}
    if len(sys.argv) >= 3 and sys.argv[1] == "processo":
        cmd_processo(sys.argv[2:])
    elif len(sys.argv) >= 3 and sys.argv[1] == "ordenar":
        cmd_ordenar(sys.argv[2:])
    else:
        print("Uso: python3 reservas.py processo P1..P5 [--grupo N] [--duracao S] [--atraso S] [--mesma-maquina]")
        print("     python3 reservas.py ordenar P1.log P2.log P3.log")
        sys.exit(1)
```

---

## Referência bibliográfica

COULOURIS, George et al. **Sistemas distribuídos: conceitos e projetos**. 5. ed. Porto Alegre:
Bookman, 2013. Capítulo 15, coordenação e acordo, seções sobre exclusão mútua distribuída e
eleições.

GARCIA-MOLINA, Hector. Elections in a distributed computing system. **IEEE Transactions on
Computers**, v. C-31, n. 1, p. 48-59, 1982.

RICART, Glenn; AGRAWALA, Ashok K. An optimal algorithm for mutual exclusion in computer networks.
**Communications of the ACM**, v. 24, n. 1, p. 9-17, 1981.

VAN STEEN, Maarten; TANENBAUM, Andrew S. **Distributed Systems**. 4. ed. 2023. Capítulo 5,
coordenação, seções sobre exclusão mútua e algoritmos de eleição.
