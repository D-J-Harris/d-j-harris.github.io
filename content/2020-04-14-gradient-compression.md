---
title: Gradient Compression in Distributed Deep Learning
description: "Introduction to distributed deep learning, and in particular gradient compression: a technique used to reduce communication overhead between machines when training a large deep learning model."
date: 2020-04-14
toc: true
taxonomies:
  tags: [machine learning]
---

## Introduction to Deep Learning

Deep Learning (DL) has burst onto the scene as one of the most powerful tools we currently have for tackling problems
in a range of domains, from machine translation and image captioning, to object and image detection, to speech
recognition and much more. Making the distinction between deep learning and machine learning is an important one - much
in the way that machine learning is one approach to investigating artificial intelligence, deep learning is a single
implementation of machine learning. Particularly, DL focuses on artificial neural networks, abstractions of the function
of the brain that aim to model data distributions through co-learning of layers of computational nodes, or neurons.

For a long time this concept wasn't able to gain traction, since deep neural networks (DNNs) have many nuts and bolts to
tune, and there just wasn't the compute power or data available to do it with. However, along with advances in the two
(especially practicality of GPU usage), DNNs began to achieve state of the art results in many fields. A
landmark point was the emergence of convolutional neural networks (CNNs) in the field of image recognition in 2012
[Krizhevsky et al.](http://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks.pdf).
Results on the ImageNet task were still taking close to weeks of compute time, but there was a lot of promise.

Today, this hunger for more data and more computation only grows. For example in the field of natural language
processing, transformer architectures are shown to have a correlation between performance and model complexity,
resulting in an almost arms-race for unthinkably large models that ask for more and more from us... questions of research
accessibility of these models and impact of large-scale training on the environment aside, let's ask the question of
how we can even begin to make this kind of problem scalable.

## Scaling Up Model Training

Introducing: distributed deep learning. The idea is that in training a DNN, we should be able to save
both time and resources by distributing the workload from one single machine, to multiple machines. Teamwork makes the
dream work.

There are many important dimensions to talk about when it comes to distributed deep learning. I want to briefly cover
the two main aspects here before pushing into the main topic of compression techniques, however please do check out
these extensive surveys to delve deeper into further considerations, such as system architecture and
gradient aggregation methods.
([Mayer et al.](https://arxiv.org/pdf/1903.11314.pdf),
[Chahal et al.](https://arxiv.org/pdf/1810.11787.pdf),
[Tang et al.](https://arxiv.org/pdf/2003.06307.pdf))

### Synchronisation

In the standard setting, a single model is trained to learn a set of parameters \\(\theta\\) that minimise an objective function
defined on the task at hand. In doing this, the model should be generalisable - it has learned a way to map data from
a given distribution (e.g. the pixels in an image of cats), to the target result (e.g. is it a cat). This process is
called optimisation, and a popular algorithm for this is Stochastic Gradient Descent (SGD). Mathematically this
iterative method takes the form:

\\[\mathbf{\theta}\_{t+1} = \mathbf{\theta}_t - \eta \nabla\_{\theta} J(\mathbf{x}\_t; \theta)\\]

for objective function \\(J\\), and sample of our data \\(\boldsymbol{x}\_t\\) at timestep \\(t\\). The way this sample is taken already
allows us to determine some of the speed with which the model trains: moving from the stochastic setting of one data
point to a larger batch size of many data points can lead to computational speedup due to parallelism in GPUs. Perfect!
However there is a tradeoff, where this can harm the generalisability of the model.

- _Synchronous SGD_: these 'gradient descent' steps can be simply extended to a multi-worker setting. If batches of data
  are split up among multiple machines, then we can afford to use more at once. Each machine, or worker, takes a copy of
  the model and performs the update step above on its chunk of the data. These \\(\nabla\_{\theta}\\) updates are then
  aggregated to a central model to complete the process. One key problem with this is that there can be single failure
  points if any one worker is too slow.

- _Asynchronous SGD_: Workers could instead send their step updates as and when they have them, eliminating this
  straggler problem and increasing parallelisation. However in practice this technique is not preferred, as it can lead to
  instability in model convergence, due to some workers maybe training on 'stale' models.

### Parallelism

- _Data Parallelism_: In this setting, the data that we want to train on is split into non-overlapping batches, and
  distributed to every worker which has identical copies of the model. Here, because each worker is sharing parameters of
  the model, then they have to communicate their updates (aka what they've learned about the parameters from their data
  chunk) regularly. This lends data parallelism being a better scheme for compute-intensive, but low-parameter-number
  models such as CNNs.

- _Model Parallelism_: In the case where a model is too big to fit in memory on a single device, it is useful to rather
  split the model into layered chunks to train separately. This splitting is often done using reinforcement learning. Due
  to the feed-forward nature of DNNs, this method requires heavy communication of updates and potential bottlenecks,
  making data parallelism more common practice.

## How Costly is Update Communication?

Pretty costly. For example, the model size of ResNet-50 is over
110MB - calculating gradients for making gradient descent updates contain millions of floating point values, and so the
size of these scale proportionally. Considering a standard Ethernet connection can have a speed of around 1Gbps, then
this makes scaling up these distributed systems a problem, as communication between workers can prove to be a bottleneck
or rate limiting factor in training models at scale, thus under-utilising computing resources.

There are three main methods in tackling reducing communication overhead in distributed deep learning. These are:

- Reducing Model Precision - by changing the size of the model parameters, for example from a common 32-bit
  floating point representation to 16-bit, then exchanged gradients are inherently smaller in memory and so cost less to
  communicate.

- Improving Communication Scheduling - imagine you want to print something, but everybody else hits print at the same
  time, and now you have to wait much longer to get your result back. By being smart about which times gradients should
  be sent, distributed systems can avoid exceeding bandwidth limits and therefore not bottleneck the process.

- Compressing Gradients - this is where my work will step in. This area looks at the ability to compress gradients to
  smaller sizes before they are communicated, in such a way that enough information is retained to have minimal impact
  on the accuracy of the resulting model. Due to compression limitation these techniques are
  often lossy, and fall under mainly two broad categories: gradient quantisation, and gradient sparsification (Figure 1).

### Gradient Quantisation

The idea behind gradient quantisation is to reduce the bit-representation of gradients to make them less costly to
communicate. There are three popular approaches to quantisation:

[Seide et al.](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/IS140694.pdf)
demonstrate that gradients can be compressed to just 1-bit (a representation of their sign) and communicated to provide
speedups of up to 100 times on a large 160 million parameter model. As with many techniques in the field, this uses an
error-feedback scheme, where quantisation errors are accumulated and added to the next batch, which can achieve the same
rate of convergence as SGD
([Karimireddy et al.](https://arxiv.org/pdf/1901.09847.pdf)).

[Wen et al.](https://arxiv.org/pdf/1705.07878.pdf)
introduce TernGrad, a scheme that works by quantising on only three
numerical levels {-1,0,1}. Unlike 1-bit quantisation, this is applicable also to gradients arising from convolutional
layers. The authors use two techniques to be able to additionally prove the convergence of a
model under this quantisation, under the assumption of a bound on the gradients.

- Firstly,when ternarising gradients a scaler function is applied that scales the operation
  relative to the largest absolute value in the gradient. This ternarising is therefore done on a layer-by-layer basis,
  to avoid scaling by global absolute values, and therefore provide a stronger bound on the gradients.
- Secondly, gradient clipping is used to address the issue of dynamic ranges in the bound gap.

[Alistarh et al.](https://arxiv.org/pdf/1610.02132.pdf)
introduce Quantised SGD (QSGD), which is a family of algorithms
that are able to give a tradeoff between bits communicated and variance of the compressed gradients. This allows for
tradeoff between bandwidth communication and model convergence time. In words, what QSGD does is takes a probabilistic
approach on quantisation, using an encoding scheme that emphasises the likelihood of gradient values into its compression.
Compared to TernGrad, this method has no hyperparameters and guarantees convergence under standard assumptions only. However,
[Wen et al.](https://arxiv.org/pdf/1705.07878.pdf)
point out that eliminating accuracy loss in QSGD requires at least 16 levels (4-bit) of encoding, whereas this is
achieved with only 3 levels using TernGrad.

### Gradient Sparsification

The other family of compression techniques is called gradient sparsification, which involves dropping any gradient
values which transmit little to no information in parameter updates. Note that the compression rate
of previously mentioned quantisation methods is at best 32 times, due to the limitation in how many bits can be
reduced from the standard 32-bit gradients. Therefore compression rates can be greatly helped by sparsification.

[Aji et al.](https://arxiv.org/pdf/1704.05021.pdf)
note that gradient updates are positively skewed, as most updates are near zero. This motivates the use of cheaper
communication via sparse matrix transmission, which is done by mapping a large fraction of updates to zero. This
fraction threshold is expensive to find, however can be approximated as 99% through sampling.

[Strom et al.](https://www.isca-speech.org/archive/interspeech_2015/papers/i15_1488.pdf)
adopt a similar method, however choose a static threshold value above which gradients should be zero. This resulted in
high speed gains of 54 times for only a 1.8% error on a particular speech recognition task, however it should be noted
that the threshold was a hyperparameter that was difficult to tune.
[Dryden et al.](https://dl.acm.org/doi/10.5555/3018874.3018875)
on the other hand employ an adaptive threshold technique, which determines a threshold of gradients to communicate every
iteration that results in a fixed compression ratio throughout training. This involves extra overhead compute time in
sorting gradient vectors, which is addressed in more recent works.

## Further Reading

This quick tour of distributed deep learning reflects on the richness in
this area for future study. There are a host of more recent exciting advances, from Sketch communication
[Ivkin et al.](https://arxiv.org/pdf/1903.04488.pdf),
to low-rank approximation gradient compression
[Vogels et al.](https://arxiv.org/pdf/1905.13727.pdf)
and more. I've even left out of my description the current most used gradient compression technique, Deep Gradient
Compression (DGC)
[Lin et al.](https://arxiv.org/pdf/1712.01887.pdf).
This technique enjoys impressive whole-model gradient compression ratios of up to 600 times(!) using momentum correction
and masking to address issues with staleness in error accumulation and feedback.
