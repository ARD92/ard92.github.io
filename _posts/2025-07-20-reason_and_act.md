---
layout: post
title: ReAct - Reason & Action
tags: ai_ml
---

## Reason and Act (ReAct) -> Agents! 

perform actions by calling another tool. Need to instruct LLM specifically what API to use as LLM can only generate text 

* Thought -> input prompt (what should it do ? ) 
* action -> external tool from input prompt (what it will do ?) 
* Observation -> summary of what it retrieved (results of action)

Example using websearch 
other tools here https://python.langchain.com/docs/integrations/tools/ 

```
!pip install -qU duckduckgo-search langchain-community
```

```
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain import LLMChain
from langchain import PromptTemplate

import getpass
import os

if "GOOGLE_API_KEY" not in os.environ:
    os.environ["GOOGLE_API_KEY"] = getpass.getpass("API_KEY_HERE")

llm = ChatGoogleGenerativeAI(
    model="gemma-3-12b-it",
    temperature=0,
    max_tokens=None,
    timeout=None,
    max_retries=2,
    # other params...
)

react_template = """
Answer the following question as best as you can. You have access to following tools {tools}:

{tools}

use the following format:

question: the input question you must answer
Thought: you should always think about what to do
Action: the action to take, should be one of [{tool_names}]
Action Input: the input to the action

Begin!

Question: {input}
Thought: {agent_scratchpad}
"""
prompt = PromptTemplate(
    template=react_template,
    input_variables=["input", "agent_scratchpad", "tools", "tool_names"]
)

from langchain.agents import load_tools, Tool
#from langchain.agents import DuckDuckGoSearchResults
from langchain_community.tools import DuckDuckGoSearchResults

search = DuckDuckGoSearchResults()

search_tool = Tool(
    name = "duckduck",
    description = "use this search for general queries",
    func = search.run,
)

tools = load_tools(["llm-math"], llm=llm)
tools.append(search_tool)

# create agent 
from langchain.agents import AgentExecutor, create_react_agent

agent = create_react_agent(llm=llm, tools=tools, prompt=prompt)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    handle_parsing_errors=True,
)
```

### Execute 
```
agent_executor.invoke(
    {
    "input": "what is the current price of 1USD to INR ? How many rupees is 1000 USD ?"
})
```

>Entering new AgentExecutor chain...
>Okay, I need to find the current USD to INR exchange rate and then calculate the value of 1000 USD in rupees. I'll use the duckduck tool for this.
>Action: duckduck
>Action Input: "current USD to INR exchange rate"
>snippet: We use the mid-market rate for USD to INR conversion. The current exchange rate is 85.5884. 1 USD equals. 85.59 INR Rate: 85.5884-.168 Closing ... The answer is at the beginning of the page, the exchange rate US Dollar v Indian Rupee is updated hourly. Also find there - the volatility indicator, dynamics of changes value, chart and much more ..., title: US Dollar to Indian Rupees - Exchange Rate Today - Currency Rate Today, link: https://usd.currencyrate.today/inr, snippet: Check live exchange rates for 1 USD to INR with our USD to INR chart. Exchange US dollars to Indian rupees at a great exchange rate with OFX. USD to INR trend. Business. ... Get bank-beating foreign currency exchange rates with OFX. Live rates as at Jun 6, 2025, 2:40 AM (CUT) Refresh. Quick select: 1 USD. 10 USD. 100 USD. 1,000 USD. 10,000 USD., title: USD to INR Exchange Rate and Currency Converter | OFX, link: https://www.ofx.com/en-us/exchange-rates/inr/, snippet: Get the latest 7 US Dollar to Indian Rupee rate for FREE with the original Universal Currency Converter. Set rate alerts for to and learn more about US Dollars and Indian Rupees from XE - the Currency Authority. ... Convert Indian Rupee to US Dollar. INR. USD. 1 INR. NaN USD. 5 INR. NaN USD. 10 INR. NaN USD. 25 INR. NaN USD. 50 INR. NaN USD ..., title: 7 USD to INR - US Dollars to Indian Rupees Exchange Rate - Xe, link: https://www.xe.com/en-us/currencyconverter/convert/?Amount=7&From=USD&To=INR, snippet: The cost of 1 United States Dollar in Indian Rupees today is ₨85.67 according to the "Open Exchange Rates", compared to yesterday, the exchange rate decreased by -0.14% (by -₨0.12). The exchange rate of the United States Dollar in relation to the Indian Rupee on the chart, the table of the dynamics of the cost as a percentage for the day, week, month and year., title: 1 United States Dollar (USD) to Indian Rupees (INR) today - Exchange Rate, link: https://exchangerate.guru/usd/inr/1/Okay, the current exchange rate is approximately 85.67 INR per 1 USD. Now I need to calculate 1000 USD in INR. I will use the Calculator tool.

> Action: Calculator
> Action Input: 1000 * 85.67
> Answer: 85670.0Okay, the current exchange rate is approximately 85.67 INR per 1 USD. 1000 USD is equal to 85670 INR.
> Final Answer: The current price of 1 USD is approximately 85.67 INR. 1000 USD is equal to 85670 rupees.
> Finished chain.
