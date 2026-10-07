---
title: "AI from Scratch, Part 1: A Neuron That Decides Whether You Need a Coat"
date: 2026-10-07T18:00:00-03:00
tags: [machine-learning, neural-networks, python, ai-from-scratch]
description: "I started with a maze, got lost and went back to basics: a single neuron in plain Python, no frameworks, learning whether I need a coat. Two training runs that blew up, a 'perfect' model that got one wrong, and the sigmoid that fixed it."
---

I've used YOLO and TensorFlow.js to [make an AI play Chrome's T-Rex game](/en/2026/10/05/ia-joga-t-rex-do-chrome/). It worked, but the model was a black box: I called `train`, waited and hoped. This time I wanted the opposite: **write every calculation by hand**, in plain Python, with no PyTorch and no TensorFlow, until I understood what happens inside a neural network.

This is the first part of that study.

> I used AI assistants as tutors. I asked them to explain one concept at a time and let me write the code myself.

---

## Before: the maze I got lost in

I started out ambitious. A robot in a 5×5 maze learning to reach the exit with **Q-Learning**, then a neural network with 16 neurons imitating the table the robot had learned, then 3,000 different mazes, two hidden layers…

It worked, but I didn't understand what was running. Every answer brought three new concepts: hidden layer, bias, tanh, gradient. At one point I literally typed "I think I give up".

The way out was to go back to the **simplest possible problem**: one neuron, one input, one question anybody understands.

---

## The problem: do I need a coat?

- Input: the temperature (in °C).
- Output: `1` if I need a coat, `0` if I don't.

That's it. No map, no rewards, no 16 neurons.

---

## Step 1: the examples

Before any neuron, the network needs examples with the right answer. Here, I'm the one who decides what's right:

```python
exemplos = [
    [0, 1], [5, 1], [10, 1], [15, 1], [20, 1], [22, 1],   # coat
    [23, 0], [25, 0], [30, 0], [35, 0], [40, 0],          # no coat
]
```

My first version jumped straight from 20° to 25°. That meant nothing told the network what happens in between, so it would have to guess. I added 22° and 23° to mark exactly where I change my mind.

(The code identifiers are in Portuguese: `exemplos` is examples, `peso` is weight, `vies` is bias, `prever` is predict, `erro` is error, `taxa` is learning rate.)

---

## Step 2: the neuron

A neuron is a calculation:

```python
def prever(temperatura, peso, vies):
    return temperatura * peso + vies
```

Weight and bias are the two **knobs**. "Training" means turning those knobs until the answers are good. With both at zero, the neuron answers `0` for everything. It even "gets" the hot days right, but only by luck.

---

## Step 3: turning the knobs by hand

Before letting the computer adjust anything, I tried to find values myself. The goal: cold above 0.5, hot below 0.5.

| Weight | Bias | What happened |
|---:|---:|---|
| −0.1 | 1.0 | drops too fast: crosses 0.5 around 5° |
| −0.042 | 1.0 | gentler: crosses 0.5 around 12° |
| −0.021 | 1.45 | too gentle: 40° still gives 0.61 |
| **−0.042** | **1.45** | **all correct**: the turning point falls between 22° and 23° |

I'll admit I just kept guessing numbers until it worked. But that's when it clicked what each knob does:

- **Weight** is the slope: how much the prediction changes per degree. With −0.1, each degree takes away 0.1 (a steep hill); with −0.042, only 0.042 (a gentle one). The negative sign means the prediction goes **down** as the temperature goes up.
- **Bias** is where the hill starts: the value at 0°. Raising the bias pushes the turning point to the right.

They work together: change one and the other has to compensate.

---

## Step 4: error and loss

I was judging "looks good" by reading the screen. The computer needs a number.

```python
erro = previsao - real
```

The **sign** of the error matters. Positive means I predicted too high; negative, too low. With my hand-picked values, the largest errors were at 22° (−0.47) and 23° (+0.48). That makes sense: the line has to cross 0.5 right there, while the right answer is 1 on one side and 0 on the other. A straight line can't "jump".

To get a single number, I use the **loss**: the mean of the squared errors (mean squared error). Squaring keeps +0.48 and −0.47 from cancelling each other out.

| Configuration | Loss |
|---|---:|
| weight −0.1, bias 1.0 | 3.23 |
| weight −0.042, bias 1.45 | 0.10 |

---

## Step 5: the computer adjusts the bias (and the first training run that blew up)

The rule uses the sign of the error. If, on average, I predicted too low, the bias goes up; too high, it goes down:

```python
vies = vies - taxa * media_erro
```

The learning rate (`taxa`) is the step size. Each pass through all the examples is an **epoch**.

On my first attempt, I computed the average using `erro ** 2`. The result:

```
OverflowError: (34, 'Numerical result out of range')
```

A square is always positive, so the bias **always went down**. The prediction dropped, the error grew, the next step was even bigger, and within a few epochs the prediction had more than 130 digits. Squaring erases exactly the information training needs: (−0.8)² and (+0.8)² are both 0.64.

The fix was **two sums with different jobs**: the error **with its sign** says which way to turn the knob, and the **squared** error only measures how bad things are.

With that, the bias went from 0 to **1.4045** on its own, close to the 1.45 I had guessed, with a slightly lower loss. And the steps got smaller as the error shrank: big at first, tiny near the target.

---

## Step 6: the weight too (and the second training run that blew up)

The weight rule has one extra detail: the error is multiplied by the temperature.

```python
soma_erros_peso += erro * temperatura
...
peso = peso - taxa_peso * media_erro_peso
```

The reason is to split the blame. At 0°, the weight isn't even part of the calculation, so it's not to blame for the error at all. At 40°, changing the weight by 0.01 moves the prediction by 0.4, so it's very much to blame.

With the same 0.1 learning rate as the bias, it blew up again, but differently:

```
weight:  0.65 → −35 → 1,922 → −105,082 → 5,742,556 → ...
```

The sign flipped every epoch and the number grew about 50× each time. This time the direction was right, but the **step was too big**: since temperatures go up to 40, the weight's push is up to 40 times larger than the bias's. Picture a ball kicked too hard in a valley: it overshoots the bottom and ends up higher on the other side.

I fixed it with **one learning rate per knob**: 0.1 for the bias and 0.001 for the weight. (The more common fix is to normalize the input, dividing the temperature by 40, so that a single rate works for both.)

After about 400 epochs, training settled at:

```
weight = −0.0336   bias = 1.2323   loss = 0.0931
```

That's the best possible line; I checked it against the closed-form linear regression formula and got the same numbers. The loss is lower than my guess. But then came the surprise:

```
22° → 22 × −0.0336 + 1.2323 = 0.49   ← below 0.5: "no coat". Wrong.
```

The "perfect" configuration **gets wrong an example my guess got right**.

The reason is what the loss measures: the distance to 1 or 0, not being on the right side of 0.5. To the loss, 0° predicting 1.23 ("overshooting" 1) is as bad as 22° predicting 0.49 (wrong side). So training accepts getting 22° wrong to bring the extremes closer. **Training optimizes exactly what you tell it to, not what you want.**

---

## Step 7: the sigmoid

The fix is to pass the line through a curve that squeezes any number into the range 0 to 1:

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
| sigmoid(z) | 0.002 | 0.12 | 0.50 | 0.88 | 0.998 |

With it, 0° can predict 0.9999 without "overshooting" 1. The extremes, which were already right, stop pulling, and training focuses on the turning point.

The update rule in the code stays **exactly the same**. That's not obvious: behind the scenes, the error is now measured with *cross-entropy*, and with it the update works out to the same formula. One line changed: `prever`.

With 10,000 epochs:

| Temperature | Prediction | Correct |
|---:|---:|---:|
| 0° | 1.0000 | 1 |
| 15° | 0.9865 | 1 |
| 20° | 0.7993 | 1 |
| **22°** | **0.5546** | **1** |
| **23°** | **0.4104** | **0** |
| 25° | 0.1788 | 0 |
| 40° | 0.0000 | 0 |

**All 11 examples on the right side.** The turning point is where `z = 0`, that is, `bias ÷ (−weight) = 13.0079 ÷ 0.5813 ≈ 22.4°`.

A curious detail: weight and bias **never stop growing**. Since my examples split perfectly, training keeps making the "S" steeper and steeper, almost a step, pushing 22° toward 1 and 23° toward 0. The turning point stays at 22.4°; only the "confidence" increases. In practice, you stop training when it's good enough.

---

## The final code

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
            soma_erros += erro                    # for the bias
            soma_erros_peso += erro * temperatura # for the weight

        vies -= taxa * soma_erros / len(exemplos)
        peso -= taxa_peso * soma_erros_peso / len(exemplos)

    for temperatura, real in exemplos:
        print(f"{temperatura}°: previsão {prever(temperatura, peso, vies):.4f}, certo {real}")

if __name__ == "__main__":
    main()
```

About 30 lines. A whole neuron, with training, and no libraries.

---

## What I take from this

1. **Start smaller than you think you need.** The maze had too many concepts at once. One neuron with one input taught me more in an afternoon.
2. **Tuning by hand before automating pays off.** After turning weight and bias by guessing, the training rule stopped being magic: it's the same guessing, just guided by the sign of the error.
3. **The sign of the error is the most important information in training.** Losing it (my `erro ** 2`) makes training always move the same way until it blows up.
4. **A step that's too big also blows up,** but differently: it oscillates, with the sign flipping every epoch.
5. **Lower loss is not the same as more correct answers.** Training optimizes what the loss measures, and it's up to you to pick a loss that measures what matters.
6. **An activation changes what the model can represent.** Swapping the line for a sigmoid fixed what no number of epochs would.

## Next steps

- **Step 2, "should I go to the beach?":** two inputs (temperature and chance of rain), one weight for each, and the weights showing how much each one matters.
- **Step 3, "is the weather pleasant?":** neither cold nor hot. One neuron can't represent "the middle is good", but **two hidden neurons** can. It's the hidden layer that tripped me up in the maze, now with a role I can actually understand.
- **Back to the maze**, where the 16 neurons will just be "step 3 with more sensors".

---

## References

- [Neural Networks, 3Blue1Brown](https://www.3blue1brown.com/topics/neural-networks): the video series that best explains gradients and backpropagation visually
- [Sigmoid function (Wikipedia)](https://en.wikipedia.org/wiki/Sigmoid_function)
- [Gradient descent (Wikipedia)](https://en.wikipedia.org/wiki/Gradient_descent)
- [Cross-entropy (Wikipedia)](https://en.wikipedia.org/wiki/Cross-entropy), the loss that keeps the update rule unchanged with the sigmoid
- [I Taught an AI to Play Chrome's T-Rex Game](/en/2026/10/05/ia-joga-t-rex-do-chrome/), the post where I used the frameworks I'm now opening up
