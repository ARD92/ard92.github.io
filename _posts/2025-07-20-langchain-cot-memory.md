---
layout: post
title: Using Langchain to use COT and Conversation memory on gemini 
tags: ai_ml
---
Google provides collab to try gemini models and experimenting within the google env

## Install Packages
```
!pip install -U langchain-google-genai
!pip install langchain-community langchain-core
!pip install llama-cpp-python
```
## Google gemini API key and load model
```
import os
import google.generativeai as genai

os.environ['GOOGLE_API_KEY'] = "<YOUR_KEY>"
genai.configure(api_key = os.environ['GOOGLE_API_KEY'])

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
### Try a prompt
```
messages = [
    ("you are a jncie engineer, help configure BGP on a router")
]
ai_msg = llm.invoke(messages)
print(ai_msg)
```
>Okay, let's configure BGP on a router. I'll guide you through a common scenario and provide a detailed configuration.  I'll assume we're working with Cisco IOS (the most common), but the principles apply to other vendors with slight syntax variations.\n\n**Scenario:**\n\nWe have two Autonomous Systems (AS) connected.  Let's call them AS65001 and AS65002.  We'll configure BGP on a router in AS65001 to peer with a router in AS65002.  The goal is to exchange routing information between these two ASes.\n\n**Assumptions:**\n\n*   **Router Name:** `Router-AS65001` (This is the router we'll configure)\n*   **AS Number:** 65001 (For Router-AS65001)\n*   **Neighbor IP Address:** 192.168.1.2 (The IP address of the router in AS65002)\n*   **Neighbor AS Number:** 65002 (The AS number of the neighbor)\n*   **Network to Advertise:** 10.1.1.0/24 (This is a network within AS65001 that we want to advertise to AS65002)\n*   **Interface:** `GigabitEthernet0/0` (The interface connected to the neighbor)\n\

## Chain of Thought 

```
from langchain import LLMChain
from langchain import PromptTemplate

template = """<s><|user|> explain how a network engineer would troubleshoot {connectivity_type} issue. <|end|> <|assistant|>"""

debug_prompt=PromptTemplate(template=template, input_variables=["connectivity_type"])
debug = debug_prompt|llm

template = """<s><|user|>provide detailed steps about the {connectivity_type} issue and present {debug} as bullet points<|end|><|assistant|>"""
bullet_prompt = PromptTemplate(
    template=template, input_variables=["connectivity_type", "debug"]
)
bullet = bullet_prompt|llm

connectivity_type = "BGP protocol flap"

debug_result = debug.invoke({"connectivity_type": connectivity_type})


final = bullet.invoke({
    "connectivity_type": connectivity_type,
    "debug": debug_result
})
```

### Pretty print the output 
The library is used for formatting content. 

```
from rich.console import Console
console = Console()
console.print(final.content)
```

## Conversations and Memory 
LLM by default doesnt remember. need to have conversation buffer and conversation summary
conversation buffer -> replay the entire conversation history into the prompt. In langchain use `ConversationBufferMemory`

```
template = """<s><|user|>Current conversation: {chat_history} {input_prompt}<|end|><|assistant|>"""

prompt = PromptTemplate(
    input_variables=["chat_history", "input_prompt"], template=template
)

from langchain.memory import ConversationBufferMemory
memory = ConversationBufferMemory(memory_key="chat_history")

llm_chain = LLMChain(prompt=prompt, llm=llm, memory=memory)

llm_chain.invoke({"input_prompt": "What is the meaning of life?"})
```

> {'input_prompt': 'What is the meaning of life?',
 'chat_history': '',
 'text': 'Okay, that\'s *the* big question! There\'s no single, universally accepted answer to "What is the meaning of life?" It\'s a question that philosophers, theologians, scientists, and individuals have grappled with for centuries. Here\'s a breakdown of different perspectives, grouped into categories, to give you a comprehensive overview:\n\n**1. Philosophical Perspectives:**\n\n*   **Nihilism:** This view suggests that life is inherently without objective meaning, purpose, or intrinsic value. It doesn\'t necessarily mean despair; it can be liberating, as it frees you to create your own meaning.\n*   **Existentialism:**  Existentialists (like Jean-Paul Sartre and Albert Camus) believe that life has no pre-ordained meaning. We are born into existence, and it\'s *our* responsibility to create meaning through our choices and actions.  We are "condemned to be free."  Camus, in particular, explored the "absurdity" of the human condition – the conflict between our desire for meaning and the meaningless universe – and advocated for rebellion and embracing life despite this absurdity.\n*   **Absurdism:** Closely related to Existentialism, Absurdism focuses on the inherent conflict between humanity\'s search for meaning and the universe\'s lack of it.  It suggests accepting this absurdity and finding joy and purpose within it.\n*   **Hedonism:**  The pursuit of pleasure and avoidance of pain is the ultimate goal and meaning of life. This can range from simple physical pleasures to more complex intellectual and emotional satisfactions.\n*   **Stoicism:**  Finding meaning through virtue, reason, and living in accordance with nature.  Focusing on what you *can* control (your thoughts and actions) and accepting what you cannot.  Inner peace and tranquility are key.\n*   **Utilitarianism:**  The meaning of life is to maximize happiness and well-being for the greatest number of people.  Actions are judged based on their consequences.\n\n**2. Religious/Spiritual Perspectives:**\n\n*   **Abrahamic Religions (Judaism, Christianity, Islam):**  Life\'s meaning is often tied to serving and worshipping God, following divine commandments, and striving for a relationship with the divine.  Often involves a belief in an afterlife and a purpose beyond earthly existence.\n*   **Hinduism:**  The goal is to achieve *moksha* (liberation) from the cycle of birth and death (samsara) through dharma (righteous conduct), karma (action and consequence), and ultimately, realizing the unity of the individual self (Atman) with the ultimate reality (Brahman).\n*   **Buddhism:**  The meaning of life is to end suffering (dukkha) by understanding the nature of reality, practicing mindfulness, and following the Eightfold Path, ultimately achieving enlightenment (Nirvana).\n*   **Other Spiritual Beliefs:** Many other spiritual traditions emphasize connection to something larger than oneself, finding purpose through service, compassion, and personal growth.\n\n**3. Scientific Perspectives:**\n\n*   **Evolutionary Biology:** From a purely biological standpoint, the "meaning" of life is survival and reproduction – passing on your genes to the next generation.  This doesn\'t offer a subjective meaning, but explains a fundamental drive.\n*   **Neuroscience:**  Neuroscience explores the biological basis of consciousness, emotions, and motivations. While it doesn\'t define meaning, it can shed light on *why* we seek it and what experiences bring us joy and fulfillment.\n*   **Cosmology:**  The vastness and age of the universe can lead to a sense of insignificance, but also to awe and wonder.  Some find meaning in understanding our place within the cosmos.\n\n**4. Personal/Subjective Perspectives:**\n\n*   **Relationships:**  Finding meaning through connection with others – family, friends, romantic partners, community.\n*   **Contribution:**  Making a positive impact on the world, whether through work, volunteering, creativity, or activism.\n*   **Personal Growth:**  Continuously learning, developing skills, and striving to become a better version of yourself.\n*   **Experiences:**  Seeking out new and enriching experiences – travel, art, music, nature.\n*   **Creating Something:**  Expressing yourself through art, writing, music, or any other creative endeavor.\n*   **Simply Being:**  Appreciating the present moment and finding joy in the simple things.\n\n\n\n**Ultimately, the meaning of life is often what *you* choose to make it.** It\'s a deeply personal and evolving journey.  It\'s okay to question, to explore different perspectives, and to redefine your meaning as you grow and change.\n\nTo help me tailor my response further, could you tell me:\n\n*   Are you leaning towards a particular perspective (e.g., philosophical, religious, scientific)?\n*   What prompted you to ask this question? Are you feeling lost, curious, or something else?'}

### Try asking a question related to the output 
```
llm_chain.invoke({"input_prompt": "what was bullet 1 in the earlier reference ?"})
```
> output will be 
> /*history truncated*/ 
> 'text': 'Bullet 1 in the earlier reference was **Philosophical Perspectives**.'}

### Pass Top K conversations 

Instead of passing entire history, pass only top k conversations. This is called windowed buffer approach 

```
from langchain.memory import ConversationBufferWindowMemory
memory = ConversationBufferWindowMemory(k=2, memory_key="chat_history")

template = """<s><|user|>Current conversation: {chat_history} {input_prompt}<|end|><|assistant|>"""

prompt = PromptTemplate(
    input_variables=["chat_history", "input_prompt"], template=template
)

llm_chain = LLMChain(prompt=prompt, llm=llm, memory=memory)

llm_chain.invoke({"input_prompt": "my name is james, and what is your name ?"})
```

following this try 

```
llm_chain.invoke({"input_prompt": "what is my name?"})
```

While this works by passing samples and if answer not known in history, llm would glitch and not answer accurately. instead we can use conversation summary so that we dont run into token limits. This will work by passing entire history as a summary instead of asking for a higher context window size

### Summarization templates
prepare a summarization template

```
summary_prompt_template = """<s><|user|>Summarize the conversations and update
    with the new lines.
    Current summary: {summary}
    new lines of conversation:  {new_lines}
    New summary:<|end|>
    <|assistant|>"""

summary_prompt = PromptTemplate(input_variables=["new_lines", "summary"],
                                template=summary_prompt_template
    )

template = """<s><|user|>Current conversation: {chat_history} {input_prompt}<|end|><|assistant|>"""

prompt = PromptTemplate(
    input_variables=["chat_history", "input_prompt"], template=template
)

from langchain.memory import ConversationSummaryMemory
memory = ConversationSummaryMemory(llm=llm, memory_key="chat_history", prompt=summary_prompt)
llm_chain = LLMChain(prompt=prompt, llm=llm, memory=memory)

llm_chain.invoke({"input_prompt": "my name is james, whats yours ?"})
```

> {'input_prompt': 'my name is james, whats yours ?',
 'chat_history': '',
 'text': "My name is Gemini. It's nice to meet you, James!"}

load most recent memory

```
memory.load_memory_variables({})
```