---
title: "Ensinei uma IA a jogar o T-Rex do Chrome: do vídeo ao bot de 10 mil pontos"
date: 2026-10-05T17:00:00-03:00
tags: [machine-learning, visao-computacional, yolo, tensorflow-js, javascript]
description: "Como treinei um YOLOv8 com frames de um vídeo meu jogando, rodei o modelo no navegador e descobri, passo a passo, por que o bot errava até chegar a 10.500 pontos."
---

Eu jogo o dinossauro do Chrome desde sempre e nunca passei de umas poucas centenas de pontos. O meu bot, depois de uma semana de ajustes, fez **10.500 pontos**, a mais de 13 de velocidade, só olhando para a tela.

Este post conta como cheguei lá: treinei um modelo de visão computacional (YOLOv8) com frames de um vídeo meu jogando, coloquei esse modelo para rodar dentro do navegador e fui, passo a passo, descobrindo por que o bot errava. Spoiler: o modelo era a parte fácil. A maior parte do trabalho foi entender o que acontecia *entre* o modelo e o jogo.

> A ideia nasceu de uma aula do **Erick Wendel** que vi na pós-graduação da **[UNIPDS](https://unipds.com.br/)**, no curso de **Engenharia de Software com IA Aplicada**. Lá ele mostra como fazer uma IA jogar o DuckHunt-JS, e eu quis aplicar a mesma abordagem a outro jogo, treinando o meu próprio modelo.
>
> O código está em [github.com/rpaggi/t-rex-runner-machine-learning](https://github.com/rpaggi/t-rex-runner-machine-learning). Ele parte do clone open source do T-Rex Runner (wayou/t-rex-runner).

---

## A ideia

O bot não lê o código do jogo para saber onde estão os obstáculos. Ele **enxerga**, como um jogador: captura a imagem do canvas, pergunta ao modelo "o que tem aqui?" e decide se pula, abaixa ou continua correndo.

```
canvas do jogo ──► imagem ──► Web Worker (YOLOv8 + TensorFlow.js)
      ▲                                  │
      │                                  ▼
   pula / abaixa ◄── decisão ◄── caixas: cacto, dino, ptero
```

Tudo roda no navegador, sem servidor e sem Python em tempo de jogo.

---

## Etapa 1: os dados vieram de um vídeo meu jogando

Em vez de procurar um dataset pronto, eu **gravei a tela jogando** (um `.webm`) e extraí os frames. Cada frame mostra o dino, cactos e pterodáctilos em posições diferentes, que é exatamente o que o modelo precisa aprender.

Rotulei os frames no Roboflow com três classes:

- `cacto`
- `dino`
- `ptero`

O dataset final ficou pequeno: **137 imagens de treino, 25 de validação e 9 de teste**. O Roboflow redimensiona tudo para 640×640, esticando a imagem sem manter a proporção. Guarde esse detalhe, ele volta mais adiante.

Uma boa notícia sobre datasets pequenos: o jogo é visualmente simples (sprites pixelados, fundo claro), então poucos exemplos já bastam para um resultado bom.

---

## Etapa 2: treino com YOLOv8n

Usei o `yolov8n` (a versão "nano", a menor) com 30 épocas e imagens de 640×640. O treino levou cerca de 12 minutos. Resultado na validação:

| Métrica | Valor |
|---|---|
| Precisão | 0,95 |
| Recall | 0,99 |
| mAP50 | 0,96 |
| mAP50-95 | 0,90 |

A matriz de confusão mostrou o `dino` com 100% de acerto. Esses números são provavelmente otimistas: os frames de um mesmo vídeo se parecem muito entre si, então validação e treino têm "parentes". Mas foi o suficiente para seguir.

Depois exportei o modelo para **TensorFlow.js**: um `model.json` mais três arquivos `.bin` com os pesos, e um `labels.json` com os nomes das classes.

---

## Etapa 3: rodando o modelo no navegador, dentro de um Web Worker

Inferência é pesada e bloquearia o jogo se rodasse na thread principal, então ela vive num **Web Worker**. A thread principal captura o frame com `createImageBitmap(canvas)` e o transfere para o worker sem copiar.

A parte que mais ensina está na saída do modelo. O YOLOv8 exportado devolve **um único tensor `[1, 7, 8400]`**: para cada um dos 8400 candidatos, 4 números da caixa (centro x, centro y, largura, altura, em pixels de 640×640) e 3 notas, uma por classe. **Não há NMS embutido**, então eu mesmo fiz o pós-processamento:

1. Transpor para `[8400, 7]`.
2. Separar caixas e notas; pegar a maior nota e a classe de cada candidato.
3. Converter `(cx, cy, w, h)` em cantos `(x1, y1, x2, y2)`.
4. Rodar o NMS (`IoU 0.5`, nota mínima `0.4`, no máximo 10 caixas) para eliminar caixas duplicadas.

---

## Etapa 4: o bug que me custou mais tempo (e o que mais ensinou)

Ao rodar ao vivo, o modelo "alucinava": via `ptero` em nuvens e, pior, **nunca reconhecia o dino**. Mas na validação ele tinha 100% de acerto no dino. Algo estava errado entre o treino e o uso.

Para investigar, criei uma tecla **P** que pausa o jogo e imprime no console o tensor bruto e as detecções finais. Ao olhar a tabela, a pista veio das notas: a classe `dino` ficava em zero.

O culpado: **o canvas do jogo é transparente**. Ele é limpo com `clearRect`, então os pixels do fundo são `[0, 0, 0, 0]`. O `tf.browser.fromPixels` descarta o canal alfa, e o modelo recebia um dino cinza-escuro sobre um **fundo preto**. No treino, o fundo era branco. Para o modelo, era outro mundo.

A correção foram poucas linhas: desenhar o frame sobre um `OffscreenCanvas` branco antes de entregar ao TensorFlow.js.

```js
function flattenOnWhite(bitmap) {
  const canvas = new OffscreenCanvas(bitmap.width, bitmap.height)
  const ctx = canvas.getContext('2d')
  ctx.fillStyle = '#ffffff'
  ctx.fillRect(0, 0, canvas.width, canvas.height)
  ctx.drawImage(bitmap, 0, 0)
  return canvas
}
```

Depois disso: dino com 0,98 de confiança e as nuvens sumiram das detecções.

**Lição:** quando um modelo bom vai mal em produção, desconfie primeiro da *entrada*, não do modelo. Aqui eu tinha métricas excelentes e uma imagem de entrada completamente diferente da do treino.

---

## Etapa 5: do espaço do modelo para o espaço do jogo

As caixas saem em 640×640, mas o canvas do jogo tem 600×150. Como a imagem foi esticada para virar um quadrado, o eixo y encolhe quase 4 vezes na volta. A conversão é simples (`x * 600/640`, `y * 150/640`), mas esquecê-la faz toda decisão ficar errada.

Também montei um **overlay**: um segundo canvas por cima do jogo desenhando as caixas e uma linha de HUD com a decisão atual e a latência. Importante: desenhar no canvas do próprio jogo contaminaria o frame seguinte, e o modelo passaria a "ver" as próprias caixas.

---

## Etapa 6: decidir e agir

Com as caixas em coordenadas do jogo, a lógica de decisão ficou assim:

- Agrupar cactos próximos (um conjunto de 2 ou 3 cactos exige um único pulo).
- Calcular a distância do dino até o centro do próximo obstáculo.
- **Cacto:** pular quando faltar cerca de 0,3 s para chegar. Medir em segundos, e não em pixels, faz o limite crescer sozinho conforme o jogo acelera.
- **Ptero:** depende da altura. Alto, passa por cima e o dino só corre. Médio, o dino abaixa. Baixo, o dino pula.
- Agir chamando `tRex.startJump()` e `tRex.setDuck()` direto, sem simular teclado.

Dois erros meus apareceram nessa fase. Primeiro, classifiquei a altura do ptero pelo topo da caixa, e um ptero rente ao chão foi tratado como "médio": o dino abaixava e levava a pancada. O critério certo é saber se o **dino abaixado** passa por baixo, e isso depende da base da caixa. Segundo, a condição `distância <= limite` é verdadeira para qualquer número negativo, então o bot pulava para cactos que já tinha ultrapassado.

---

## Etapa 7: o inimigo é a latência

A inferência levava de 200 a 300 ms. A 7 de velocidade o jogo anda ~420 px/s, então ~100 px entre uma detecção e outra. O bot só decidia quando uma inferência voltava, e o pulo caía em instantes "quantizados", frequentemente atrasado.

Duas mudanças resolveram boa parte:

1. **Loop de controle a 60 fps.** Ele reaproveita a última detecção a cada frame de tela.
2. **Projeção pela idade da detecção.** Os obstáculos andam em velocidade conhecida, então dá para deslocar cada caixa pelo que ela já andou desde a captura. A posição estimada fica boa mesmo com a imagem "velha".

Também descartei frames enquanto o worker está ocupado (em vez de enfileirar imagens velhas) e passei a fechar todo `ImageBitmap` para não vazar memória a 5+ capturas por segundo.

---

## Etapa 8: parar de chutar e olhar os dados

Eu ajustava parâmetros "no olho" e não saía do lugar. Então fiz o bot registrar, a cada batida, uma tabela com as últimas 12 decisões: ação, distância, tipo e altura do alvo, velocidade, se estava pulando e a latência.

Com a tabela, os erros ficaram classificáveis:

- Batida com o dino **subindo**: pulou tarde.
- Batida com o dino **descendo**: pulou cedo, caiu em cima.
- Batida **sem pular**: o obstáculo não foi visto a tempo (percepção ou latência).

Isso permitiu que o bot **aprendesse com os próprios erros**: a cada batida ele ajusta o tempo de antecedência do pulo (para mais ou para menos), com um passo que vai diminuindo, e guarda o resultado no `localStorage`.

Por que não um algoritmo evolutivo clássico (mutar parâmetros e comparar pontuações)? Porque a pontuação é muito ruidosa: os obstáculos são sorteados e uma partida boa pode ser só sorte. Cada batida, em compensação, traz um sinal direcional claro, então ajustar a partir dela é muito mais barato.

---

## Etapa 9: a velocidade que não era a que eu achava

Nas tabelas, a maioria das batidas era "pulou cedo". Fui ler o código do jogo e achei:

```js
this.xPos -= Math.floor((speed * FPS / 1000) * deltaTime);
```

O `Math.floor` roda **a cada frame**, então a velocidade real dos obstáculos depende da taxa de atualização do monitor. Com velocidade 7, a 60 Hz o obstáculo anda 7 px por frame; a 144 Hz, onde cada frame dura ~7 ms, anda `floor(2,9) = 2` px por frame, isto é, bem mais devagar. O bot assumia `velocidade × 60` e achava que o obstáculo estava mais perto do que estava.

A correção foi repetir a mesma conta do jogo sobre os tempos reais dos últimos 60 frames e usar essa **velocidade efetiva** em todas as projeções. Resetei o valor aprendido (ele tinha aprendido a compensar o erro) e joguei de novo. Foi aí que vieram os **10.500 pontos**.

---

## O que eu levo disso

1. **Métricas boas não garantem nada em produção.** O modelo tinha mAP50 de 0,96 e mesmo assim falhava, porque a entrada era diferente da do treino.
2. **Faça ferramentas de debug cedo.** A tecla P e a tabela de batidas deram mais resultado do que qualquer ajuste de parâmetro.
3. **Latência é uma restrição de projeto.** Em jogo em tempo real, o tempo entre ver e agir limita a velocidade máxima que o bot aguenta, e compensar a idade da imagem rende muito.
4. **Leia o código do ambiente.** Um único `Math.floor` explicava um erro sistemático que eu vinha tentando consertar com parâmetros.
5. **Dataset pequeno e simples pode bastar.** 137 imagens e 12 minutos de treino resolveram a percepção, e o resto foi engenharia.

## Próximos passos

- Reduzir a latência: confirmar o backend (WebGL) e reexportar o modelo com entrada menor, já que o canvas é bem pequeno.
- Acelerar a descida depois do ápice do pulo, para estar pronto para o próximo obstáculo quando eles vêm em sequência.
- Mais frames de treino com o modo noite (cores invertidas) e a tela de game over.
- Aprender o tempo de abaixar do ptero da mesma forma que o do pulo.

---

## Referências

- [Repositório deste projeto](https://github.com/rpaggi/t-rex-runner-machine-learning)
- [wayou/t-rex-runner](https://github.com/wayou/t-rex-runner), o jogo original que usei como base, extraído do [código-fonte do Chromium](https://cs.chromium.org/chromium/src/components/neterror/resources/offline.js?q=t-rex+package:%5Echromium$&dr=C&l=7)
- [UNIPDS](https://unipds.com.br/), a pós-graduação em Engenharia de Software com IA Aplicada
- [Erick Wendel](https://github.com/ErickWendel), autor da aula e coordenador do curso da UNIPDS que deu origem à ideia
- [DuckHunt-JS](https://github.com/MattSurabian/DuckHunt-JS), o jogo usado no exemplo da aula
- [Ultralytics YOLOv8](https://docs.ultralytics.com), treino e [exportação para TensorFlow.js](https://docs.ultralytics.com/modes/export/)
- [TensorFlow.js](https://www.tensorflow.org/js), para rodar o modelo no navegador
- [Roboflow](https://roboflow.com), para rotular o dataset
- [Web Workers (MDN)](https://developer.mozilla.org/pt-BR/docs/Web/API/Web_Workers_API) e [OffscreenCanvas (MDN)](https://developer.mozilla.org/pt-BR/docs/Web/API/OffscreenCanvas)
