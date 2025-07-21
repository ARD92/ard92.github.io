---
layout: post
title: prompting with langchain
tags: ai_ml
---


## Initiate Gemini LLM with langchain 
```
import getpass
import os

if "GOOGLE_API_KEY" not in os.environ:
    os.environ["GOOGLE_API_KEY"] = getpass.getpass("YOUR_API_KEY")

from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(
    model="gemma-3-12b-it",
    temperature=0,
    max_tokens=None,
    timeout=None,
    max_retries=2,
    # other params...
)
```

multiple types of prompting is possible for which prompt-completion templates have to be defined. prompting helps in 
1. zero shot
2. one shot
3. Few shot 

## Creating multiple prompts 

```
from langchain import LLMChain
from langchain import PromptTemplate

template = """<s><|user|> create a title for story about {summary}. only return title. <|end|> <|assistant|>"""

title_prompt=PromptTemplate(template=template, input_variable=["summary"])
title = LLMChain(llm=llm, prompt=title_prompt, output_key="title")
title.invoke({"summary": " a girl that lost her mother"})

template = """<s><|user|>
Describe the main character of a story about {summary} with the title {title}. Use only two sentences.<|end|>
<|assistant|>"""
character_prompt = PromptTemplate(
    template=template, input_variables=["summary", "title"]
)
character = LLMChain(llm=llm, prompt=character_prompt, output_key="character")


template = """<s><|user|>
Create a story about {summary} with the title {title}. The main character is: {character}. Only return the story and it cannot be longer than one paragraph. <|end|>
<|assistant|>"""

story_prompt = PromptTemplate(
    template=template, input_variables=["summary", "title", "character"]
    )

#story = LLMChain(llm=llm, prompt=story_prompt, output_key="story")
story = story_prompt|llm

llm_chain = title | character | story

llm_chain.invoke("a girl that lost her mother")

```
>/tmp/ipython-input-3-1452160459.py:7: LangChainDeprecationWarning: The class `LLMChain` was deprecated in LangChain 0.1.17 and will be removed in 1.0. Use :meth:`~RunnableSequence, e.g., `prompt | llm`` instead.
  title = LLMChain(llm=llm, prompt=title_prompt, output_key="title")
AIMessage(content="The Empty Chair sat sentinel at the head of the dinner table, a constant, aching reminder of what Elara had lost. Sixteen and perpetually quiet, she moved through her days in a muted haze, clinging to the predictable rhythm of her home – the dusting of antique clocks, the watering of her mother’s orchids, the faint, lingering scent of lavender that clung to the curtains and Elara’s own clothes. Each evening, she’d set a plate for her mother, a ghost of a habit she couldn't break, the silence amplifying the absence, the empty chair a physical manifestation of the gaping hole in her world. Memories, sharp and bittersweet, flickered constantly – her mother’s laughter, her warm hand in Elara’s, the way she always knew exactly what to say – and Elara found herself adrift, struggling to reconcile the vibrant world she knew with the stark, altered reality where her mother’s chair remained stubbornly, heartbreakingly, empty.", additional_kwargs={}, response_metadata={'prompt_feedback': {'block_reason': 0, 'safety_ratings': []}, 'finish_reason': 'STOP', 'model_name': 'gemma-3-12b-it', 'safety_ratings': []}, id='run--bfa60474-778e-452d-b82d-4bb35934c46c-0', usage_metadata={'input_tokens': 124, 'output_tokens': 0, 'total_tokens': 124, 'input_token_details': {'cache_read': 0}}) 



## Chaining prompts

langchain provides an easy way to chain multiple prompts to construct few shot prompts. The template has to be passed to the LLM and a chain can be created further 

```
from langchain import LLMChain
from langchain import PromptTemplate

template = """<s><|user|> create a title for story about {summary}. only return title. <|end|> <|assistant|>"""

title_prompt=PromptTemplate(template=template, input_variable=["summary"])
title = LLMChain(llm=llm, prompt=title_prompt, output_key="title")
title.invoke({"summary": " a girl that lost her mother"})

template = """<s><|user|>
Describe the main character of a story about {summary} with the title {title}. Use only two sentences.<|end|>
<|assistant|>"""
character_prompt = PromptTemplate(
    template=template, input_variables=["summary", "title"]
)
character = LLMChain(llm=llm, prompt=character_prompt, output_key="character")


template = """<s><|user|>
Create a story about {summary} with the title {title}. The main character is: {character}. Only return the story and it cannot be longer than one paragraph. <|end|>
<|assistant|>"""

story_prompt = PromptTemplate(
    template=template, input_variables=["summary", "title", "character"]
    )

#story = LLMChain(llm=llm, prompt=story_prompt, output_key="story")
story = story_prompt|llm

llm_chain = title | character | story

llm_chain.invoke("a girl that lost her mother")
```


## References
- https://python.langchain.com/docs/how_to/few_shot_examples/
- https://python.langchain.com/docs/concepts/few_shot_prompting/
- https://www.promptingguide.ai/ 


