---
title: Generative Adversarial Networks
tags:
  - machine-learning
---
## Generative Adversarial Networks
Generative Adversarial Networks (GANs) are a class of deep generative models comprising of two different models competing against each other in a zero-sum game.

![[Pasted image 20250526110456.png|center|400]]

The generator network maps random noise $z$ to a candidate generation $G(z)$ where $G$ is the generator. The discriminator network $D$ determines how real the generated candidate $G(z)$ is, represented by the number $D(G(z))$. During the training, the generator learns to produce candidates that are as real as possible, making it difficult for the discriminator network to distinguish between real and generated candidates.

The loss function used in the training is a min-max loss,
$$
\min_G \max_D \ \mathbb{E}_x[\log(D(x))] + \mathbb{E}_z[\log(1 - D(G(z))]
$$
The generator learns to minimize the loss, whereas the discriminator learns to maximize it. The generator is kept constant when the discriminator is being trained and vice-versa, helping the overall GAN model converge better. 

### Common Problems while training GANs
1. Vanishing gradients: If the discriminator is too good at classifying the generated candidates as fake instances, it doesn't provide enough information to the generator to improve the quality of the candidates. The generator stops improving due as the magnitude of the gradients during back-propagation decreases significantly.

2. Mode collapse: If the generator learns to produce an output plausible to discriminator, the discriminator's best move would be to learn to reject it. If, in the following iteration, the discriminator is not able to make this move, the generator will keep producing the same plausible output as it did in the previous iteration. Thus, the generator keeps producing outputs from a small subset instead of producing varied outputs.

3. Being a two-player game, training GANs is generally difficult and convergence is hard to achieve.

### Types of GANs
1. Wasserstein GANs: They use the Wasserstein loss instead of the traditional min-max loss, that helps solve problems of vanishing gradients and mode-collapse. The discriminator outputs a real number representing the quality of the generated candidates, thus acting like a 'critic'.

2. Conditional GANs: They are variants of GANs that can be conditioned (or biased) to produce a certain desired output, such as images of a 'cat', by providing images of cats to the generator and the discriminator.

3. StyleGANs: They are used to transfer the 'style' or unique characteristics of given image to another image/signal which does not contain those characteristics. The generator of the GAN is modified to produce images from two noise sources, where one of the sources can be perturbed to generate new images.

4. SRGANs: They are GANs used to perform super-resolution i.e. enhancing the resolution of the given signal. The generator produces a high-res image from a given low-res image, which is then matched against the 'real' high-res image by the discriminator.