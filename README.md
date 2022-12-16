## Classifier Free Guidance - Pytorch (wip)

Implementation of <a href="https://arxiv.org/abs/2207.12598">Classifier Free Guidance</a> in Pytorch, with emphasis on text conditioning, and flexibility to include multiple text embedding models, as done in <a href="https://deepimagination.cc/eDiff-I/">eDiff-I</a>

It is clear now that text guidance is the ultimate interface to models. This repository will leverage some python decorator magic to make it easy to incorporate SOTA text conditioning to any model.

## Appreciation

- <a href="https://stability.ai/">StabilityAI</a> for the generous sponsorship, as well as my other sponsors out there

- <a href="https://huggingface.co/">🤗 Huggingface</a> for their amazing transformers library. The text conditioning module will use T5 embeddings, as latest research recommends

- <a href="https://github.com/mlfoundations/open_clip">OpenCLIP</a> for providing SOTA open sourced CLIP models. The eDiff model sees immense improvements by combining the T5 embeddings with CLIP text embeddings


## Install

```bash
$ pip install classifier-free-guidance-pytorch
```

## Usage

```python
import torch
from classifier_free_guidance_pytorch import TextConditioner

text_conditioner = TextConditioner(
    model_types = 't5',    
    hidden_dims = (256, 512),
    hiddens_channel_first = False,
    cond_drop_prob = 0.2  # conditional dropout 20% of the time, must be greater than 0. to unlock classifier free guidance
).cuda()

# pass in your text as a List[str], and get back a List[callable]
# each callable function receives the hiddens in the dimensions listed at init (hidden_dims)

first_condition_fn, second_condition_fn = text_conditioner(['a dog chasing after a ball'])

# these hiddens will be in the direct flow of your model, say in a unet

first_hidden = torch.randn(1, 16, 256).cuda()
second_hidden = torch.randn(1, 32, 512).cuda()

# conditioned features

first_conditioned = first_condition_fn(first_hidden)
second_conditioned = second_condition_fn(second_hidden)
```

## Magic Decorator (wip)

This is a work in progress to make it as easy as possible to text condition your network.

First, let's say you have a simple two layer network

```python
import torch
from torch import nn

class MLP(nn.Module):
    def __init__(
        self,
        dim
    ):
        super().__init__()
        self.proj_in = nn.Sequential(nn.Linear(dim, dim * 2), nn.ReLU())
        self.proj_mid = nn.Sequential(nn.Linear(dim * 2, dim), nn.ReLU())
        self.proj_out = nn.Linear(dim, 1)

    def forward(
        self,
        data
    ):
        hiddens1 = self.proj_in(data)
        hiddens2 = self.proj_mid(hiddens1)
        return self.proj_out(hiddens2)

# instantiate model and pass in some data, get (in this case) a binary prediction

model = MLP(dim = 256)

data = torch.randn(2, 256)

pred = model(data)
```

You would like to condition the hidden layers (`hiddens1` and `hiddens2`) with text. Each batch element here would get its own free text conditioning

This has been whittled down to ~4 step using this repository. Always open to suggestions

```python
import torch
from torch import nn

from classifier_free_guidance_pytorch import TextConditioner, classifier_free_guidance

class MLP(nn.Module):
    def __init__(
        self,
        dim
    ):
        super().__init__()
        self.proj_in = nn.Sequential(nn.Linear(dim, dim * 2), nn.ReLU())
        self.proj_mid = nn.Sequential(nn.Linear(dim * 2, dim), nn.ReLU())
        self.proj_out = nn.Linear(dim, 1)

        # (1) you must instantiate a text conditioner
        # and pass in the hidden dimensions you would like to condition on. in this case there are two hidden dimensions (dim * 2 and dim, after the first and second projections)
        self.text_conditioner = TextConditioner(hidden_dims = (dim * 2, dim))

    @classifier_free_guidance
    def forward(
        self,
        inp,
        cond_fns,               # List[Callable] - (2) your forward function now receives a list of conditioning functions, which you invoke on your hidden tensors
        cond_drop_prob = 0.2    # (3) this must be [optionally] set for classifier free guidance (Ho et al.), will randomly drop out the text conditioning automatically at training.
    ):
        cond_fn1, cond_fn2 = cond_fns # conditioning functions are given back in the order of the `hidden_dims` set on the text conditioner

        hiddens1 = self.proj_in(inp)
        hiddens1 = cond_fn1(hiddens1) # (3) condition the first hidden layer with FiLM

        hiddens2 = self.proj_mid(hiddens1)
        hiddens2 = cond_fn2(hiddens2) # condition the second hidden layer with FiLM

        return self.proj_out(hiddens2)

# instantiate your model

model = MLP(dim = 256)

# now you have your input data as well as corresponding free text as List[str]

data = torch.randn(2, 256)
texts = ['a description', 'another description']

# (4) train your model, passing in your list of strings as 'texts'

pred  = model(data, texts = texts)

# after much training, you can now do classifier free guidance by passing in a condition scale of > 1. !

model.eval()
guided_pred = model(data, texts = texts, cond_scale = 3.)  # cond_scale stands for conditioning scale from classifier free guidance paper
```

## Todo

- [x] complete film conditioning, without classifier free guidance (used <a href="https://github.com/lucidrains/robotic-transformer-pytorch/blob/main/robotic_transformer_pytorch/robotic_transformer_pytorch.py">here</a>)
- [x] add classifier free guidance for film conditioning

- [ ] complete cross attention conditioning
- [ ] complete auto perceiver resampler
- [ ] stress test for spacetime unet in <a href="https://github.com/lucidrains/make-a-video-pytorch">make-a-video</a>

## Citations

```bibtex
@article{Ho2022ClassifierFreeDG,
    title   = {Classifier-Free Diffusion Guidance},
    author  = {Jonathan Ho},
    journal = {ArXiv},
    year    = {2022},
    volume  = {abs/2207.12598}
}
```

```bibtex
@article{Balaji2022eDiffITD,
    title   = {eDiff-I: Text-to-Image Diffusion Models with an Ensemble of Expert Denoisers},
    author  = {Yogesh Balaji and Seungjun Nah and Xun Huang and Arash Vahdat and Jiaming Song and Karsten Kreis and Miika Aittala and Timo Aila and Samuli Laine and Bryan Catanzaro and Tero Karras and Ming-Yu Liu},
    journal = {ArXiv},
    year    = {2022},
    volume  = {abs/2211.01324}
}
```
