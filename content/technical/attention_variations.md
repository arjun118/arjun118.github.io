+++
date = '2026-05-08T20:46:06+05:30'
draft = false
title = 'Self-Attention Variations'
tags=['technical','LLM']
+++

As the title suggests, we will look at some self-attention mechanisms in this blog. These are by no means all the known attention mechanisms.
I’m writing these based on my understanding while trying to keep the implementations accurate.

# Causal Scaled dot product attention

![attention_formula](/images/attention_formula.png)

This is the original (“vanilla”) attention mechanism proposed in the Attention Is All You Need paper.

we will implement Causal (future tokens masked) scaled dot product self-attention

lets get our dummy inputs right

## Dummy input setup

```python
import torch
import torch.nn as nn
inputs=torch.rand(2048,512)

batch=torch.stack((inputs,inputs),dim=0)

print(batch.shape)
# torch.Size([2, 2048, 512])
```

2 is our batch size and each batch has 2048 tokens and each token has an embedding of dimension of 512 (of course all random, nothing real)

lets also define our config dictionary, just to keep things organized

```python
config={
    "context_length":2048*2,
    "dropout":0.1,

}
```

we initialize the maximum supported context length to 4096, although our current input sequence length is 2048 and dropout factor is 0.1 (to be used in our self-attention block)

> All inputs, batch,config will stay same for all the variants we are gonna learn

below are some other variables we initialize so that we don`t keep repeating ourselves

```python
d_in=inputs.shape[1]
d_out=512 #input dimension to output dimension -> generally preferred to preserve input dimensionality
```

first lets see the code, i've also included comments,shapes beside the code lines but we will also see some explanation below

## Code

```python
class CausalSelfAttention(nn.Module):
    def __init__(self, d_in,d_out,context_length, dropout,qkv_bias=False):
        super().__init__()
        self.W_query= nn.Linear(d_in,d_out,bias=qkv_bias)
        self.W_key= nn.Linear(d_in,d_out,bias=qkv_bias)
        self.W_value= nn.Linear(d_in,d_out,bias=qkv_bias)
        self.d_out=d_out
        self.dropout=nn.Dropout(dropout)
        self.register_buffer("mask",torch.triu(torch.ones(context_length,context_length),diagonal=1))
        self.scale=d_out**-0.5
    def forward(self,x):
        # batch size, number of tokens, input dimension
        b,num_tokens,d_in=x.shape
        # find out projections using the weight matrices
        queries,keys,values=self.W_query(x),self.W_key(x),self.W_value(x)
        # keys -> b, num_tokens, d_out
        # queries -> b, num_tokens, d_out
        # values -> b,num_tokens, d_out
        attn_scores=queries@keys.transpose(1,2) # this is QK^T part of the attention calculation
        attn_scores.masked_fill_(self.mask.bool()[:num_tokens,:num_tokens],-torch.inf) # apply causal mask and don`t let the current tokens attend to future tokens, only past tokens
        attn_weights=torch.softmax(attn_scores*self.scale,dim=-1) # (1/sqrt(dk)) part of attention calc
        attn_weights=self.dropout(attn_weights)
        context_vec=attn_weights@values
        return context_vec
```

if you are familiar with PyTorch, the class declaration should make sense along with the usage of Linear, Dropout, transpose, view etc. refer to pytorch documentation if it doesn't, they are pretty self explanatory

## Explanation

1. `constructor`:W_query, W_key, W_value are linear projection layers which we will need to calcuate the projections from the input matrix - x
2. dropout is for us to apply some sort of regularization - achieved via randomly dropping attention weights at a factor of 0.1 3.`forward`: in the forward method you determine the batch size, input sequence length, and input dimensionality
3. then you calculate `queries`, `keys`, `values` matrices which are basically projections of the input matrix x calculated using the corresponding weight matrices
4. we calculate attention scores by making the keys matrix multiplication compatible with our queries matrix, refer the shapes from the comments in the code
5. Apply causal mask (mask future tokens) by setting their attention score as negative infinity so that they approximate to zero after softmax.
6. Apply softmax and scale by the inverse square root of the key dimension (head_dim or d_k)
7. scaling factor is used because,
```
1.keep the variance of these attention scores(dot products) stable
2. as dimension increases , attention logits become larger and larger
3.without it, softmax becomes extremely peaky, gradients become tiny, training becomes unstable
```
8. apply dropout (for regularization and good generalization)
9. finally calcuate the context vectors by multiplying attention weights with values

![causal_sdpa](/images/causal_sdpa.png)

check the outputs by running the below code

```python
torch.manual_seed(123)
ca=CausalSelfAttention(d_in,d_out,config['context_length'],config['dropout'])
context_vecs=ca(batch)
print(context_vecs.shape)
# torch.Size([2, 2048, 512])
```

# Multi Head Attention

now lets move onto multi head attention, i am not going to draw any images, just code and explanation.

For visual intuition , please refer to this excellent blog - https://magazine.sebastianraschka.com/p/visual-attention-variants

we will use the same `batch` as before,

## Code

```python
class MultiHeadAttention(nn.Module):
    def __init__(self,d_in,d_out,context_length,dropout,n_heads,qkv_bias=False):
        super().__init__()
        assert d_out % n_heads == 0, "output dimension must be divisible by number of heads"
        self.W_query = nn.Linear(d_in, d_out, bias=qkv_bias)
        self.W_key   = nn.Linear(d_in, d_out, bias=qkv_bias)
        self.W_value = nn.Linear(d_in, d_out, bias=qkv_bias)
        self.d_out=d_out
        self.n_heads=n_heads
        self.head_dim=d_out//self.n_heads
        self.out_proj=nn.Linear(d_out,d_out) # linear layer we use to combine our individual head outputs
        self.dropout=nn.Dropout(dropout)
        self.scale = self.head_dim ** -0.5
        # Non-trainable tensors that are still part of model state.
        self.register_buffer("mask",torch.triu(torch.ones(context_length,context_length),diagonal=1))
    def forward(self,x):
        # batch size, number of input tokens, input dimension
        b,input_tokens,d_in=x.shape
        # calculate query , key ,value projections
        queries,keys,values=self.W_query(x),self.W_key(x),self.W_value(x)
        # each of queries,keys, values has
        # size of b, input_tokens, d_out
        # split queries keys and values into different heads - only the last dimension
        queries=queries.view(b,input_tokens,self.n_heads,self.head_dim)
        values=values.view(b,input_tokens,self.n_heads,self.head_dim)
        keys=keys.view(b,input_tokens,self.n_heads,self.head_dim)
        # now each of queries, values, keys has
        # (batch size, input tokens, number of heads, each head dimension)
        # transpose 1 and 2 dimension of these matrices (0 based indexing)
        queries=queries.transpose(1,2)
        values=values.transpose(1,2)
        keys=keys.transpose(1,2)
        # now each of queries, values, keys has
        # (batch size, n_heads,input_tokens, head_dim)
        attn_scores=queries@keys.transpose(2,3)
        # above multiplication fares out as -> (b, n_heads,input_tokens,head_dim) @ (b,n_heads,head_dim,input_tokens) => (b, n_heads,input_tokens,input_tokens)
        mask_bool=self.mask.bool()[:input_tokens,:input_tokens]
        attn_scores.masked_fill_(mask_bool,-torch.inf)
        attn_weights=torch.softmax(attn_scores*self.scale,dim=-1)
        attn_weights=self.dropout(attn_weights)
        # context vector
        # shape before transpose: batch_size, n_heads,num_tokens, head_dim
        # after transpose: batch_size, num_tokens,n_heads, head_dim

        # attn weights - (b, n_heads,input_tokens,input_tokens)
        # values- (b, n_heads,input_tokens, head_dim)
        # attn_weights @ values => (b, n_heads,input_tokens, head_dim)
        context_vec=(attn_weights @ values).transpose(1,2)
        # shape of context vec => (b, input_tokens, n_heads, head_dim)
        context_vec=context_vec.contiguous().view(b,input_tokens,self.d_out)
        context_vec=self.out_proj(context_vec)
        return context_vec
```

## Explanation

lets consider our d_in = d_out = 256

1. Every single head sees the entire input sequence (all the tokens). They do not look at just a slice of the sentence.
2. conceptually, What the heads split up is the embedding dimension (the features), not the number of tokens (the words). keep in mind that heads are not literally assigned fixed semantic feature ranges, this is just for our understanding.

### The Sequence vs. The Features

Think about the shape of your tensor right before you calculate the attention scores: query shape: (batch_size, n_heads, num_tokens, head_dim). example numbers: (2, 8, 16, 32)

Let's isolate just one sequence in the batch and look at what Head 1 gets:

Head 1's slice: A matrix of 16 (tokens) by 32 (features).

Notice that the 16 is completely intact. Head 1 has access to Token 1, Token 2, Token 3... all the way to Token 16. It sees the whole sentence.
What it doesn't have the full picture of is the d_out dimension. It only gets 32 out of the 256 total features available for those words.

Every head sees every token by looking at the scaled dot-product calculation:

```python
attn_scores = query @ keys.transpose(2,3)
```

> The resulting attn_scores tensor has the shape (batch_size, n_heads, num_tokens, num_tokens). Using your numbers, that is (2, 8, 16, 16).

That 16 x 16 grid exists inside every single one of the 8 heads. That means inside Head 1, Token 1 calculates an attention score with Token 1, Token 2, Token 3, etc. Every token interacts with every other token, inside every single head.

### The "Inspectors" Analogy

Imagine your input sequence is a line of 16 cars.

The Embedding (d_out): A massive, 256-page spec sheet for each car (paint color, engine type, tire pressure, top speed, etc.).

Single-Head Attention: One general inspector looks at all 16 cars and tries to read the entire 256-page spec sheet for every car at the same time to figure out how the cars relate to each other. It's overwhelming, and they average things out.

Multi-Head Attention (8 Heads): You hire 8 highly specialized inspectors.

Inspector 1 (Head 1) looks at all 16 cars, but only cares about the paint and interior colors (features 0-31).

Inspector 2 (Head 2) looks at all 16 cars, but only cares about the engine and transmission (features 32-63).

we eventually combine them using .contiguous().view(...) and self.out_proj.
We do this because while Inspector 1 perfectly understands which cars have matching colors, and Inspector 2 perfectly understands which cars can race each other, neither of them has the full picture of the cars. The final linear projection layer is the "meeting room" where all 8 inspectors bring their 16 x 32 reports, staple them back together into a 16 x 256 report, and synthesize a final understanding of the entire sequence.

usage:

```python
torch.manual_seed(123)
mha=MultiHeadAttention(d_in,d_out,config['context_length'],config['dropout'],8)
context_vecs=mha(batch)
print(context_vecs.shape)
# torch.Size([2, 2048, 512])
```

# Multi Query Attention

multi query attention is very similar to multi head attention but all the query heads share the same key and value projections across all calculations

## Code

```python
class MultiQueryAttention(nn.Module):
    def __init__(self,d_in,d_out,context_length,dropout,n_heads,qkv_bias=False):
        super().__init__()
        assert d_out % n_heads == 0, "output dimension must be divisible by number of heads"
        self.d_out=d_out
        self.n_heads=n_heads
        self.head_dim=d_out//self.n_heads
        self.W_query = nn.Linear(d_in, self.d_out, bias=qkv_bias)
        self.W_key   = nn.Linear(d_in, self.head_dim, bias=qkv_bias)
        self.W_value = nn.Linear(d_in, self.head_dim, bias=qkv_bias)
        self.out_proj=nn.Linear(d_out,d_out) # linear layer we use to combine our individual head outputs
        self.dropout=nn.Dropout(dropout)
        self.scale = self.head_dim ** -0.5
        # Non-trainable tensors that are still part of model state.
        self.register_buffer("mask",torch.triu(torch.ones(context_length,context_length),diagonal=1))
    def forward(self,x):
        # batch size, number of input tokens, input dimension
        b,input_tokens,d_in=x.shape
        # calculate query , key ,value projections
        queries,keys,values=self.W_query(x),self.W_key(x),self.W_value(x)
        # each of queries,keys, values has
        # size of b, input_tokens, d_out
        # split queries keys and values into different heads - only the last dimension
        queries=queries.view(b,input_tokens,self.n_heads,self.head_dim)
        values,keys=values.unsqueeze(1),keys.unsqueeze(1)
        # values=values.view(b,1,input_tokens,self.head_dim)
        # keys=keys.view(b,1,input_tokens,self.head_dim)
        # queries has(b, input tokens,n_heads, head_dim)
        # transpose 1 and 2 dimension only for queries
        queries=queries.transpose(1,2)
        # now  queries has (batch size, n_heads,input_tokens, head_dim)
        # values, keys (batch size, 1,input_tokens, head_dim)
        attn_scores=queries@keys.transpose(2,3)
        # above multiplication fares out as -> (b, n_heads,input_tokens,head_dim) @ (b,1,head_dim,input_tokens) => (b, n_heads,input_tokens,input_tokens)
        mask_bool=self.mask.bool()[:input_tokens,:input_tokens]
        attn_scores.masked_fill_(mask_bool,-torch.inf)
        attn_weights=torch.softmax(attn_scores*self.scale,dim=-1)
        attn_weights=self.dropout(attn_weights)
        # attn weights - (b, n_heads,input_tokens,input_tokens)
        # values- (b, 1,input_tokens, head_dim)
        # attn_weights @ values => (b, n_heads,input_tokens, head_dim)
        context_vec=(attn_weights @ values).transpose(1,2)
        # shape of context vec => (b, input_tokens, n_heads, head_dim)
        context_vec=context_vec.contiguous().view(b,input_tokens,self.d_out)
        context_vec=self.out_proj(context_vec)
        return context_vec
```

usage:

```python
torch.manual_seed(123)
mqa=MultiQueryAttention(d_in,d_out,config['context_length'],config['dropout'],8)
context_vecs=mqa(batch)
```

# Group Query Attention

please read this awesome explanatory blog for the theory - https://www.ibm.com/think/topics/grouped-query-attention

## Code

```python
class GroupedQueryAttentionV2(nn.Module):
    def __init__(self,d_in,d_out,context_length,dropout,n_q_heads,n_kv_heads,qkv_bias=False):
        # n_q_heads -> number of query heads
        # n_kv_heads -> number of key/value heads  or number of groups
        # n_q_heads//n_kv_heads implies the number of query heads that share a key and value head
        super().__init__()
        assert d_out % n_q_heads == 0, "output dimension must be divisible by number of heads"
        assert n_q_heads%n_kv_heads==0, "number of query heads must be divisible by number of kv heads to form groups"
        self.d_out=d_out
        self.n_q_heads=n_q_heads
        self.n_kv_heads=n_kv_heads
        self.groupsize=self.n_q_heads//self.n_kv_heads
        self.head_dim=self.d_out//self.n_q_heads
        self.W_query = nn.Linear(d_in, self.d_out, bias=qkv_bias)
        self.W_key = nn.Linear(
            d_in,
            n_kv_heads * self.head_dim
        )
        self.W_value = nn.Linear(
            d_in,
            n_kv_heads * self.head_dim
        )
        self.out_proj=nn.Linear(d_out,d_out) # linear layer we use to combine our individual head outputs
        self.dropout=nn.Dropout(dropout)
        self.scale = self.head_dim ** -0.5
        # Non-trainable tensors that are still part of model state.
        self.register_buffer(
            "mask",
            torch.triu(
                torch.ones(context_length, context_length, dtype=torch.bool),
                diagonal=1
            )
        )
    def forward(self,x):
        # batch size, number of input tokens, input dimension
        b,input_tokens,d_in=x.shape
        # calculate query , key ,value projections
        queries,keys,values=self.W_query(x),self.W_key(x),self.W_value(x)
        # queries -> b, input_tokens, d_out == b, input_tokens, n_q_heads*head_dim
        # keys, values of b, input_tokens, n_kv_heads*head_dim
        # split queries keys and values into different heads - only the last dimension
        queries=queries.view(b,input_tokens,self.n_q_heads,self.head_dim)
        values=values.view(b,input_tokens,self.n_kv_heads, self.head_dim)
        keys=keys.view(b,input_tokens,self.n_kv_heads,self.head_dim)
        # transpose 1 and 2 dimension
        queries=queries.transpose(1,2)
        values=values.transpose(1,2)
        keys=keys.transpose(1,2)
        # now  queries has (batch size, n_q_heads,input_tokens, head_dim)
        # values, keys (batch size, n_kv_heads,input_tokens, head_dim)
        # lets split the queries into (batch size, n_kv_heads, group_size, input_tokens,head_dim)
        queries=queries.reshape(b,self.n_kv_heads,self.groupsize,input_tokens,self.head_dim)
        # lets add one another dimension to our key and value matrices, final dimension of keys and values =
        # (batch_size, n_kv_heads, 1 , input_tokens, head_dim)
        values=values.unsqueeze(2)
        keys=keys.unsqueeze(2)
        # (batch size, n_kv_heads, group_size, input_tokens,head_dim) @ (batch_size, n_kv_heads, 1 , head_dim, input_tokens)
        attn_scores=queries@keys.transpose(-2,-1)
        # above multiplication fares out as -> (b,  n_kv_heads, group_size,input_tokens,head_dim) @ (b, n_kv_heads, 1,head_dim,input_tokens) => (b,  n_kv_heads, group_size,input_tokens,input_tokens)
        mask_bool=self.mask[:input_tokens,:input_tokens]
        attn_scores.masked_fill_(mask_bool,-torch.inf)
        attn_weights=torch.softmax(attn_scores*self.scale,dim=-1)
        attn_weights=self.dropout(attn_weights)
        # values-  (batch_size, n_kv_heads, 1 , input_tokens, head_dim)
        # attn_weights @ values => (b, n_kv_heads, group_size,input_tokens, head_dim)
        context_vec=(attn_weights @ values).reshape(b,self.n_q_heads,input_tokens,self.head_dim).transpose(1,2)
        # shape of context vec => (b, input_tokens, n_q_heads, head_dim)
        context_vec=context_vec.contiguous().view(b,input_tokens,self.d_out)
        context_vec=self.out_proj(context_vec)
        return context_vec
```

usage remains the same as above

The aim of this blog is to provide code level view of few attention implementations

there are many other implementations for attention and i will write about those when i learn about them.
