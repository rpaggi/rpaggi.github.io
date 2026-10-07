---
title: "IA do zero, parte 1: um neurônio que decide se você precisa de casaco"
date: 2026-10-07T18:00:00-03:00
tags: [machine-learning, redes-neurais, python, ia-do-zero]
description: "Comecei pelo labirinto, me perdi e voltei ao básico: um único neurônio em Python puro, sem frameworks, aprendendo a decidir se preciso de casaco. Dois treinos que explodiram, um modelo 'perfeito' que errava e a sigmoide que resolveu."
---

Eu já usei YOLO e TensorFlow.js para [fazer uma IA jogar o T-Rex do Chrome](/2026/10/05/ia-joga-t-rex-do-chrome/). Funcionou, mas o modelo era uma caixa-preta: eu chamava `train`, esperava e torcia. Desta vez eu quis o contrário: **escrever cada conta na mão**, em Python puro, sem PyTorch e sem TensorFlow, até entender o que acontece dentro de uma rede neural.

Esta é a primeira parte desse estudo.

> Usei assistentes de IA como professores. Pedi que me explicassem um conceito por vez e me deixassem escrever o código.

---

## Antes: o labirinto em que eu me perdi

Comecei ambicioso. Um robô num labirinto 5×5 aprendendo a chegar na saída com **Q-Learning**, depois uma rede neural com 16 neurônios imitando a tabela que o robô aprendeu, depois 3.000 labirintos diferentes, duas camadas ocultas…

Funcionou, mas eu não entendia o que estava rodando. Cada resposta trazia três conceitos novos: camada oculta, viés, tanh, gradiente. Chegou um ponto em que eu escrevi, literalmente, "acho que desisto".

A saída foi voltar para o problema **mais simples possível**: um neurônio, uma entrada, uma pergunta que qualquer pessoa entende.

---

## O problema: preciso de casaco?

- Entrada: a temperatura.
- Saída: `1` se preciso de casaco, `0` se não preciso.

Só isso. Nada de mapa, nada de recompensa, nada de 16 neurônios.

---

## Etapa 1: os exemplos

Antes de qualquer neurônio, a rede precisa de exemplos com a resposta certa. Aqui, quem decide o que é certo sou eu:

```python
exemplos = [
    [0, 1], [5, 1], [10, 1], [15, 1], [20, 1], [22, 1],   # casaco
    [23, 0], [25, 0], [30, 0], [35, 0], [40, 0],          # sem casaco
]
```

Na primeira versão eu tinha pulado de 20° direto para 25°. Ou seja, eu não dizia em lugar nenhum o que acontecia entre os dois, e a rede teria que adivinhar. Coloquei 22° e 23° para marcar exatamente onde eu mudo de ideia.

---

## Etapa 2: o neurônio

Um neurônio é uma conta:

```python
def prever(temperatura, peso, vies):
    return temperatura * peso + vies
```

Peso e viés são os dois **botões de ajuste**. "Treinar" é girar esses botões até as respostas ficarem boas. Com os dois em zero, o neurônio responde `0` para tudo. Ele até "acerta" os dias quentes, mas por sorte.

---

## Etapa 3: girando os botões na mão

Antes de deixar o computador ajustar, tentei achar valores sozinho. O objetivo: frio acima de 0,5, calor abaixo de 0,5.

| Peso | Viés | O que aconteceu |
|---:|---:|---|
| −0,1 | 1,0 | desce rápido demais: já passa de 0,5 perto de 5° |
| −0,042 | 1,0 | mais suave: passa de 0,5 perto de 12° |
| −0,021 | 1,45 | suave demais: 40° ainda dá 0,61 |
| **−0,042** | **1,45** | **acerta todos**: a virada fica entre 22° e 23° |

Confesso que fui chutando números até acertar. Mas foi aí que caiu a ficha do que cada botão faz:

- **Peso** é a inclinação: quanto a previsão muda a cada grau. Com −0,1, cada grau tira 0,1 (uma ladeira íngreme); com −0,042, tira só 0,042 (uma ladeira suave). O sinal negativo quer dizer que a previsão **desce** quando a temperatura sobe.
- **Viés** é de onde a ladeira começa: o valor com 0°. Subir o viés empurra o ponto de virada para a direita.

Os dois trabalham juntos: mexeu em um, o outro precisa compensar.

---

## Etapa 4: erro e perda

Eu estava julgando "está bom" olhando a tela. O computador precisa de um número.

```python
erro = previsao - real
```

O **sinal** do erro importa. Positivo quer dizer que previ demais; negativo, que previ de menos. Com os meus valores na mão, os maiores erros eram os de 22° (−0,47) e 23° (+0,48). Faz sentido: a reta precisa passar por 0,5 ali, e o certo é 1 de um lado e 0 do outro. Uma reta não consegue "pular".

Para ter um número só, uso a **perda**, a média dos erros ao quadrado (o erro quadrático médio). O quadrado serve para que +0,48 e −0,47 não se cancelem.

| Configuração | Perda |
|---|---:|
| peso −0,1, viés 1,0 | 3,23 |
| peso −0,042, viés 1,45 | 0,10 |

---

## Etapa 5: o computador ajusta o viés (e o primeiro treino que explodiu)

A regra usa o sinal do erro. Se, na média, previ de menos, o viés sobe; se previ demais, ele desce:

```python
vies = vies - taxa * media_erro
```

A `taxa` é o tamanho do passo. Cada passada por todos os exemplos é uma **época**.

Na minha primeira tentativa, eu calculei a média usando `erro ** 2`. O resultado:

```
OverflowError: (34, 'Numerical result out of range')
```

O quadrado é sempre positivo, então o viés **sempre descia**. A previsão caía, o erro crescia, o passo seguinte era ainda maior, e em poucas épocas a previsão tinha mais de 130 dígitos. O quadrado apaga justamente a informação de que o treino precisa: (−0,8)² e (+0,8)² dão os dois 0,64.

A correção foi usar **duas somas com trabalhos diferentes**: o erro **com sinal** diz para que lado girar o botão, e o erro **ao quadrado** só mede o quanto está ruim.

Com isso, o viés foi de 0 até **1,4045** sozinho, perto do 1,45 que eu tinha achado no chute, com perda um pouco menor. E os passos foram ficando menores conforme o erro diminuía: grandes no começo, minúsculos perto do alvo.

---

## Etapa 6: o peso também (e o segundo treino que explodiu)

A regra do peso tem um detalhe a mais: o erro é multiplicado pela temperatura.

```python
soma_erros_peso += erro * temperatura
...
peso = peso - taxa_peso * media_erro_peso
```

O motivo é dividir a culpa. Com 0°, o peso nem entra na conta, então não tem culpa nenhuma do erro. Com 40°, mexer 0,01 no peso muda a previsão em 0,4, então ele tem muita culpa.

Com a mesma taxa de 0,1 do viés, explodiu de novo, mas de um jeito diferente:

```
peso:  0,65 → −35 → 1.922 → −105.082 → 5.742.556 → ...
```

O sinal trocava a cada época e o número crescia umas 50 vezes. Dessa vez a direção estava certa, mas o **passo era grande demais**: como as temperaturas vão até 40, o empurrão do peso é até 40 vezes maior que o do viés. É uma bola chutada forte demais num vale, que passa do fundo e para mais alto do outro lado.

Resolvi com **uma taxa para cada botão**: 0,1 para o viés e 0,001 para o peso. (A solução mais comum é normalizar a entrada, dividindo a temperatura por 40, para que uma taxa só sirva para os dois.)

Depois de umas 400 épocas, o treino parou em:

```
peso = −0,0336   viés = 1,2323   perda = 0,0931
```

É a melhor reta possível; conferi com a fórmula exata da regressão linear, e dá os mesmos números. A perda é menor que a do meu chute. Mas aí vem a surpresa:

```
22° → 22 × −0,0336 + 1,2323 = 0,49   ← abaixo de 0,5: "sem casaco". Errou.
```

A configuração "perfeita" **erra um exemplo que o meu chute acertava**.

O motivo é o que a perda mede: a distância até 1 ou 0, e não o lado certo do 0,5. Para ela, o 0° prever 1,23 ("passou" de 1) é tão ruim quanto o 22° prever 0,49 (lado errado). Então o treino aceita errar o 22° para aproximar as pontas. **O treino otimiza exatamente o que você manda, não o que você quer.**

---

## Etapa 7: a sigmoide

A solução é passar a reta por dentro de uma curva que espreme qualquer número entre 0 e 1:

```python
import math

def sigmoide(z):
    return 1 / (1 + math.exp(-z))

def prever(temperatura, peso, vies):
    z = temperatura * peso + vies
    return sigmoide(z)
```

| z | −6 | −2 | 0 | +2 | +6 |
|---|---:|---:|---:|---:|---:|
| sigmoide(z) | 0,002 | 0,12 | 0,50 | 0,88 | 0,998 |

Com ela, o 0° pode prever 0,9999 sem "passar" de 1. As pontas, que já estavam certas, param de puxar, e o treino se concentra na virada.

A regra de ajuste do código continua **igualzinha**. Isso não é óbvio: por trás, a forma de medir o erro passa a ser a *entropia cruzada*, e com ela a conta do ajuste fica a mesma. Mudou uma linha, a do `prever`.

Com 10.000 épocas:

| Temperatura | Previsão | Certo |
|---:|---:|---:|
| 0° | 1,0000 | 1 |
| 15° | 0,9865 | 1 |
| 20° | 0,7993 | 1 |
| **22°** | **0,5546** | **1** |
| **23°** | **0,4104** | **0** |
| 25° | 0,1788 | 0 |
| 40° | 0,0000 | 0 |

**Os 11 exemplos do lado certo.** O ponto de virada é onde `z = 0`, ou seja, `viés ÷ (−peso) = 13,0079 ÷ 0,5813 ≈ 22,4°`.

Um detalhe curioso: peso e viés **nunca param de crescer**. Como os meus exemplos têm uma divisão perfeita, o treino vai deixando o "S" cada vez mais íngreme, quase um degrau, e o 22° vai subindo para perto de 1 e o 23° descendo para perto de 0. O ponto de virada fica parado em 22,4°; só a "certeza" aumenta. Na prática, para-se o treino quando está bom o bastante.

---

## O código final

```python
import math

exemplos = [
    [0, 1], [5, 1], [10, 1], [15, 1], [20, 1], [22, 1],
    [23, 0], [25, 0], [30, 0], [35, 0], [40, 0],
]

def sigmoide(z):
    return 1 / (1 + math.exp(-z))

def prever(temperatura, peso, vies):
    return sigmoide(temperatura * peso + vies)

def main():
    peso, vies = 0.0, 0.0
    taxa, taxa_peso = 0.1, 0.001

    for epoca in range(10000):
        soma_erros = 0.0
        soma_erros_peso = 0.0
        for temperatura, real in exemplos:
            erro = prever(temperatura, peso, vies) - real
            soma_erros += erro                    # para o viés
            soma_erros_peso += erro * temperatura # para o peso

        vies -= taxa * soma_erros / len(exemplos)
        peso -= taxa_peso * soma_erros_peso / len(exemplos)

    for temperatura, real in exemplos:
        print(f"{temperatura}°: previsão {prever(temperatura, peso, vies):.4f}, certo {real}")

if __name__ == "__main__":
    main()
```

São umas 30 linhas. É um neurônio inteiro, com treino, sem nenhuma biblioteca.

---

## O que eu levo disso

1. **Comece menor do que você acha que precisa.** O labirinto tinha conceitos demais ao mesmo tempo. Um neurônio com uma entrada me ensinou mais em uma tarde.
2. **Ajustar na mão antes de automatizar vale a pena.** Depois de girar peso e viés no chute, a regra do treino deixou de ser mágica: é o mesmo chute, só que guiado pelo sinal do erro.
3. **O sinal do erro é a informação mais importante do treino.** Perdê-lo (o meu `erro ** 2`) faz o treino andar sempre para o mesmo lado até explodir.
4. **Passo grande demais também explode,** e de um jeito diferente: oscilando, com o sinal trocando a cada época.
5. **Perda menor não é o mesmo que acertar mais.** O treino otimiza o que a perda mede, e cabe a você escolher uma perda que meça o que importa.
6. **Uma ativação muda o que o modelo consegue representar.** Trocar a reta pela sigmoide resolveu o que nenhuma quantidade de épocas resolveria.

## Próximos passos

- **Degrau 2, "vou à praia?":** duas entradas (temperatura e chance de chuva), um peso para cada, e o peso mostrando o quanto cada coisa importa.
- **Degrau 3, "o clima está agradável?":** nem frio nem quente. Um neurônio não consegue representar "o meio é bom", mas **dois neurônios ocultos** conseguem. É a camada oculta que me travou no labirinto, agora com um papel que dá para entender.
- **Voltar ao labirinto**, onde os 16 neurônios vão ser só "o degrau 3 com mais sensores".

---

## Referências

- [Neural Networks, 3Blue1Brown](https://www.3blue1brown.com/topics/neural-networks): a série em vídeo que melhor explica gradiente e retropropagação visualmente
- [Sigmoid function (Wikipedia)](https://en.wikipedia.org/wiki/Sigmoid_function)
- [Gradient descent (Wikipedia)](https://en.wikipedia.org/wiki/Gradient_descent)
- [Cross-entropy (Wikipedia)](https://en.wikipedia.org/wiki/Cross-entropy), a perda que faz a regra de ajuste continuar igual com a sigmoide
- [Ensinei uma IA a jogar o T-Rex do Chrome](/2026/10/05/ia-joga-t-rex-do-chrome/), o post em que usei os frameworks que agora estou abrindo por dentro
