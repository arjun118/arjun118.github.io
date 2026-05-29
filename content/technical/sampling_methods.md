+++
date = '2026-05-29T21:57:46+05:30'
draft = false
title = 'Sampling Methods'
tags=['technical']
+++

Let's look at some sampling methods used to control the randomness of an LLM's text generation.
# Background

LLMs by nature are auto-regressive. Meaning the future values (tokens) are predicted by learning from its own past values 

> next step in the sequence depends directly on the steps that came before it

> each new word is generated based on the context of  all the words that were generated previously

You can control this exact behaviour, how randomly you want your LLM to generate the next token

Lets draw an analogy. Lets say you and I are speaking and it's my turn to talk and i start talking. to simplify my thinking process, lets consider that the next word that comes out of my mouth depends on all the previous words that came out of my mouth at the start of my turn.

now i would pay attention to all the previous words i uttered and speak the next word.

If I am not in the mood to talk to you , i will make small talk to end my sentences quickly and you could predict my response easily 

If i enjoy talking to you, i pull up all my creativity to make my sentence more interesting.

Now remember, any word that comes out of my mouth is just from the dictionary of bag of words that i know until that point of time,also called my `vocabulary`

When i am speaking my next word i can say , 

> i am sampling my next word from my vocabulary basis all my previous words spoken until that point

Now the same way, LLMs also have their `vocabulary`: essentially a bunch of words. If you prompt them, they will also `sample` their next word from their `vocabulary` (obviously, unless your tokenizer is broken)

the way you speak depends upon your mood, the same way the  `LLM responds depends upon a bunch of numbers` which you can change anytime you want

Lets look at some of those sampling methods. Below is a list of methods we are going to discuss

1. Greedy
2. Temperature Scaling
3. Top-k
4. Top-P
5. min-P

# Setup

For some hands-on we are gonna use : refer to [awesome-chatper-4](https://github.com/rasbt/LLMs-from-scratch/blob/main/ch04/01_main-chapter-code/ch04.ipynb) and [awesome-chapter-5](https://github.com/rasbt/LLMs-from-scratch/tree/main/ch05/01_main-chapter-code) and [weight-loading-from-pytorch-state-dicts](https://github.com/rasbt/LLMs-from-scratch/blob/main/ch05/02_alternative_weight_loading/weight-loading-pytorch.ipynb)

Below is the whole setup

```python
import torch
import torch.nn as nn
import tiktoken
import numpy as np


class LayerNorm(nn.Module):
    def __init__(self, emb_dim):
        super().__init__()
        self.eps = 1e-5
        self.scale = nn.Parameter(torch.ones(emb_dim))
        self.shift = nn.Parameter(torch.zeros(emb_dim))

    def forward(self, x):
        mean = x.mean(dim=-1, keepdim=True)
        var = x.var(dim=-1, keepdim=True, unbiased=False)
        norm_x = (x - mean) / torch.sqrt(var + self.eps)
        return self.scale * norm_x + self.shift

class GELU(nn.Module):
    def forward(self, x):
        return 0.5 * x * (
            1 + torch.tanh(
                torch.sqrt(torch.tensor(2.0 / torch.pi)) *
                (x + 0.044715 * torch.pow(x, 3))
            )
        )


class FeedForward(nn.Module):
    def __init__(self, cfg):
        super().__init__()

        self.layers = nn.Sequential(
            nn.Linear(cfg["emb_dim"], 4 * cfg["emb_dim"]),
            GELU(),
            nn.Linear(4 * cfg["emb_dim"], cfg["emb_dim"]),
        )

    def forward(self, x):
        return self.layers(x)


class MultiHeadAttention(nn.Module):
    def __init__(
        self,
        d_in,
        d_out,
        context_length,
        dropout,
        num_heads,
        qkv_bias=False,
    ):
        super().__init__()

        assert d_out % num_heads == 0

        self.d_out = d_out
        self.num_heads = num_heads
        self.head_dim = d_out // num_heads

        self.W_query = nn.Linear(d_in, d_out, bias=qkv_bias)
        self.W_key = nn.Linear(d_in, d_out, bias=qkv_bias)
        self.W_value = nn.Linear(d_in, d_out, bias=qkv_bias)

        self.out_proj = nn.Linear(d_out, d_out)
        self.dropout = nn.Dropout(dropout)

        self.register_buffer(
            "mask",
            torch.triu(
                torch.ones(context_length, context_length),
                diagonal=1
            )
        )

    def forward(self, x,attention_mask=None,past_k=None,past_v=None,use_cache=False):
        b, num_tokens, d_in = x.shape

        keys = self.W_key(x)
        queries = self.W_query(x)
        values = self.W_value(x)

        keys = keys.view(
            b, num_tokens, self.num_heads, self.head_dim
        )
        values = values.view(
            b, num_tokens, self.num_heads, self.head_dim
        )
        queries = queries.view(
            b, num_tokens, self.num_heads, self.head_dim
        )

        keys = keys.transpose(1, 2)
        queries = queries.transpose(1, 2)
        values = values.transpose(1, 2)

        attn_scores = queries @ keys.transpose(2, 3)

        mask_bool = self.mask.bool()[:num_tokens, :num_tokens]
        # (batch, n_heads, seq_len,seq_len)
        attn_scores.masked_fill_(mask_bool, -torch.inf)

        if attention_mask is not None:
            # shape - (batch,seq_len)
            # expand to (batch,1,1,seq_len)
            padding_mask=(attention_mask.unsqueeze(1).unsqueeze(2))
            attn_scores=attn_scores.masked_fill(
                padding_mask==0,
                -torch.inf
            )
            
        attn_weights = torch.softmax(
            attn_scores / keys.shape[-1]**0.5,
            dim=-1
        )

        attn_weights = self.dropout(attn_weights)

        context_vec = (attn_weights @ values).transpose(1, 2)

        context_vec = context_vec.contiguous().view(
            b, num_tokens, self.d_out
        )

        context_vec = self.out_proj(context_vec)

        return context_vec


class TransformerBlock(nn.Module):
    def __init__(self, cfg):
        super().__init__()

        self.att = MultiHeadAttention(
            d_in=cfg["emb_dim"],
            d_out=cfg["emb_dim"],
            context_length=cfg["context_length"],
            num_heads=cfg["n_heads"],
            dropout=cfg["drop_rate"],
            qkv_bias=cfg["qkv_bias"],
        )

        self.ff = FeedForward(cfg)

        self.norm1 = LayerNorm(cfg["emb_dim"])
        self.norm2 = LayerNorm(cfg["emb_dim"])

        self.drop_shortcut = nn.Dropout(cfg["drop_rate"])

    def forward(self, x,attention_mask=None):

        shortcut = x
        x = self.norm1(x)
        x = self.att(x,attention_mask=attention_mask)
        x = self.drop_shortcut(x)
        x = x + shortcut

        shortcut = x
        x = self.norm2(x)
        x = self.ff(x)
        x = self.drop_shortcut(x)
        x = x + shortcut

        return x

class GPTModel(nn.Module):
    def __init__(self, cfg):
        super().__init__()

        self.tok_emb = nn.Embedding(
            cfg["vocab_size"],
            cfg["emb_dim"]
        )

        self.pos_emb = nn.Embedding(
            cfg["context_length"],
            cfg["emb_dim"]
        )

        self.drop_emb = nn.Dropout(cfg["drop_rate"])

        self.trf_blocks = nn.Sequential(
            *[TransformerBlock(cfg)
              for _ in range(cfg["n_layers"])]
        )

        self.final_norm = LayerNorm(cfg["emb_dim"])

        self.out_head = nn.Linear(
            cfg["emb_dim"],
            cfg["vocab_size"],
            bias=False
        )

    def forward(self, in_idx,attention_mask=None):
        batch_size, seq_len = in_idx.shape

        tok_embeds = self.tok_emb(in_idx)

        pos_embeds = self.pos_emb(
            torch.arange(seq_len, device=in_idx.device)
        )

        x = tok_embeds + pos_embeds

        x = self.drop_emb(x)

        for block in self.trf_blocks:
            x=block(x,attention_mask=attention_mask)
        x = self.final_norm(x)

        logits = self.out_head(x)

        return logits

# GPT2 config

GPT2_SMALL_124M = {
    "vocab_size": 50257,
    "context_length": 1024,
    "emb_dim": 768,
    "n_heads": 12,
    "n_layers": 12,
    "drop_rate": 0.0,
    "qkv_bias": True,
}

GPT2_MEDIUM_355M = {
    "vocab_size": 50257,
    "context_length": 1024,
    "emb_dim": 1024,
    "n_heads": 16,
    "n_layers": 24,
    "drop_rate": 0.0,
    "qkv_bias": True,
}

GPT2_LARGE_774M = {
    "vocab_size": 50257,
    "context_length": 1024,
    "emb_dim": 1280,
    "n_heads": 20,
    "n_layers": 36,
    "drop_rate": 0.0,
    "qkv_bias": True,
}

GPT2_XL_1558M = {
    "vocab_size": 50257,
    "context_length": 1024,
    "emb_dim": 1600,
    "n_heads": 25,
    "n_layers": 48,
    "drop_rate": 0.0,
    "qkv_bias": True,
}

BASE_CONFIG = {
    "vocab_size": 50257,    # Vocabulary size
    "context_length": 1024, # Context length
    "drop_rate": 0.0,       # Dropout rate
    "qkv_bias": True        # Query-key-value bias
}

model_configs = {
    "gpt2-small (124M)": {"emb_dim": 768, "n_layers": 12, "n_heads": 12},
    "gpt2-medium (355M)": {"emb_dim": 1024, "n_layers": 24, "n_heads": 16},
    "gpt2-large (774M)": {"emb_dim": 1280, "n_layers": 36, "n_heads": 20},
    "gpt2-xl (1558M)": {"emb_dim": 1600, "n_layers": 48, "n_heads": 25},
}


CHOOSE_MODEL = "gpt2-small (124M)"
BASE_CONFIG.update(model_configs[CHOOSE_MODEL])

file_name = "gpt2-small-124M.pth"



import os
import requests

url = f"https://huggingface.co/rasbt/gpt2-from-scratch-pytorch/resolve/main/{file_name}"

if not os.path.exists(file_name):
    response = requests.get(url, timeout=60)
    response.raise_for_status()
    with open(file_name, "wb") as f:
        f.write(response.content)
    print(f"Downloaded to {file_name}")

# load weights

gpt = GPTModel(BASE_CONFIG)
gpt.load_state_dict(torch.load(file_name, weights_only=True))
gpt.eval()

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
gpt.to(device);

gpt

import tiktoken

torch.manual_seed(123)

tokenizer = tiktoken.get_encoding("gpt2")


import torch

PAD_TOKEN_ID=0

def texts_to_token_ids(texts, tokenizer):

    encoded = [
        tokenizer.encode(text)
        for text in texts
    ]
    max_len = max(len(seq) for seq in encoded)

    padded=[]
    attention_masks=[]

    for seq in encoded:
        pad_len=max_len-len(seq)
        padded_seq=[PAD_TOKEN_ID]*pad_len+seq
        mask=[0]*pad_len+[1]*len(seq)
        padded.append(padded_seq)
        attention_masks.append(mask)
    return (
        torch.tensor(padded),
        torch.tensor(attention_masks)
    )


def token_ids_to_texts(token_ids, tokenizer):

    return [
        tokenizer.decode(seq.tolist())
        for seq in token_ids
    ]
```

i have made some changes to token to texts and  text to tokens to add padding and all, but that is still wip

you can just use those from chapter-4 and chapter-5 from [Build a Large Language Model From Scratch book by Sebastian Raschka](https://github.com/rasbt/LLMs-from-scratch)

# Sampling Methods

## Greedy

this is the simplest, most deterministic approach (boringggg)

here ,there is less of sampling happening and more of `choose the most likely answer`

### Working
1. model scans the probability distribution and strictly selects the token with the highest probability

### Features
1. Sounds robotically logical, highly predictable
2. fast, computationally cheap,and excellent for tasks requiring strict factual accuracy

### Code

```python

def generate(model, idx,attention_mask,max_new_tokens, context_size, eos_id=None):
    batch_size=idx.shape[0]
    finished=torch.zeros(
        batch_size,
        dtype=torch.bool,
        device=idx.device
    )
    for _ in range(max_new_tokens):
        active_mask=~finished
        active_indices=torch.nonzero(active_mask,as_tuple=False).squeeze(-1)
        if len(active_indices)==0:
            break
        active_idx=idx[active_indices]
        active_attention_mask=attention_mask[active_indices]
        idx_cond=active_idx[:,-context_size:]
        attn_cond=active_attention_mask[:, -context_size:]
        with torch.no_grad():
            logits=model(idx_cond,attention_mask=attn_cond)
        # only select the last token - newly generated token
        logits=logits[:,-1,:]
        idx_next=torch.argmax(logits,dim=-1,keepdim=True)
        new_tokens=torch.full(
            (batch_size,1),
            eos_id if eos_id is not None else 0,
            dtype=idx.dtype,
            device=idx.device
        )
        new_tokens[active_indices]=idx_next
        idx=torch.cat((idx,new_tokens),dim=1)

        new_attention=torch.zeros(
            (batch_size,1),
            dtype=attention_mask.dtype,
            device=attention_mask.device
        )
        new_attention[active_indices]=1
        attention_mask=torch.cat((attention_mask,new_attention),dim=1)
        if eos_id is not None:
            newly_finished = (
                idx_next.squeeze(-1) == eos_id
            )
            finished[active_indices]|=newly_finished
        if finished.all():
            break
    return idx
```

pay attention to this line: 

> idx_next=torch.argmax(logits,dim=-1,keepdim=True)

that is the whole `sampling` in the greedy method

i have made some changes to include early stopping and including attention masks, different from what that is given in sebastian's github. but yeah you can ignore them, not needed for now

let's generate some text, the greedy way

```python

input_ids, attention_mask = texts_to_token_ids(
    [
        "Every effort moves",
        "practice makes man"
    ],
    tokenizer
)

input_ids = input_ids.to(device)
attention_mask = attention_mask.to(device)

token_ids = generate(
    model=gpt,
    idx=input_ids,
    attention_mask=attention_mask,
    max_new_tokens=30,
    context_size=BASE_CONFIG["context_length"],
    eos_id=tokenizer.eot_token
)

print("generated output: \n")
for i,out in enumerate(token_ids_to_texts(token_ids,tokenizer)):
    print(f"""{i+1}. \n{out}\n""")
```

```
    generated output: 
    
    1. 
    Every effort moves forward, but it's not enough.
    
    "I'm not going to sit here and say, 'I'm not going to do this,'
    
    2. 
    practice makes man's life easier.
    
    The first step is to understand the concept of "self-awareness."
    
    Self-awareness is the ability to recognize
```

## Temperature Scaling

1. this does not actually select the token, as greedy does
2. it changes the shape of the probability distribution before we select the token
3. `parameter: T` will be used in our softmax equation

### Working
1. if T=1.0 -> its the default state - probabilities remain unchanged
2. T<1.0 -> makes distribution sharper - amplifies the differences between the logits
  - rich gets richer, and top choice dominates
  - approaches greedy as T nears 0
3. T>1.0 - flattens the distribution, the math minimizes the differences boosting the probabilites of obscure , low-ranked tokens

### Features
1. this is bascially the model's `creativity` or `hallucination` dial
2. gives granual control over the predictability of the output
3. High temperatures (e.g., T > 1.2) will often cause the model to output complete gibberish by assigning too much probability to nonsensical tokens.
4. Standard sampling with temperature still leaves a tiny probability that the model might pick the absolute worst token in the vocabulary

### Code
```python

def generate(model, idx, attention_mask,max_new_tokens, context_size,temperature=0.0, eos_id=None):
    batch_size=idx.shape[0]
    finished=torch.zeros(
        batch_size,
        dtype=torch.bool,
        device=idx.device
    )
    for _ in range(max_new_tokens):
        active_mask=~finished
        active_indices=torch.nonzero(active_mask,as_tuple=False).squeeze(-1)
        if len(active_indices)==0:
            break
        active_idx=idx[active_indices]
        active_attention_mask=attention_mask[active_indices]
        idx_cond=active_idx[:,-context_size:]
        attn_cond=active_attention_mask[:, -context_size:]
        with torch.no_grad():
            logits=model(idx_cond,attention_mask=attn_cond)
        # only select the last token - newly generated token
        logits=logits[:,-1,:]
    ###### code block change ######
        if temperature>0.0:
            logits=logits/temperature
            logits=logits-logits.max(dim=-1,keepdim=True).values
            probs=torch.softmax(logits,dim=-1)
            idx_next=torch.multinomial(probs,num_samples=1)
        else:
            # it temp is 0. its same as greedy 
            idx_next=torch.argmax(logits,dim=-1,keepdim=True)
    ###### code block change ######
        new_tokens=torch.full(
            (batch_size,1),
            eos_id if eos_id is not None else 0,
            dtype=idx.dtype,
            device=idx.device
        )
        new_tokens[active_indices]=idx_next
        idx=torch.cat((idx,new_tokens),dim=1)

        new_attention=torch.zeros(
            (batch_size,1),
            dtype=attention_mask.dtype,
            device=attention_mask.device
        )
        new_attention[active_indices]=1
        attention_mask=torch.cat((attention_mask,new_attention),dim=1)
        if eos_id is not None:
            newly_finished = (
                idx_next.squeeze(-1) == eos_id
            )
            finished[active_indices]|=newly_finished
        if finished.all():
            break
    return idx
```
focus on 
```python
if temperature>0.0:
    logits=logits/temperature
    logits=logits-logits.max(dim=-1,keepdim=True).values
    probs=torch.softmax(logits,dim=-1)
    idx_next=torch.multinomial(probs,num_samples=1)
```
we scale the logits and substract the max (for softmax stability) and calculate the probabilities by applying softmax

usage:

```python
input_ids, attention_mask = texts_to_token_ids(
    [
        "Every effort moves",
        "practice makes man"
    ],
    tokenizer
)

input_ids = input_ids.to(device)
attention_mask = attention_mask.to(device)

token_ids = generate(
    model=gpt,
    idx=input_ids,
    attention_mask=attention_mask,
    max_new_tokens=30,
    context_size=BASE_CONFIG["context_length"],
    temperature=0.5,
    eos_id=tokenizer.eot_token
)
print("generated output: \n")
for i,out in enumerate(token_ids_to_texts(token_ids,tokenizer)):
    print(f"""{i+1}. \n{out}\n""")
```
```
  generated output: 
  
  1. 
  Every effort moves forward, but a little time needs to be made to get it done.
  
  The project is currently in its early stages, and has been a
  
  2. 
  practice makes man-made disasters much more likely.
  
  But if you want to know how to prevent a disaster, you need to know how to do it right
```

## Top-K Sampling
1. you set a number `K`
2. the model sorts its output (basically a probability for each of the token) probabilities and throws away everything from `K+1` rank downwards
3. redistributes the remaining probability mass among the top K tokens. It then samples from that truncated list.

### Features
1. controlled randomness
2. it eliminates the long tail of garbage tokens by preventing model from suddenly going completely off the rails - it does so by being very rigid with the prob cutoff
3. if the model is highly confident  (one token as 90% prob), the top-k still forces the model to consider the next K-1 terrible options
4. if the model is deeply uncertain , low K might prematurely cutoff perfectly valid words

### Code

```python

def generate(model, idx, attention_mask,max_new_tokens, context_size,top_k=None,temperature=0.0, eos_id=None):
    batch_size=idx.shape[0]
    finished=torch.zeros(
        batch_size,
        dtype=torch.bool,
        device=idx.device
    )
    for _ in range(max_new_tokens):
        active_mask=~finished
        active_indices=torch.nonzero(active_mask,as_tuple=False).squeeze(-1)
        if active_indices.numel()==0:
            break
        active_idx=idx[active_indices]
        active_attention_mask=attention_mask[active_indices]
        idx_cond=active_idx[:,-context_size:]
        attn_cond=active_attention_mask[:, -context_size:]
        with torch.no_grad():
            logits=model(idx_cond,attention_mask=attn_cond)
        # only select the last token - newly generated token
        logits=logits[:,-1,:]
        ###### code block change ######
        if temperature>0.0:
            logits=logits/temperature
        if top_k is not None:
            top_logits,_=torch.topk(logits,top_k)
            min_val=top_logits[:,-1].unsqueeze(-1)
            logits=torch.where(logits<min_val,torch.tensor(float("-inf")).to(logits.device),logits)
        ###### code block change ######
        if temperature>0.0:
            logits=logits-logits.max(dim=-1,keepdim=True).values
            probs=torch.softmax(logits,dim=-1)
            idx_next=torch.multinomial(probs,num_samples=1)
        else:
            # it temp is 0. its same as greedy 
            idx_next=torch.argmax(logits,dim=-1,keepdim=True)
        new_tokens=torch.full(
            (batch_size,1),
            eos_id if eos_id is not None else 0,
            dtype=idx.dtype,
            device=idx.device
        )
        new_tokens[active_indices]=idx_next
        idx=torch.cat((idx,new_tokens),dim=1)

        new_attention=torch.zeros(
            (batch_size,1),
            dtype=attention_mask.dtype,
            device=attention_mask.device
        )
        new_attention[active_indices]=1
        attention_mask=torch.cat((attention_mask,new_attention),dim=1)
        if eos_id is not None:
            newly_finished = (
                idx_next.squeeze(-1) == eos_id
            )
            finished[active_indices]|=newly_finished
        if finished.all():
            break
    return idx
```
usage:

```python
input_ids, attention_mask = texts_to_token_ids(
    [
        "Every effort moves",
        "practice makes man"
    ],
    tokenizer
)

input_ids = input_ids.to(device)
attention_mask = attention_mask.to(device)

token_ids = generate(
    model=gpt,
    idx=input_ids,
    attention_mask=attention_mask,
    max_new_tokens=30,
    context_size=BASE_CONFIG["context_length"],
    top_k=40,
    temperature=0.5,
    eos_id=tokenizer.eot_token
)
print("generated output: \n")
for i,out in enumerate(token_ids_to_texts(token_ids,tokenizer)):
    print(f"""{i+1}. \n{out}\n""")
```
```
    generated output: 
    
    1. 
    Every effort moves forward. I'm hopeful that we will be able to do this," said one of the people.
    
    The group plans to hold a meeting at
    
    2. 
    practice makes man a better leader.
    
    The problem is, there is no such thing as a perfect leader.
    
    You have to start somewhere. You have
```
## Top-P (Nucleus Sampling)
1. this fixes the rigidity of the Top-K by using dynamic cutoff based on probability mass, rather than fixed number of tokens

### Working
1. you set a percentage `top_p:0.9`, i.e. 90%
2. model sorts the tokens by probability and keeps adding them to a pool until the cumulative sum of their probs hits 90%
  - if model is highly confident: it will take less tokens to reach 90%
  - if the model is uncertain: it will take more tokens to reach 90%

### Features
1. this is kind of a `context-aware` filtering rather a rigid number of tokens
2. superior and produces more natural sounding, diverse text

### Code

```python

def generate(model, idx, attention_mask,max_new_tokens, context_size,top_p=None,top_k=None,temperature=0.0, eos_id=None):
    batch_size=idx.shape[0]
    finished=torch.zeros(
        batch_size,
        dtype=torch.bool,
        device=idx.device
    )
    for _ in range(max_new_tokens):
        active_mask=~finished
        active_indices=torch.nonzero(active_mask,as_tuple=False).squeeze(-1)
        if active_indices.numel()==0:
            break
        active_idx=idx[active_indices]
        active_attention_mask=attention_mask[active_indices]
        idx_cond=active_idx[:,-context_size:]
        attn_cond=active_attention_mask[:, -context_size:]
        with torch.no_grad():
            logits=model(idx_cond,attention_mask=attn_cond)
        # only select the last token - newly generated token
        logits=logits[:,-1,:]
        if temperature>0.0:
            logits=logits/temperature
        if top_k is not None:
            # only cares about ranking not probs
            top_logits,_=torch.topk(logits,top_k)
            min_val=top_logits[:,-1].unsqueeze(-1)
            logits=torch.where(logits<min_val,torch.tensor(float("-inf")).to(logits.device),logits)
         ###### code block change ######
        if top_p is not None:
            # renormalize the remaining tokens probs 
            probs=torch.softmax(logits,dim=-1)
            sorted_probs,sorted_indices=torch.sort(
                probs,
                descending=True,
                dim=-1
            )
            cumulative_probs=torch.cumsum(sorted_probs,dim=-1)
            sorted_indices_to_remove=cumulative_probs>top_p
            # shift by one - include the token that actually crossed
            # the threshold
            sorted_indices_to_remove[..., 1:]=sorted_indices_to_remove[..., :-1].clone()
            sorted_indices_to_remove[..., 0] = False
            indices_to_remove=torch.zeros_like(logits,dtype=torch.bool)
            indices_to_remove.scatter_(
                dim=-1,
                index=sorted_indices,
                src=sorted_indices_to_remove
            )
            logits = logits.masked_fill(
                indices_to_remove,
                float("-inf")
            )
         ###### code block change ######
        if temperature>0.0:
            logits=logits-logits.max(dim=-1,keepdim=True).values
            probs=torch.softmax(logits,dim=-1)
            idx_next=torch.multinomial(probs,num_samples=1)
        else:
            # it temp is 0. its same as greedy 
            idx_next=torch.argmax(logits,dim=-1,keepdim=True)
        new_tokens=torch.full(
            (batch_size,1),
            eos_id if eos_id is not None else 0,
            dtype=idx.dtype,
            device=idx.device
        )
        new_tokens[active_indices]=idx_next
        idx=torch.cat((idx,new_tokens),dim=1)

        new_attention=torch.zeros(
            (batch_size,1),
            dtype=attention_mask.dtype,
            device=attention_mask.device
        )
        new_attention[active_indices]=1
        attention_mask=torch.cat((attention_mask,new_attention),dim=1)
        if eos_id is not None:
            newly_finished = (
                idx_next.squeeze(-1) == eos_id
            )
            finished[active_indices]|=newly_finished
        if finished.all():
            break
    return idx
```
usage:
```python
input_ids, attention_mask = texts_to_token_ids(
    [
        "Every effort moves",
        "practice makes man"
    ],
    tokenizer
)

input_ids = input_ids.to(device)
attention_mask = attention_mask.to(device)

token_ids = generate(
    model=gpt,
    idx=input_ids,
    attention_mask=attention_mask,
    max_new_tokens=30,
    context_size=BASE_CONFIG["context_length"],
    # top_k=40,
    top_p=0.8,
    temperature=0.3,
    eos_id=tokenizer.eot_token
)
print("generated output: \n")
for i,out in enumerate(token_ids_to_texts(token_ids,tokenizer)):
    print(f"""{i+1}. \n{out}\n""")
```

```
generated output: 

1. 
Every effort moves forward, but the next step is to try to get the best of the best.

"We have to make sure that we're not going

2. 
practice makes man's life more difficult, but it's also a good thing.

The best way to learn to be a better man is to be a better
```

## min-p sampling
1. scales the cutoff relative to the top choice
2. effectively does the job of temperature and top-p combined without needing constant tuning

### Working
1. you set a `min_p:0.1`
2. The model looks at the probability of the absolute top token. The cutoff for all other tokens becomes min_p multiplied by the top token's probability.
  - If the top token has an 80% chance, any other token must have at least an 8% chance to be considered.
  - If the top token only has a 20% chance, other tokens only need a 2% chance to be considered

### Features
1. balances the strictness and creativity. aggressively prunes bad options when the model is confident

```python

def generate(model, idx, attention_mask,max_new_tokens, context_size,min_p=None,top_p=None,top_k=None,temperature=0.0, eos_id=None):
    batch_size=idx.shape[0]
    finished=torch.zeros(
        batch_size,
        dtype=torch.bool,
        device=idx.device
    )
    for _ in range(max_new_tokens):
        active_mask=~finished
        active_indices=torch.nonzero(active_mask,as_tuple=False).squeeze(-1)
        if active_indices.numel()==0:
            break
        active_idx=idx[active_indices]
        active_attention_mask=attention_mask[active_indices]
        idx_cond=active_idx[:,-context_size:]
        attn_cond=active_attention_mask[:, -context_size:]
        with torch.no_grad():
            logits=model(idx_cond,attention_mask=attn_cond)
        # only select the last token - newly generated token
        # batchsize, 1, logits
        logits=logits[:,-1,:]
        if temperature>0.0:
            logits=logits/temperature
        if top_k is not None:
            top_logits,_=torch.topk(logits,top_k)
            min_val=top_logits[:,-1].unsqueeze(-1)
            logits=torch.where(logits<min_val,torch.tensor(float("-inf")).to(logits.device),logits)
        if top_p is not None:
            # renormalize the remaining tokens probs 
            probs=torch.softmax(logits,dim=-1)
            sorted_probs,sorted_indices=torch.sort(
                probs,
                descending=True,
                dim=-1
            )
            cumulative_probs=torch.cumsum(sorted_probs,dim=-1)
            sorted_indices_to_remove=cumulative_probs>top_p
            # shift by one - include the token that actually crossed
            # the threshold
            sorted_indices_to_remove[..., 1:]=sorted_indices_to_remove[..., :-1].clone()
            sorted_indices_to_remove[..., 0] = False
            indices_to_remove=torch.zeros_like(logits,dtype=torch.bool)
            indices_to_remove.scatter_(
                dim=-1,
                index=sorted_indices,
                src=sorted_indices_to_remove
            )
            logits = logits.masked_fill(
                indices_to_remove,
                float("-inf")
            )
         ###### code block change ######
        if min_p is not None:
            # (batchsize,1,1)
            probs=torch.softmax(logits,dim=-1)
            max_prob_values,_=torch.max(probs, dim=-1,keepdim=True)
            threshold=max_prob_values*min_p
            indices_to_remove= probs < threshold
            logits=logits.masked_fill(
                indices_to_remove,
                float("-inf")
            )
         ###### code block change ######
        # token selection
        if temperature>0.0:
            logits=logits-logits.max(dim=-1,keepdim=True).values
            probs=torch.softmax(logits,dim=-1)
            idx_next=torch.multinomial(probs,num_samples=1)
        else:
            # it temp is 0. its same as greedy 
            idx_next=torch.argmax(logits,dim=-1,keepdim=True)
        
        new_tokens=torch.full(
            (batch_size,1),
            eos_id if eos_id is not None else 0,
            dtype=idx.dtype,
            device=idx.device
        )
        new_tokens[active_indices]=idx_next
        idx=torch.cat((idx,new_tokens),dim=1)

        new_attention=torch.zeros(
            (batch_size,1),
            dtype=attention_mask.dtype,
            device=attention_mask.device
        )
        new_attention[active_indices]=1
        attention_mask=torch.cat((attention_mask,new_attention),dim=1)
        if eos_id is not None:
            newly_finished = (
                idx_next.squeeze(-1) == eos_id
            )
            finished[active_indices]|=newly_finished
        if finished.all():
            break
    return idx
```
usage
```python
input_ids, attention_mask = texts_to_token_ids(
    [
        "Every effort moves",
        "practice makes man"
    ],
    tokenizer
)

input_ids = input_ids.to(device)
attention_mask = attention_mask.to(device)

token_ids = generate(
    model=gpt,
    idx=input_ids,
    attention_mask=attention_mask,
    max_new_tokens=30,
    context_size=BASE_CONFIG["context_length"],
    min_p=0.1,
    top_p=0.9,
    temperature=0.7,
    eos_id=tokenizer.eot_token
)
print("generated output: \n")
for i,out in enumerate(token_ids_to_texts(token_ids,tokenizer)):
    print(f"""{i+1}. \n{out}\n""")
```
```
    generated output: 
    
    1. 
    Every effort moves forward, and we will continue to build on our strong partnership with the United States," the statement said. "We will continue to work closely with our
    
    2. 
    practice makes man's life easier, and it's the best way to get rid of the stress.
    
    The best way to get rid of stress is to get
```

# Operating Space

> Temperature and Top-k operate on logits

> although top-k can technically operate on probabilities since the rank order doesnot change after softmax but operating on logits is industry standard because it skips calculating exponentials for the grabage tokens you are about to throw away anyway.

> Top-p and min-p operate on probabilities

and 

> Top-p, top-k, min-p won't actually make a difference while greedy decoding, they only make a difference when temperature > 0.0

the general order of applying these sampling methods is as below

```
    LOGIT SPACE
    ↓
    PROBABILITY SPACE
    ↓
    SAMPLING
```

```
      logits
      ↓
      temperature scaling
      ↓
      top_k
      ↓
      top_p
      ↓
      min_p
      ↓
      softmax
      ↓
      multinomial
```

I plan to cover,

1. implementing KV-cache in the above GPTModel implementation
2. IDK whatever comes to mind after that

Sampling Methods
  1. repetition penalty
  2. presence penalty
  3. frequency penalty
  4. logit bias
  5. bad-word masking
  6. forced tokens

Decoding Methods
  1. Beam Search
  2. Speculative Decoding
  3. Contrastive Search/Decoding

in the upcoming blogs.
