# Scope

I don't really like writing these sorts of things, to be honest. I usually prefer to just keep my head down and code, but I wanted to document this process before I forget it all. I guess I wanted to understand how machine learning actually works under the hood. The best way to do that, I suppose, is to just dive into the deep end and see if you can swim out. 

So, I built a neural network completely from scratch using C++. No frameworks, no external maths libraries. Just standard C++. I'm definitely not an expert, and this whole project was incredibly humbling, but here are my notes, the roadblocks I hit, and why I ended up building it the way I did.

Code: [Here](https://github.com/nir659/neural-network)

---

## Step 1: The Tensor Engine and Memory
I started by creating a `tensor` struct. Instead of using nested vectors (like `std::vector<std::vector<float>>`), I backed it with a single, flat `std::vector<float>`. 

I decided to go with a flat vector for two primary reasons: cache locality and preventing memory fragmentation. CPUs like reading memory in contiguous lines. The naive triple for-loop I wrote for matrix multiplication hoists `a[i,k]` and reads `b[k,j]` contiguously in the innermost loop to keep the CPU cache happy. 

But there was a steep cost to this. By abandoning nested vectors, I sacrificed intuitive bounds checking. Having to manually calculate `index = row * cols + col` everywhere led to my first major failure. I messed up the indexing logic early on during matrix transposition, resulting in silent memory corruption and shape mismatches. It took me an embarrassing amount of time to track down. I thought it would be easy, but I was wrong.

---
## Step 2: Layer Abstraction
I tried to follow the principle of single responsibility, keeping the maths completely decoupled from the training logic. 

I created a `dense_layer` that is strictly a mathematical operator. It pre-allocates everything in its constructor, generating its weight matrix using Xavier uniform initialization so the variance of activations stays stable across layers. Its only job during the forward pass is to compute `Z = X * W + b`. It knows how to transform an input and how to calculate a gradient, but it does *not* know how to update its own weights. 

---
## Step 3: Activations and the Dispatch Problem
For the activation, I wrote a standalone, pure function for ReLU that just applies `std::max(0.0f, x)` element-wise. 

When wiring the network together, each stage is just a `dense_layer` and an activation pair. I used a simple function pointer for the activation: `using activation_fn = tensor (*)(const tensor &);`. I specifically avoided C++ inheritance and virtual functions here. In a neural network, activations are called millions of times. A virtual function requires a vtable lookup at runtime, which prevents the compiler from inlining the code.

Any C++ dev reading this will probably wonder: *why not just use templates for maximum inlining?* Honestly, I tried. But it quickly turned into compile-time template hell. Trying to define the entire network architecture at compile time made dynamically adding layers or changing the shape a nightmare for me. The function pointer was a quiet compromise, a slight overhead compared to a direct call, but it allowed the network to remain flexible without the abstraction tax of object-oriented hierarchies.

---
## Step 4: Backpropagation
When I started designing the backpropagation logic, I hit a massive wall. I honestly felt completely out of my depth. I initially tried to calculate the gradients just by looking at the final outputs and the loss, which failed completely. The network learned nothing. 

I realised that calculating the gradients isn't that simple. I had to explicitly save the state from the forward pass because multivariable calculus strictly requires it.

* **X (Inputs):** To calculate the weight gradients, I need to know exactly what values were multiplied by those weights in the first place. The derivative of the loss with respect to the weights is essentially the incoming error gradient multiplied by the transpose of the input. Without caching the input, I'd have no way to complete that multiplication.
* **Z (Pre-activations):** For activations like ReLU, the gradient depends entirely on the value *before* the non-linearity was applied. Since ReLU's derivative is 1 if `Z > 0` and 0 otherwise, I had to store `Z` to act as a binary mask. If I only had the final output, I wouldn't know which neurons were originally suppressed, making the chain rule impossible to execute.

I did some research into how big frameworks like PyTorch handle this, just to make sure I wasn't doing something completely wrong. They use similar caching mechanisms because you just can't escape the maths. 

The systems trade-off here is pretty severe, though. By caching every intermediate `X` and `Z`, I am trading a massive amount of RAM to save CPU cycles. The memory footprint of the network essentially triples during training compared to inference. It felt wasteful, but it's a necessary evil.

---
## Step 5: The Optimiser
Because the layers don't update themselves, I built a dedicated optimiser struct for Stochastic Gradient Descent (SGD). It's just a plain struct with a single `step()` function that mutates the network parameters in place. It iterates through every stage's weights and biases in lockstep with the matching gradient entry, applying `parameter -= learning_rate * gradient`. No virtual dispatch, no heap allocation.

By separating this, I made the system modular. I could theoretically swap my SGD out for Adam tomorrow without touching a single line of code in the dense layer.

---
## Step 6: Data Pipeline
I was a bit lazy and just downloaded the free MNIST CSV dataset from Kaggle rather than generating synthetic data. I wrote a CSV parser to extract the floats straight from the disk into my tensor structs.

But raw data doesn't just work out of the box. I had to add a normalisation step, dividing the pixel values (0–255) by 255.0 to squeeze them into a 0.0–1.0 range, otherwise the network gradients would explode. I also had to convert the scalar labels (e.g., "5") into one-hot encoded arrays (e.g., `[0, 0, 0, 0, 0, 1, 0, 0, 0, 0]`) so the Mean Squared Error (MSE) loss function could actually compute a meaningful distance.

Finally, I built a `dataloader` to handle batching and shuffling using a reproducible RNG seed so I could actually feed the network in memory-friendly mini-batches.

---
## Step 7: Testing and the Gradient Checker
Before I even tried to train the model, I wrote a test suite. The test suite covers tensor operations, forward/backward correctness, optimiser updates, and a numerical gradient checker that validates back propagation against finite differences. 

The gradient checker was the most important part of this entire project; without it, I would have been flying blind. It acts as a numerical sanity test for backprop. I put it in `tests/test_gradient_check.cpp`. 

It builds a small, deterministic 2-layer net with fixed inputs and targets so the results are perfectly reproducible. First, it computes the analytical gradients via normal backprop (forward pass, caching, loss gradient, and network backward). Then, for every single weight and bias, it computes a numerical gradient using central differences: `numerical = (L(theta + eps) - L(theta - eps)) / (2*eps)`. 

It compares the analytical results against the numerical ones using a mixed tolerance (an absolute tolerance of 2e-3 and a relative tolerance of 2e-2). This matters because it catches all the insidious bugs in the backward maths, wrong transposes, bad bias reductions, or chain rule mistakes, by checking the gradients directly against how the loss function actually behaves.

---
## Step 8: The Reality Check
I wired it all together in `main.cpp`. Here is the output from a recent run:

```text
nir@pibble:~~/D/1/M/neural-network$ ./build/test_network
...
64 / 64 tests passed

nir@pibble:~/D/1/M/neural-network$ ./build/neural_net
Epoch 1 loss: 0.0703494
Epoch 2 loss: 0.0460912
Epoch 3 loss: 0.0400578
Epoch 4 loss: 0.0364848
Epoch 5 loss: 0.0339429
...
Epoch 10 loss: 0.0270176
```

The steadily decreasing loss indicates the gradient pipeline is wired correctly, though this alone doesn’t guarantee generalisation.

Looking back, the biggest insight I gained wasn't just memorising the multi variable calculus. It was realising how unforgiving systems engineering is when applied to maths. A single misplaced transpose or a tiny memory overlap doesn't throw a neat, readable exception; it just silently prevents the network from learning. Translating abstract theory into a performant, hardware-aware system in C++ was far more brutal than I anticipated, but it forced me to actually understand what I was building.