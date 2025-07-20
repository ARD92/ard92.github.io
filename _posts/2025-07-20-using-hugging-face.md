---
layout: post
title: Using Hugging face to load language models
tags: ai_ml
---

Hugging face provides capabilities to load various models in order to infer or train. 

```
!pip install numpy
!pip install torch 
!pip install transformers
!pip install flash-attn
!pip install accelerate
!pip install --upgrade transformers torch
```

## Import modules
```
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline
```

## Load models
```
model = AutoModelForCausalLM.from_pretrained(
    "microsoft/Phi-4-mini-instruct",
    #device_map="auto",
    torch_dtype="auto",
    trust_remote_code=True,
)
```
> Loading checkpoint shards: 100%|██████████| 2/2 [00:00<00:00,  8.51it/s]

## Create a pipeline
```
pipe = pipeline(
    "text-generation",
    model = model,
    tokenizer = tokenizer,
    return_full_text = False,
    max_new_tokens = 500,
    do_sample = False,
)
```

## Try an example
```
messages = [
    {"role":"user",
     "content": "explain what hiphop is one sentence."}
]

output = pipe(messages)
print(output)
```
> The following generation flags are not valid and may be ignored: ['temperature']. Set `TRANSFORMERS_VERBOSITY=info` for more details.
> [{'generated_text': 'Hiphop is a cultural movement and genre of music that originated in the 1970s, characterized by its rhythmic beats, lyrical expression, and associated styles of dance and fashion.'}]

## Try a prompt 

```
singer_prompt = [
    {"role": "user", "content": "you are a rapper who writes single line rhymes"}
    ]
outputs = pipe(singer_prompt)
singer_description = outputs[0]["generated_text"]
print(singer_description)
```
> The following generation flags are not valid and may be ignored: ['temperature']. Set `TRANSFORMERS_VERBOSITY=info` for more details.
> I spit bars like a master, rhymes that last,
> Crafting lines that hit hard, never second to last.
> Flow's smooth, the words tight, a lyrical delight,
> Dropping bars that make the crowd go wild, all night.

## Extend the above to another prompt ?

```
sales_prompt = [
    {"role": "user", "content": f"Generate a very short sales pitch for the following product: '{singer_description}'"} 
    ]

outputs = pipe(sales_prompt)
sales_pitch = outputs[0]["generated_text"] 
print(sales_pitch)
```

> The following generation flags are not valid and may be ignored: ['temperature']. Set `TRANSFORMERS_VERBOSITY=info` for more details.
> Discover the future of communication with LexiBot – your AI conversation partner. Say goodbye to language barriers and hello to seamless, intelligent interactions. LexiBot is here to unlock the power of language for you. Try LexiBot today and experience the revolution in conversation!

## References 
- hands on large language models book
- https://huggingface.co/docs/transformers/index 