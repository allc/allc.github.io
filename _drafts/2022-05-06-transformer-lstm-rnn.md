---
layout: post
title: Transformer, LSTM and RNN
tags: [AI, ML, NLP, NLU, Natural Language Generation, Machine Translation]
---

This post has an unconventional order talking about these three topics, while many other tutorials talks them in the reverse order. This is to align with how I learn the structure- from putting everything together, then look into to details. Despite the things were evolved in a different order.

This is not aim to provide a comprehensive tutorial about these model architectures, some awesome tutorials are listed in [Readings](#readings) near the bottom of this post.

## Transformer

### Transformer Architecture

{% include figure.html src="/assets/image/transformer-lstm-rnn/transformer.png" caption="Transformer architecture, figure reproduced from [[1]](#reference-transformer)." %}

{% include figure.html src="/assets/image/transformer-lstm-rnn/multi-head-attention.png" caption="Multi-head Attention, figure reproduced from [[1]](#reference-transformer)." %}

{% include figure.html src="/assets/image/transformer-lstm-rnn/scaled-dot-product-attention.png" caption="Scaled Dot-Product Attention, figure reproduced from [[1]](#reference-transformer)." %}

Transformer uses Multi-Head Attention with Scaled Dot-Product Attention.

### Dataflow in Transformer

## Readings



## Reference

<span id="reference-transformer">[1] Ashish Vaswani, Noam Shazeer, Niki Parmar, Jakob Uszkoreit, Llion Jones, Aidan N Gomez, Łukasz
Kaiser, and Illia Polosukhin. Attention is all you need. In I. Guyon, U. Von Luxburg, S. Bengio,
H. Wallach, R. Fergus, S. Vishwanathan, and R. Garnett, editors, <i>Advances in Neural Information
Processing Systems</i>, volume 30. Curran Associates, Inc., 2017.<span>