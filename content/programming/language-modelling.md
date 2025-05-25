---
title: Language Models
tags:
  - machine-learning
  - mathematics
  - "#notes"
---

##  Language Models
Language models are mathematical structures that map a sequence of tokens to a probability. The probability assigned to a sequence of tokens depends on how often the tokens are observed together in order. This mapping between the sequence of tokens and their associated probability is derived from vast quantities of textual data using a suitable learning rule. 

We define a sequence of tokens, $\boldsymbol{x} = [ x_1, x_2, ..., x_t ]$, and the joint probability distribution that the language model learns from empirical data,
$$
p(\boldsymbol{x}) =   p(x_1,x_2,...,x_t)
$$
The language model is also a generative model, as it models the joint probability distribution of the given data. We can simplify the above distribution,
$$
\begin{aligned}
p(x_1, x_2, ..., x_t) &= p(x_t|x_1, x_2, ..., x_{t-1}) \ p(x_1, x_2, ..., x_{t-1}) \\
&= p(x_t|x_1, x_2, ..., x_{t-1}) \ p(x_{t-1}|x_1, x_2, ..., x_{t-2}) \ p(x_1, x_2, ..., x_{t-2}) \\
&= p(x_1) \ \prod_{t=2}^T p(x_t|x_1, ..., x_{t-1})
\end{aligned}
$$
Consider the following example, where $\boldsymbol{x} = [Apple, sells, excellent, computers]$, the probability assigned by the language model to this sequence will be computed as,
$$
\begin{aligned}
p(Apple, sells, excellent, computers) &= p(computers|Apple, sells, excellent) \\ & \times p(excellent | Apple, sells) \\
& \times p(sells | Apple) \\
& \times p(Apple)
\end{aligned}

$$
As we observed, the probability associated with each token is dependent or conditioned on tokens that occur prior in the sequence. This gives us a hint, that language models are contextually rich models considering different parts of the sequence when assigning probability to a token.
### N-grams Model
Instead of considering the entire previous sequence $[x_1, x_2, ..., x_{t-1}]$ to compute the probability $p(x_t| x_1, x_2, ..., x_{t-1})$ used in the language model above, we only consider the past $N$ tokens from the sequence. 
$$
p(x_t | x_1, x_2, ..., x_{t-1}) \approx p(x_t| x_{t - N + 1}, x_{t - N + 2}, ..., x_{t - 1})
$$
With $N = 2$, we obtain the bigram model, where the probability of a token is dependent only on its previous token,
$$
p(x_t|x_1, x_2, ..., x_{t-1}) \approx p(x_t|x_{t-1})
$$
The assumption that the probability of a token depends only on the previous token in the sequence is called a Markov assumption.

#### Improving the N-grams Model: Using Log Probabilities
As observed in the expression of the language model, the desired probability $p(x_t| x_1, x_2, ..., x_{t-1})$ is a product of many probabilities, which might be very small if the text corpus under consideration is very large. Multiplying many small numbers together makes the product smaller making it 'underflow' the computer's memory. The product is so small that it is represented as a zero even though it is not.

Using logarithm solves the problem, converting a huge product expression to a summation over individual log probabilities,
$$
\begin{aligned}
p(x_1, x_2, ..., x_t) &= p(x_1) \ \prod_{t=2}^T p(x_t|x_1, ..., x_{t-1}) \\
\log(p(x_1, x_2, ..., x_t)) &= \log(p(x_1)) + \sum_{t=2}^T \log(p(x_t|x_1, ..., x_{t-1}))
\end{aligned}
$$

#### Improving the N-grams models: Using Smoothing Techniques
Just as we avoided smaller products by introducing log probabilities, we also need to avoid zero-ing out the entire product if only one of the individual probabilities was zero. If the probability of a token is calculated by computing its relative frequency in the corpus,
$$
p(x_i) = \frac{C(x_i)}{M}
$$
where $C(x_i)$ is the count of token $x_i$ in the corpus and $M$ is the total number of tokens in the corpus. We can add $1$ to the numerator and denominator, the process known as Laplace Smoothing,
$$
p^*(x_i) = \frac{C(x_i) + 1}{M + 1}
$$

### Evaluating Language Models: Perplexity
To evaluate how accurately a language model performs on a test-dataset (unobserved data), we can compute the perplexity of the model. It is defined as,
$$
\begin{aligned}
\text{perplexity}(x_1, x_2, ..., x_n) &= p(x_1, x_2, ..., x_n)^{-\frac{1}{n}} \\
&= \sqrt[n]{\frac{1}{p(x_1, x_2, ..., x_n)}}
\end{aligned}
$$
Lower the perplexity of the test sequence, the better the model. Perplexity also represents the uncertainty in the value of a sample being derived from a given distribution. Thus, a lower perplexity suggests that the chances of a sequence $[x_1, x_2, ..., x_n]$ being derived or sampled by the probability distribution constructed by the language model are high.


---
### References
- https://web.stanford.edu/~jurafsky/slp3/3.pdf
- https://arxiv.org/pdf/2303.05759
- https://en.wikipedia.org/wiki/Perplexity
- https://en.wikipedia.org/wiki/Word_n-gram_language_model