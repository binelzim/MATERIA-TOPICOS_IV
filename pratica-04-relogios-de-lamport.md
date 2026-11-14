# Prática 04: relógios lógicos de Lamport

**Disciplina:** Tópicos Especiais IV (Computação Distribuída)
**Curso:** Bacharelado em Sistemas de Informação, UNIPAM

A Prova 1 terminou com os relógios físicos. O NTP aproximou o relógio da máquina `aplicacao` do
relógio da máquina `banco`, e o serviço de pedidos voltou a aceitar as requisições. A aproximação,
porém, tem limite: mesmo sincronizados, dois relógios diferem em milissegundos, e dois eventos em
máquinas diferentes, separados por menos que esse erro, não podem ser ordenados com certeza pela
hora.

Esta prática trata da outra resposta ao problema, apresentada nos slides da aula. Em vez de
aproximar relógios físicos, cada processo mantém um contador, e as mensagens levam esse contador de
um processo a outro. O resultado não diz quando um evento aconteceu. Diz o que pode ter causado o
quê, que é a informação de que muitos sistemas precisam para decidir a ordem das operações.

A prática não pede entrega. Cada parte indica a saída esperada, e a conferência é feita pelo
próprio aluno, comparando o terminal com o roteiro e com as figuras dos slides. No fim de cada parte,
um quadro destaca os **valores para analisar**. São esses os valores que entram no relatório final
da sequência de práticas, descrito na seção 7.

---

## Sumário

| Seção | Assunto |
|---|---|
| Quadro de referência | máquinas, processos, portas e comandos, para consulta rápida |
| 1 | Fundamentação |
| 2 | Preparação do ambiente |
| 3 | Primeira parte: três processos com contador |
| 4 | Segunda parte: relógios em ritmos diferentes |
| 5 | Terceira parte: a ordem total a partir dos registros |
| 6 | O que esta prática prepara |
| 7 | O relatório final |
| 8 | Síntese |
| 9 | Questões para estudo |
| 10 | Referência de comandos |
| 11 | Diagnóstico de falhas |
| 12 | Apêndice com o conteúdo integral do programa |

---

## Objetivos de aprendizagem

Ao final desta prática o aluno deve ser capaz de:

- Aplicar as três regras do relógio lógico de Lamport a uma sequência de eventos locais, envios e
  recebimentos
- Identificar a relação aconteceu-antes entre eventos de processos diferentes a partir das
  mensagens trocadas
- Explicar por que L(a) < L(b) não prova que a aconteceu antes de b, com um exemplo observado
- Reconhecer, em relógios que avançam em ritmos diferentes, a mensagem que chega antes de sair, e
  explicar como a regra do recebimento elimina essa contradição
- Construir a ordem total dos eventos a partir dos registros dos processos e explicar o papel do
  desempate pelo número do processo

---

## Quadro de referência

**As máquinas, as mesmas das práticas anteriores**

| Papel | Nome | Endereço na rede exclusiva de hospedeiro |
|---|---|---|
| processo P2 | `banco` | 192.168.56.101 |
| processos P1 e P3 | `aplicacao` | 192.168.56.102 |

**Os processos e as portas**

| Processo | Máquina | Porta UDP | Ritmo na segunda parte |
|---|---|---|---|
| P1 | `aplicacao` | 9501 | 6 |
| P2 | `banco` | 9502 | 8 |
| P3 | `aplicacao` | 9503 | 10 |

**O diretório de trabalho, nas duas máquinas**

```
/home/SEU_USUARIO/pratica04
```

**Os comandos aceitos por um processo em execução**

| Comando | Efeito |
|---|---|
| `local` | um evento local |
| `local 3` | três eventos locais seguidos |
| `local b` | um evento local com o nome `b` |
| `envia P2 m1` | envia ao processo P2 uma mensagem com o texto `m1` |
| `relogio` | mostra o valor atual do relógio |
| `sair` | encerra o processo |

---

## 1. Fundamentação

### 1.1 Por que um contador basta

Num sistema distribuído, os processos não compartilham memória nem relógio. Cada um enxerga apenas
os próprios eventos e as mensagens que recebe. Dentro de um processo, a ordem dos eventos é
conhecida, porque eles acontecem um depois do outro. Entre processos diferentes, a única ligação
certa é a mensagem: o recebimento de uma mensagem acontece depois do envio dela, qualquer que seja
a hora marcada nos relógios das duas máquinas.

Lamport partiu dessas duas observações para definir a relação **aconteceu-antes**, escrita a → b.
O evento a aconteceu antes de b quando os dois estão no mesmo processo e a veio primeiro, quando a é
o envio de uma mensagem e b é o recebimento dela, ou quando existe um evento c tal que a → c e
c → b. Se nem a → b nem b → a, os dois eventos são **concorrentes**: nenhum deles pode ter
influenciado o outro.

Para essa relação, a hora não é necessária. Basta um número que cresça ao longo de cada processo e
que atravesse as mensagens. É o relógio lógico.

### 1.2 As três regras

Cada processo guarda um contador, que começa em zero.

1. Antes de cada evento, o processo avança o contador.
2. Ao enviar uma mensagem, o processo leva nela o valor do contador.
3. Ao receber uma mensagem, o processo fica com o maior valor entre o próprio contador e o da
   mensagem e, então, aplica a regra 1.

Com as três regras, vale a propriedade: se a → b, então L(a) < L(b). O inverso não vale. Dois
eventos concorrentes recebem números quaisquer, e um número menor não indica causa.

### 1.3 O ritmo do contador

Na formulação original, o contador avança uma unidade por evento. Nada muda na lógica se cada
processo avançar de um valor próprio, como os relógios da figura usada nos slides, em que P1 avança
de 6 em 6, P2 de 8 em 8 e P3 de 10 em 10. Esses ritmos diferentes representam relógios que andam em
velocidades diferentes, e mostram com números maiores o que acontece quando um processo está à
frente dos outros. O programa desta prática aceita o ritmo como opção.

### 1.4 Ordem total

O relógio de Lamport ordena parcialmente os eventos: eventos concorrentes podem ter o mesmo valor.
Para que todos os processos cheguem à mesma sequência, os empates são desfeitos por um critério que
todos conhecem, o número do processo. O par (valor do relógio, número do processo) nunca se repete,
e a ordem obtida respeita a relação aconteceu-antes. Essa ordem total é o que a terceira parte
monta a partir dos registros.

---

## 2. Preparação do ambiente

As duas máquinas virtuais das práticas anteriores, ligadas. Nada é instalado, e o programa usa
apenas a biblioteca padrão do Python.

Na máquina hospedeira, abrir **três** terminais: um com acesso à máquina `banco` e dois com acesso
à máquina `aplicacao`.

```bash
ssh SEU_USUARIO@192.168.56.101       # terminal do processo P2
```

```bash
ssh SEU_USUARIO@192.168.56.102       # terminal do processo P1
```

```bash
ssh SEU_USUARIO@192.168.56.102       # terminal do processo P3
```

Nas três sessões, criar o diretório de trabalho e entrar nele:

```bash
mkdir -p ~/pratica04
cd ~/pratica04
```

Abrir o arquivo `lamport.py` baixado do Portal num editor de texto da máquina hospedeira,
selecionar todo o conteúdo e copiar. Na sessão da máquina `banco`, abrir o editor, colar com
Ctrl+Shift+V, salvar com Ctrl+O e Enter, e sair com Ctrl+X:

```bash
nano -I lamport.py
```

Ainda na máquina `banco`, copiar o arquivo para a máquina `aplicacao`:

```bash
scp lamport.py SEU_USUARIO@192.168.56.102:~/pratica04/
```

O conteúdo integral do programa está no apêndice, na seção 12.

**Atenção.** Em todas as partes, os comandos são digitados na ordem indicada, um terminal de cada
vez. Depois de um `envia`, esperar a linha do recebimento aparecer no terminal do destino antes de
seguir para o passo seguinte. A ordem dos passos é o que define a figura que se forma.

---

## 3. Primeira parte: três processos com contador

### 3.1 Iniciar os processos

Nesta parte, o contador avança uma unidade por evento, como na formulação original. Iniciar um
processo em cada terminal:

```bash
python3 lamport.py processo P1       # terminal P1, maquina aplicacao
```

```bash
python3 lamport.py processo P2       # terminal P2, maquina banco
```

```bash
python3 lamport.py processo P3       # terminal P3, maquina aplicacao
```

Cada um mostra a porta, o ritmo e o valor inicial do relógio:

```
Processo P2, porta UDP 9502, ritmo 1, com o ajuste de Lamport.
Comandos: local [N ou NOME] | envia DESTINO TEXTO | relogio | sair
L=0 P2 inicio
P2>
```

### 3.2 A sequência

A sequência reproduz o exemplo dos slides, com os mesmos nomes de eventos: a e b em P1, c, d e e
em P2, f e g em P3. Os eventos d e g são recebimentos e recebem o nome da mensagem que chega.

| Passo | Terminal | Comando |
|---|---|---|
| 1 | P2 | `local c` |
| 2 | P1 | `envia P2 a` |
| 3 | P1 | `local b` |
| 4 | P3 | `local f` |
| 5 | P2 | `envia P3 e` |

Ao fim da sequência, os três terminais mostram as linhas abaixo, além do aviso de ajuste no
terminal P3, comentado na seção seguinte.

```
L=1 P1 envia a para P2
L=2 P1 evento local b
```

```
L=1 P2 evento local c
L=2 P2 recebe a de P1 (mensagem com 1)
L=3 P2 envia e para P3
```

```
L=1 P3 evento local f
L=4 P3 recebe e de P2 (mensagem com 3)
```

Os valores são os mesmos da figura dos slides: a = 1, b = 2, c = 1, d = 2, e = 3, f = 1 e g = 4.

### 3.3 Os dois recebimentos

No passo 2, P2 estava com o contador em 1 e recebeu uma mensagem com 1. O maior valor é 1, e a
regra 1 leva o contador a 2. A mensagem não trouxe valor maior que o de P2, e nada além da regra 1
foi aplicado.

No passo 5, P3 estava com o contador em 1 e recebeu uma mensagem com 3. O terminal P3 mostra:

```
  ajuste: o relogio iria de 1 para 2, a mensagem trouxe 3, o relogio vai para 4
L=4 P3 recebe e de P2 (mensagem com 3)
```

Sem a mensagem, P3 iria para 2. Como a mensagem trouxe 3, P3 fica com 3 e soma 1. O recebimento de
e passa a ter valor maior que o envio, como exige a relação aconteceu-antes.

### 3.4 O que os valores garantem

Seguir a cadeia de eventos ligados por mensagens: a → d, d → e dentro de P2, e → g. Os valores
crescem ao longo da cadeia: 1, 2, 3 e 4. Em toda cadeia de causa, o valor cresce.

Agora comparar b, em P1, com e, em P2. L(b) = 2 e L(e) = 3, e no entanto não existe caminho de
mensagens de b até e. O evento b aconteceu depois do envio de a, e nenhuma mensagem saiu de P1
depois dele. Os dois eventos são concorrentes, e o valor menor de b não indica que b influenciou e.

O mesmo vale para c, em P2, e f, em P3, que têm o mesmo valor, 1. Valores iguais só aparecem em
eventos concorrentes, porque eventos ligados por causa sempre têm valores diferentes.

**Valores para analisar na primeira parte**

| Evento | Processo | Valor |
|---|---|---|
| a, envio | P1 | 1 |
| b, local | P1 | 2 |
| c, local | P2 | 1 |
| d, recebimento de a | P2 | 2 |
| e, envio | P2 | 3 |
| f, local | P3 | 1 |
| g, recebimento de e | P3 | 4, com ajuste: iria para 2 e foi para 4 |

Os pares a analisar são três: a cadeia a, d, e, g, com valores crescentes, os eventos b e e, que
são concorrentes apesar de 2 ser menor que 3, e os eventos c e f, que são concorrentes e têm o mesmo
valor.

Encerrar os três processos com `sair`.

---

## 4. Segunda parte: relógios em ritmos diferentes

### 4.1 Sem o ajuste

Esta parte reproduz a figura dos slides, com P1 avançando de 6 em 6, P2 de 8 em 8 e P3 de 10 em
10. Primeiro, sem a regra do recebimento: cada processo apenas avança no próprio ritmo, como se a
mensagem não trouxesse informação nenhuma.

```bash
python3 lamport.py processo P1 --ritmo 6 --sem-ajuste      # terminal P1
```

```bash
python3 lamport.py processo P2 --ritmo 8 --sem-ajuste      # terminal P2
```

```bash
python3 lamport.py processo P3 --ritmo 10 --sem-ajuste     # terminal P3
```

### 4.2 A sequência das mensagens m1 a m4

| Passo | Terminal | Comando | O que acontece |
|---|---|---|---|
| 1 | P2 | `local` | P2 vai a 8 |
| 2 | P1 | `envia P2 m1` | m1 sai de P1 com 6 e chega a P2 |
| 3 | P3 | `local 3` | P3 vai a 10, 20 e 30 |
| 4 | P2 | `envia P3 m2` | m2 sai de P2 e chega a P3 |
| 5 | P2 | `local 2` | P2 avança dois eventos |
| 6 | P3 | `envia P2 m3` | m3 sai de P3 e chega a P2 |
| 7 | P1 | `local 6` | P1 avança seis eventos |
| 8 | P2 | `envia P1 m4` | m4 sai de P2 e chega a P1 |
| 9 | P1 | `local` | último evento de P1 |
| 10 | P2 | `local 2` | últimos eventos de P2 |
| 11 | P3 | `local 4` | últimos eventos de P3 |

Os recebimentos de m3, no terminal P2, e de m4, no terminal P1, aparecem assim:

```
  a mensagem saiu com 50 e chegou com 48: pelo relogio, chegou antes de sair
L=48 P2 recebe m3 de P3 (mensagem com 50)
```

```
  a mensagem saiu com 56 e chegou com 48: pelo relogio, chegou antes de sair
L=48 P1 recebe m4 de P2 (mensagem com 56)
```

Pelos valores, m3 e m4 chegam antes de sair. A ordem dos eventos fica contraditória, e qualquer
decisão tomada a partir desses valores, como saber qual operação veio primeiro, pode estar errada.
É a situação da figura "Sem ajuste" dos slides. Os recebimentos de m1 e de m2 não apresentam o
problema, porque P2 e P3 já estavam à frente de quem enviou.

**Valores para analisar sem o ajuste**

| Mensagem | Sai de | Valor no envio | Chega a | Valor no recebimento |
|---|---|---|---|---|
| m1 | P1 | 6 | P2 | 16 |
| m2 | P2 | 24 | P3 | 40 |
| m3 | P3 | 50 | P2 | **48** |
| m4 | P2 | 56 | P1 | **48** |

Encerrar os três processos com `sair`.

### 4.3 Com o ajuste

Iniciar os processos de novo, agora sem a opção `--sem-ajuste`:

```bash
python3 lamport.py processo P1 --ritmo 6       # terminal P1
```

```bash
python3 lamport.py processo P2 --ritmo 8       # terminal P2
```

```bash
python3 lamport.py processo P3 --ritmo 10      # terminal P3
```

Repetir os onze passos da tabela da seção 4.2, na mesma ordem. Os recebimentos de m3 e de m4
passam a mostrar o ajuste:

```
  ajuste: o relogio iria de 40 para 48, a mensagem trouxe 50, o relogio vai para 51
L=51 P2 recebe m3 de P3 (mensagem com 50)
```

```
  ajuste: o relogio iria de 42 para 48, a mensagem trouxe 59, o relogio vai para 60
L=60 P1 recebe m4 de P2 (mensagem com 59)
```

Depois do ajuste, cada processo continua no próprio ritmo. P2 segue com 59, 67 e 75, e por isso m4
sai com 59, e não com 56. P1 segue com 66. São os valores da figura "Com ajuste" dos slides, e
toda mensagem passa a chegar com valor maior que o do envio.

**Valores para analisar com o ajuste**

| Mensagem | Sai de | Valor no envio | Chega a | Sem o ajuste iria para | Valor no recebimento |
|---|---|---|---|---|---|
| m1 | P1 | 6 | P2 | 16 | 16 |
| m2 | P2 | 24 | P3 | 40 | 40 |
| m3 | P3 | 50 | P2 | 48 | **51** |
| m4 | P2 | **59** | P1 | 48 | **60** |

Comparar as duas tabelas: m1 e m2 não mudam, m3 e m4 mudam, e o valor de envio de m4 também muda,
porque o ajuste de m3 se propaga pelos eventos seguintes de P2.

Encerrar os três processos com `sair`. **Atenção.** Os registros desta execução são usados na
terceira parte, e cada novo início de um processo apaga o registro anterior dele. Não iniciar outra
execução antes de concluir a seção 5.

---

## 5. Terceira parte: a ordem total a partir dos registros

### 5.1 Os registros

Cada processo grava os próprios eventos num arquivo com o nome dele. Na máquina `aplicacao` estão
`P1.log` e `P3.log`, e na máquina `banco` está `P2.log`. Conferir o conteúdo de um deles:

```bash
cat P2.log                           # maquina banco
```

```
L=8 P2 evento local
L=16 P2 recebe m1 de P1 (mensagem com 6)
L=24 P2 envia m2 para P3
L=32 P2 evento local
L=40 P2 evento local
L=51 P2 recebe m3 de P3 (mensagem com 50)
L=59 P2 envia m4 para P1
L=67 P2 evento local
L=75 P2 evento local
```

Os registros são o que resta da execução depois que os processos terminam. É a mesma situação de
quem investiga um problema num sistema real: cada máquina guardou o que viu, e a sequência do
sistema inteiro tem de ser reconstruída a partir dessas partes.

### 5.2 Reunir os registros

Na máquina `banco`, copiar o registro de P2 para a máquina `aplicacao`:

```bash
scp P2.log SEU_USUARIO@192.168.56.102:~/pratica04/
```

### 5.3 Ordenar

Na máquina `aplicacao`, num dos terminais:

```bash
python3 lamport.py ordenar P1.log P2.log P3.log
```

A saída é a ordem total dos 27 eventos:

```
  1  L=6    P1 envia m1 para P2
  2  L=8    P2 evento local
  3  L=10   P3 evento local
  4  L=12   P1 evento local
  5  L=16   P2 recebe m1 de P1 (mensagem com 6)
  6  L=18   P1 evento local
  7  L=20   P3 evento local
  8  L=24   P1 evento local
  9  L=24   P2 envia m2 para P3   empate com o anterior, desfeito pelo numero do processo
 10  L=30   P1 evento local
 11  L=30   P3 evento local   empate com o anterior, desfeito pelo numero do processo
 12  L=32   P2 evento local
 13  L=36   P1 evento local
 14  L=40   P2 evento local
 15  L=40   P3 recebe m2 de P2 (mensagem com 24)   empate com o anterior, desfeito pelo numero do processo
 16  L=42   P1 evento local
 17  L=50   P3 envia m3 para P2
 18  L=51   P2 recebe m3 de P3 (mensagem com 50)
 19  L=59   P2 envia m4 para P1
 20  L=60   P1 recebe m4 de P2 (mensagem com 59)
 21  L=60   P3 evento local   empate com o anterior, desfeito pelo numero do processo
 22  L=66   P1 evento local
 23  L=67   P2 evento local
 24  L=70   P3 evento local
 25  L=75   P2 evento local
 26  L=80   P3 evento local
 27  L=90   P3 evento local
```

Executar de novo, com os arquivos em outra ordem:

```bash
python3 lamport.py ordenar P3.log P1.log P2.log
```

A sequência é a mesma. Ela depende apenas dos valores gravados, e não de onde nem de como os
registros foram lidos. Qualquer processo que reunisse os três registros chegaria à mesma ordem.

### 5.4 O que a ordem respeita

Localizar na saída o envio e o recebimento de cada mensagem. O envio de m1 está na posição 1 e o
recebimento na 5. O de m2, na 9 e na 15. O de m3, na 17 e na 18. O de m4, na 19 e na 20. Em todos
os casos, o envio vem antes do recebimento, porque a regra 3 garantiu valor maior ao recebimento.

Os quatro empates, nos valores 24, 30, 40 e 60, envolvem sempre eventos concorrentes. Nas posições
8 e 9, o evento local de P1 e o envio de m2 por P2 não têm nenhuma ligação por mensagem, e qualquer
dos dois poderia vir primeiro. O desempate pelo número do processo escolhe P1, e essa escolha é
arbitrária, mas é a mesma em todos os processos. É isso que torna a ordem total útil: ela não
descobre qual evento veio primeiro de fato, que é algo que ninguém no sistema consegue saber, e sim
garante que todos concordem sobre uma ordem que respeita as causas.

Na seção 4.1, sem o ajuste, essa ordem não seria possível. O recebimento de m3 teria valor 48 e o
envio, 50, e a ordenação colocaria o recebimento antes do envio.

**Valores para analisar na terceira parte**

| O que conferir | Resultado |
|---|---|
| total de eventos na ordem | 27 |
| empates, desfeitos pelo número do processo | nos valores 24, 30, 40 e 60 |
| posições de envio e recebimento de m1 | 1 e 5 |
| posições de envio e recebimento de m2 | 9 e 15 |
| posições de envio e recebimento de m3 | 17 e 18 |
| posições de envio e recebimento de m4 | 19 e 20 |
| ordem com os arquivos lidos em outra sequência | idêntica |

---

## 6. O que esta prática prepara

O programa `lamport.py` fica guardado no diretório `~/pratica04` das duas máquinas, porque as
próximas unidades partem dele.

Na Unidade 4, a exclusão mútua distribuída decide qual processo entra primeiro num recurso
disputado. No algoritmo de Ricart e Agrawala, cada pedido leva o par (relógio de Lamport, número do
processo), e o pedido de menor par tem prioridade, exatamente o critério da ordem total da seção 5.
Na Unidade 5, a replicação por máquina de estados exige que todas as réplicas apliquem as operações
na mesma ordem, e a ordem total é uma forma de obtê-la.

Em todos esses casos, o problema é o mesmo desta prática: processos sem relógio comum precisam
concordar sobre uma ordem, e a única informação confiável que eles têm é a que viaja nas mensagens.

---

## 7. O relatório final

As práticas desta sequência usam o mesmo programa, que ganha uma função a cada unidade. Ao fim da
sequência, um relatório reúne os resultados de todas elas e compara o que cada etapa mostrou. Para
montá-lo, cada prática é executada de novo, e os quadros de valores para analisar indicam o que
deve ser obtido e comparado.

Nada desta prática é entregue agora. O que vale levar dela é o domínio da sequência de comandos e
o entendimento de por que cada valor aparece, porque é isso que torna rápida a nova execução. Os
quadros desta prática que entram no relatório são três: os valores da primeira parte, as duas
tabelas de mensagens da segunda, sem e com o ajuste, e o resultado da ordem total da terceira.

---

## 8. Síntese

| Pergunta | Relógio físico | Relógio de Lamport |
|---|---|---|
| O que mede | a hora, com erro de milissegundos | a ordem causal entre eventos |
| Como se mantém | sincronização periódica, como o NTP | três regras aplicadas a cada evento |
| O que viaja na mensagem | os instantes da consulta | o valor do contador |
| O que garante | proximidade entre relógios | se a → b, então L(a) < L(b) |
| O que não garante | a ordem de eventos mais próximos que o erro | que L(a) < L(b) implique a → b |
| Como se obtém uma ordem única | não se obtém com certeza | desempate pelo número do processo |

---

## 9. Questões para estudo

1. Na seção 3.3, por que o recebimento de a em P2 não provocou ajuste, e o recebimento de e em P3
   provocou?
2. Os eventos b e e, da primeira parte, têm valores 2 e 3. Explicar, a partir das mensagens
   trocadas, por que esses valores não indicam que b influenciou e.
3. Na segunda parte, os recebimentos de m1 e de m2 não mostraram nenhum aviso, nem sem o ajuste.
   Explicar por quê.
4. Sem o ajuste, P2 recebeu m3 com o relógio em 48. Calcular o valor com o ajuste e mostrar que a
   mudança se propaga para o envio de m4 e para o recebimento em P1.
5. Na ordem total da seção 5.3, o evento local de P1 com valor 24 aparece antes do envio de m2.
   Essa é a ordem em que os eventos aconteceram de fato? Justificar.
6. Por que o desempate precisa ser o mesmo em todos os processos? O que aconteceria se P1 desfizesse
   os empates pelo número do processo e P2 pela ordem alfabética do texto do evento?
7. Um sistema de reservas recebe dois pedidos para a mesma vaga, em máquinas diferentes, com o mesmo
   valor de relógio de Lamport. Descrever como a ordem total decide qual pedido é atendido, e por que
   todas as máquinas chegam à mesma decisão.
8. Um colega propõe usar a hora do NTP no lugar do relógio de Lamport para ordenar os eventos da
   segunda parte. Em que situação essa troca produziria uma ordem que contradiz as mensagens?

---

## 10. Referência de comandos

**Iniciar um processo**

```bash
python3 lamport.py processo P1                       # ritmo 1, com ajuste
python3 lamport.py processo P1 --ritmo 6             # ritmo 6, com ajuste
python3 lamport.py processo P1 --ritmo 6 --sem-ajuste
```

**Dentro de um processo**

```
local                  um evento local
local 3                tres eventos locais
local b                um evento local com o nome b
envia P2 m1            envia a mensagem m1 ao processo P2
relogio                mostra o valor do relogio
sair                   encerra o processo
```

**Registros e ordem total**

```bash
cat P1.log
scp P2.log SEU_USUARIO@192.168.56.102:~/pratica04/   # maquina banco
python3 lamport.py ordenar P1.log P2.log P3.log
```

**Plano alternativo, com os três processos numa só máquina**

Se uma das máquinas virtuais não estiver disponível, os três processos podem executar na mesma
máquina, em três terminais, com a opção `--mesma-maquina` em todos eles. A lógica e os valores são
os mesmos, e nada atravessa a rede entre máquinas.

```bash
python3 lamport.py processo P1 --mesma-maquina
python3 lamport.py processo P2 --mesma-maquina
python3 lamport.py processo P3 --mesma-maquina
```

---

## 11. Diagnóstico de falhas

| Sintoma | Causa provável | Verificação |
|---|---|---|
| `OSError: [Errno 98] Address already in use` | o mesmo processo já está em execução em outro terminal | encerrar a outra instância com `sair`, conferindo com `ss -lunp` |
| O `envia` não mostra o recebimento no destino | o processo de destino não foi iniciado, ou está na máquina errada | conferir o quadro de referência: P2 na `banco`, P1 e P3 na `aplicacao` |
| Os valores diferem dos do roteiro | um passo foi executado fora de ordem, ou antes de o recebimento anterior aparecer | encerrar os três processos e repetir a sequência desde o primeiro passo |
| `Processo desconhecido` | nome digitado diferente de P1, P2 ou P3 | conferir o comando de início |
| `FileNotFoundError` no `ordenar` | o registro de P2 não foi copiado para a máquina `aplicacao` | repetir o `scp` da seção 5.2 |
| A ordem total não tem 27 eventos | os registros são de outra execução, ou de uma execução interrompida | repetir a seção 4.3 inteira e copiar o `P2.log` de novo |
| Erro de indentação ao executar | espaços perdidos na colagem | abrir de novo com `nano -I` e colar o conteúdo inteiro |

---

## 12. Apêndice com o conteúdo integral do programa

### 12.1 `lamport.py`, nas máquinas `banco` e `aplicacao`

```python
import json
import re
import socket
import sys
import threading

# Os tres processos da pratica: P1 e P3 na maquina aplicacao, P2 na maquina banco.
PROCESSOS = {
    "P1": ("192.168.56.102", 9501),
    "P2": ("192.168.56.101", 9502),
    "P3": ("192.168.56.102", 9503),
}


class Processo:
    def __init__(self, nome, ritmo, ajusta):
        self.nome = nome
        self.ritmo = ritmo
        self.ajusta = ajusta
        self.relogio = 0
        self.trava = threading.Lock()
        self.registro = open(nome + ".log", "w")
        self.soquete = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.soquete.bind(("0.0.0.0", PROCESSOS[nome][1]))

    def anotar(self, texto):
        linha = "L=%d %s %s" % (self.relogio, self.nome, texto)
        print(linha)
        self.registro.write(linha + "\n")
        self.registro.flush()

    # Regra 1: antes de cada evento, o relogio avanca.
    def avancar(self):
        self.relogio += self.ritmo

    def local(self, nome=""):
        with self.trava:
            self.avancar()
            self.anotar(("evento local " + nome).strip())

    # Regra 2: ao enviar, a mensagem leva o valor do relogio.
    def enviar(self, destino, texto):
        with self.trava:
            self.avancar()
            mensagem = {"de": self.nome, "relogio": self.relogio, "texto": texto}
            self.soquete.sendto(json.dumps(mensagem).encode("utf-8"), PROCESSOS[destino])
            self.anotar("envia %s para %s" % (texto, destino))

    # Regra 3: ao receber, o relogio fica maior que o valor trazido pela mensagem.
    def receber(self, mensagem):
        with self.trava:
            antes = self.relogio
            self.avancar()
            t = mensagem["relogio"]
            print()
            if self.relogio <= t:
                if self.ajusta:
                    print("  ajuste: o relogio iria de %d para %d, a mensagem trouxe %d, o relogio vai para %d"
                          % (antes, self.relogio, t, t + 1))
                    self.relogio = t + 1
                else:
                    print("  a mensagem saiu com %d e chegou com %d: pelo relogio, chegou antes de sair"
                          % (t, self.relogio))
            self.anotar("recebe %s de %s (mensagem com %d)" % (mensagem["texto"], mensagem["de"], t))
            print("%s> " % self.nome, end="", flush=True)

    def escutar(self):
        while True:
            dados, _ = self.soquete.recvfrom(4096)
            self.receber(json.loads(dados.decode("utf-8")))


def cmd_processo(args):
    nome = args[0].upper()
    if nome not in PROCESSOS:
        print("Processo desconhecido. Use P1, P2 ou P3.")
        sys.exit(1)
    ritmo = int(args[args.index("--ritmo") + 1]) if "--ritmo" in args else 1
    ajusta = "--sem-ajuste" not in args
    p = Processo(nome, ritmo, ajusta)
    print("Processo %s, porta UDP %d, ritmo %d, %s." % (nome, PROCESSOS[nome][1], ritmo,
                                                       "com o ajuste de Lamport" if ajusta else "sem ajuste"))
    print("Comandos: local [N ou NOME] | envia DESTINO TEXTO | relogio | sair")
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
            p.local(partes[1])
        elif partes[0] == "local":
            for _ in range(int(partes[1]) if len(partes) > 1 else 1):
                p.local()
        elif partes[0] == "envia" and len(partes) >= 3 and partes[1].upper() in PROCESSOS:
            p.enviar(partes[1].upper(), " ".join(partes[2:]))
        elif partes[0] == "relogio":
            print("L=%d" % p.relogio)
        elif partes[0] == "sair":
            break
        else:
            print("Comandos: local [N ou NOME] | envia DESTINO TEXTO | relogio | sair")


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
        print("Uso: python3 lamport.py processo P1|P2|P3 [--ritmo N] [--sem-ajuste] [--mesma-maquina]")
        print("     python3 lamport.py ordenar P1.log P2.log P3.log")
        sys.exit(1)
```

---

## Referência bibliográfica

COULOURIS, George et al. **Sistemas distribuídos: conceitos e projetos**. 5. ed. Porto Alegre:
Bookman, 2013. Capítulo 14, tempo e estados globais, seções sobre a relação aconteceu-antes e os
relógios lógicos.

LAMPORT, Leslie. Time, clocks, and the ordering of events in a distributed system.
**Communications of the ACM**, v. 21, n. 7, p. 558-565, 1978.

VAN STEEN, Maarten; TANENBAUM, Andrew S. **Distributed Systems**. 4. ed. 2023. Capítulo 5,
coordenação, seções sobre relógios lógicos e multicast com ordem total.
