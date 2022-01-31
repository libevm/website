---
title: ExJpeg - AI Experiments on NFT Exploration and Curation
date: 2022-01-31
---

While playing around in NFT land, one of my pain points was that apart from the list of pre-defined properties, there wasn't really a good way to filter out NFTs that were outside the scope of those objective filters.

One example of this would be the curation of [clown masks](https://clownbanks.com) from [thehashmasks project](https://www.thehashmasks.com/):

![](https://i.imgur.com/1R6rPg3.png)

Despite clown masks not being an explicit mask type in [thehashmasks](https://www.thehashmasks.com/) it has garnered enough love from the community, and has cemented itself in hashmasks lore.

*Slight disclaimer, it is also my personal favorite mask.*

![](https://i.imgur.com/e8DQl62.png)

Trying to search for these masks felt incredibly janky, and got me thinking if there was a better way to explore NFT based on more innate, visual properties.

Spoiler: Yes there is, and someone on the internet has already done that.

## Curse of Dimensionality

Images are high dimensional data -- each pixel contains information that contributes to the whole picture. For example, a `n` width by `m` height grayscale image would have `n x m` dimensions.

So how can we map high dimensional data onto a 2d plane?

### Dimensionality Reduction

> Why waste time say lot word when few word do trick? -- Tim Cook

Dimensionality reduction is just another way of saying "compression algorithm" (stupid technical jargon smh). The end goal is the same -- you want to reduce the size of some data, but you'd like to retain its key features.

For example, the 2d data below when projected onto the purple vector loses the least amount of information -- the newly projected data has the least amount of variance (relative to its original form).

![](https://i.imgur.com/xMpyAzt.gif)

##### Source: [sagarsaha455](https://sagarsaha455.medium.com/pca-for-visualization-and-dimension-reduction-14492e2acf2b)

With that in mind, I'll reiterate the goal -- to compress high dimensional image data into a 2d space, while retaining its key features.

### Autoencoders

Just like skinning a cat, there are many ways to perform dimensionality reduction. My personal favorite is autoencoders -- the concept clicked really early on for me and I can explain the rational using simple words.

An autoencoder is a type of neural network that attempts to:

1. Encode input images into a new representation (latent space)
2. Reconstruct the new representation back to its original image

![](https://i.imgur.com/WcXwfPN.png)

##### Source: [birla.deepak26](https://medium.com/@birla.deepak26/autoencoders-76bb49ae6a8f)

In order for the autoencoder to efficiently reconstruct the original image (from the latent space) with minimal losses, it needs to compress the input image into a lower dimension (latent space) **while retaining its key features**.

And, if the latent space is 2 dimensional, we can then easily depict it on a 2d plot.

LFG.

## Interactive NFT Exploration

Once the neural network has been trained, we can process each NFT and obtain its latent space. Only thing left is to create a frontend that allows anyone to easily interact with the processed data.

I am a big fan of reusing pre-existing tools (aka my frontend skills are terrible), and lucky for me I found an open sourced tool that does exactly that -- [PixPlot](https://github.com/YaleDHLab/pix-plot)

The first NFT project that I'll be processing will be BAYC because why not. If you'd like to play around with it in fullscreen, check out [ExJpeg](https://exjpeg.com).

<iframe width="100%" height="500" src="https://exjpeg.com/bayc" frameborder="40"></iframe>

I know, the model currently sucks, I'll be updating it from time to time, but its a good enough for a proof of concept for now.

But good to see promising results on the first iteration -- monkeys with similar features are already clustering together.

## Closing Remarks

This is just a fun tool to explore the intersection between art and AI, there is no token.

FYI [ExJpeg](https://exjpeg.com) stands for "Explore Jpegs".

## Links

- [ExJpeg](https://exjpeg.com)
- [ExJpeg Source Code](https://github.com/libevm/exjpeg)
- [CVAE](https://www.tensorflow.org/tutorials/generative/cvae)