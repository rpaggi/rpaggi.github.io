---
title: "I Taught an AI to Play Chrome's T-Rex Game: From a Video to a 10,000-Point Bot"
date: 2026-10-05T17:00:00-03:00
tags: [machine-learning, computer-vision, yolo, tensorflow-js, javascript]
description: "How I trained a YOLOv8 model on frames from a video of myself playing, ran it in the browser, and figured out, step by step, why the bot kept crashing until it reached 10,500 points."
---

I've been playing Chrome's dinosaur game forever and never got past a few hundred points. My bot, after a week of tweaking, scored **10,500 points**, at a speed above 13, just by looking at the screen.

This post is about how I got there: I trained a computer vision model (YOLOv8) on frames from a video of myself playing, ran that model inside the browser, and, step by step, figured out why the bot kept failing. Spoiler: the model was the easy part. Most of the work was understanding what happened *between* the model and the game.

> The idea came from a class by **Erick Wendel** that I took in the **[UNIPDS](https://unipds.com.br/)** graduate program in **Software Engineering with Applied AI**. In it he shows how to make an AI play DuckHunt-JS, and I wanted to apply the same approach to another game, training my own model.
>
> The code is at [github.com/rpaggi/t-rex-runner-machine-learning](https://github.com/rpaggi/t-rex-runner-machine-learning). It builds on the open source clone of T-Rex Runner (wayou/t-rex-runner).

---

## The idea

The bot doesn't read the game's code to find out where the obstacles are. It **sees**, like a player: it captures the canvas image, asks the model "what's in here?" and decides whether to jump, duck or keep running.

```
game canvas ──► image ──► Web Worker (YOLOv8 + TensorFlow.js)
      ▲                                  │
      │                                  ▼
  jump / duck ◄── decision ◄── boxes: cacto, dino, ptero
```

Everything runs in the browser, with no server and no Python at game time.

---

## Step 1: the data came from a video of me playing

Instead of looking for a ready-made dataset, I **recorded my screen while playing** (a `.webm`) and extracted the frames. Each frame shows the dino, cacti and pterodactyls in different positions, which is exactly what the model needs to learn.

I labeled the frames in Roboflow with three classes (the names are in Portuguese, as they appear in the code):

- `cacto` (cactus)
- `dino`
- `ptero` (pterodactyl)

The final dataset was small: **137 training images, 25 for validation and 9 for testing**. Roboflow resizes everything to 640×640, stretching the image without keeping the aspect ratio. Keep that detail in mind; it comes back later.

Good news about small datasets: the game is visually simple (pixelated sprites, light background), so a few examples are enough for a good result.

---

## Step 2: training with YOLOv8n

I used `yolov8n` (the "nano" version, the smallest one) with 30 epochs and 640×640 images. Training took about 12 minutes. Validation results:

| Metric | Value |
|---|---|
| Precision | 0.95 |
| Recall | 0.99 |
| mAP50 | 0.96 |
| mAP50-95 | 0.90 |

The confusion matrix showed `dino` at 100% accuracy. These numbers are probably optimistic: frames from the same video look a lot alike, so validation and training have "relatives" in each other. But it was enough to move on.

Then I exported the model to **TensorFlow.js**: a `model.json` plus three `.bin` weight files, and a `labels.json` with the class names.

---

## Step 3: running the model in the browser, inside a Web Worker

Inference is heavy and would block the game if it ran on the main thread, so it lives in a **Web Worker**. The main thread captures the frame with `createImageBitmap(canvas)` and transfers it to the worker without copying.

The most instructive part is the model's output. The exported YOLOv8 returns **a single `[1, 7, 8400]` tensor**: for each of the 8400 candidates, 4 numbers for the box (center x, center y, width, height, in 640×640 pixels) and 3 scores, one per class. **There's no built-in NMS**, so I did the post-processing myself:

1. Transpose to `[8400, 7]`.
2. Split boxes and scores; take the highest score and the class of each candidate.
3. Convert `(cx, cy, w, h)` into corners `(x1, y1, x2, y2)`.
4. Run NMS (`IoU 0.5`, minimum score `0.4`, at most 10 boxes) to remove duplicate boxes.

---

## Step 4: the bug that cost me the most time (and taught me the most)

When running live, the model "hallucinated": it saw `ptero` in clouds and, worse, **never recognized the dino**. But in validation it had 100% accuracy on the dino. Something was wrong between training and usage.

To investigate, I added a **P** key that pauses the game and prints the raw tensor and the final detections to the console. Looking at the table, the clue came from the scores: the `dino` class stayed at zero.

The culprit: **the game canvas is transparent**. It's cleared with `clearRect`, so background pixels are `[0, 0, 0, 0]`. `tf.browser.fromPixels` drops the alpha channel, and the model was getting a dark gray dino on a **black background**. In training, the background was white. To the model, it was a different world.

The fix was a few lines: draw the frame onto a white `OffscreenCanvas` before handing it to TensorFlow.js.

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

After that: dino at 0.98 confidence, and the clouds disappeared from the detections.

**Lesson:** when a good model performs badly in production, suspect the *input* first, not the model. Here I had excellent metrics and an input image completely different from the training ones.

---

## Step 5: from model space to game space

The boxes come out in 640×640, but the game canvas is 600×150. Since the image was stretched into a square, the y axis shrinks almost 4 times on the way back. The conversion is simple (`x * 600/640`, `y * 150/640`), but forgetting it makes every decision wrong.

I also built an **overlay**: a second canvas on top of the game that draws the boxes and a HUD line with the current decision and the latency. Important: drawing on the game's own canvas would contaminate the next frame, and the model would start "seeing" its own boxes.

---

## Step 6: deciding and acting

With the boxes in game coordinates, the decision logic looked like this:

- Group nearby cacti (a cluster of 2 or 3 cacti needs a single jump).
- Compute the distance from the dino to the center of the next obstacle.
- **Cactus:** jump when it's about 0.3 s away. Measuring in seconds, not pixels, makes the threshold grow on its own as the game speeds up.
- **Ptero:** depends on the height. High, it flies over and the dino just keeps running. Medium, the dino ducks. Low, the dino jumps.
- Act by calling `tRex.startJump()` and `tRex.setDuck()` directly, without simulating the keyboard.

Two of my mistakes showed up in this phase. First, I classified the ptero's height by the top of its box, and a ptero flying close to the ground was treated as "medium": the dino ducked and took the hit. The right criterion is whether the **ducking dino** fits underneath, and that depends on the bottom of the box. Second, the condition `distance <= threshold` is true for any negative number, so the bot jumped for cacti it had already passed.

---

## Step 7: latency is the enemy

Inference took 200 to 300 ms. At speed 7 the game moves ~420 px/s, so ~100 px between one detection and the next. The bot only decided when an inference came back, and the jump landed at "quantized" moments, often late.

Two changes solved a good part of it:

1. **A 60 fps control loop.** It reuses the latest detection on every screen frame.
2. **Projection by detection age.** Obstacles move at a known speed, so each box can be shifted by how far it has traveled since capture. The estimated position stays good even with a "stale" image.

I also dropped frames while the worker is busy (instead of queuing old images) and started closing every `ImageBitmap` so as not to leak memory at 5+ captures per second.

---

## Step 8: stop guessing and look at the data

I was tuning parameters by eye and getting nowhere. So I made the bot log, on every crash, a table with its last 12 decisions: action, distance, target type and height, speed, whether it was jumping, and latency.

With the table, the errors became classifiable:

- Crash with the dino **going up**: jumped late.
- Crash with the dino **coming down**: jumped early and landed on it.
- Crash **without jumping**: the obstacle wasn't seen in time (perception or latency).

This allowed the bot to **learn from its own mistakes**: on every crash it adjusts how early it jumps (more or less), with a shrinking step size, and stores the result in `localStorage`.

Why not a classic evolutionary algorithm (mutate parameters and compare scores)? Because the score is very noisy: obstacles are random, and a good run can be just luck. Each crash, on the other hand, carries a clear directional signal, so adjusting from it is much cheaper.

---

## Step 9: the speed that wasn't what I thought

In the tables, most crashes were "jumped early". I went to read the game's code and found:

```js
this.xPos -= Math.floor((speed * FPS / 1000) * deltaTime);
```

The `Math.floor` runs **on every frame**, so the real obstacle speed depends on the monitor's refresh rate. At speed 7, at 60 Hz an obstacle moves 7 px per frame; at 144 Hz, where each frame lasts ~7 ms, it moves `floor(2.9) = 2` px per frame, which is much slower. The bot assumed `speed × 60` and thought the obstacle was closer than it actually was.

The fix was to repeat the game's own calculation over the real timings of the last 60 frames and use this **effective speed** in every projection. I reset the learned value (it had learned to compensate for the error) and played again. That's when the **10,500 points** came.

---

## What I take away from this

1. **Good metrics guarantee nothing in production.** The model had an mAP50 of 0.96 and still failed, because the input was different from the training data.
2. **Build debugging tools early.** The P key and the crash table paid off more than any parameter tuning.
3. **Latency is a design constraint.** In a real-time game, the time between seeing and acting limits the maximum speed the bot can handle, and compensating for the image's age pays off a lot.
4. **Read the environment's code.** A single `Math.floor` explained a systematic error I had been trying to fix with parameters.
5. **A small, simple dataset can be enough.** 137 images and 12 minutes of training solved perception; the rest was engineering.

## Next steps

- Reduce latency: confirm the backend (WebGL) and re-export the model with a smaller input, since the canvas is quite small.
- Speed up the descent after the peak of the jump, to be ready for the next obstacle when they come in sequence.
- More training frames with night mode (inverted colors) and the game over screen.
- Learn the ptero ducking timing the same way as the jump timing.

---

## References

- [This project's repository](https://github.com/rpaggi/t-rex-runner-machine-learning)
- [wayou/t-rex-runner](https://github.com/wayou/t-rex-runner), the original game I used as a base, extracted from the [Chromium source code](https://cs.chromium.org/chromium/src/components/neterror/resources/offline.js?q=t-rex+package:%5Echromium$&dr=C&l=7)
- [UNIPDS](https://unipds.com.br/), the Software Engineering with Applied AI graduate program
- [Erick Wendel](https://github.com/ErickWendel), who taught the class and coordinates the UNIPDS program that inspired the idea
- [DuckHunt-JS](https://github.com/MattSurabian/DuckHunt-JS), the game used in the class example
- [Ultralytics YOLOv8](https://docs.ultralytics.com), for training and [exporting to TensorFlow.js](https://docs.ultralytics.com/modes/export/)
- [TensorFlow.js](https://www.tensorflow.org/js), to run the model in the browser
- [Roboflow](https://roboflow.com), to label the dataset
- [Web Workers (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API) and [OffscreenCanvas (MDN)](https://developer.mozilla.org/en-US/docs/Web/API/OffscreenCanvas)
