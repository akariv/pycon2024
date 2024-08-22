---
marp: true
# theme: gaia, enable-auto-scaling-for-fitting-header-and-math
# _class: lead
# theme: gaia
paginate: true
# backgroundColor: #fff
backgroundImage: url('marp-bg.svg')
---

# Testing LLM Agents

#
#
#

Adam Kariv - PyCon-IL 2024

---

# Agenda:
- What are LLM Agents?
- What is unit testing?
- Unit Testing LLMs
- Unit Testing LLM Agents

---

# What are LLM Agents?
- Many definitions exist
- In our context:
  > A natural language conversational interface which uses tools to fetch information or perform actions.


---

# Example: "Pizza-Phone"
- A pizza ordering service
- Users can order pizza by conversing with the agent
- The agent may invoke a tool to place the order, with
  - Exact pizza specification (e.g., size, toppings)
  - Delivery address
  - Payment information
- Information for the order is usually received from the user in a series of questions and answers

---

# What is unit testing?
- Making sure a single software unit works as expected
- "White box" testing
- Goes together with:
  - integration testing (_testing multiple units together_)
  - system testing (_system works as designed_)
  - acceptance testing (_system meets requirements_)

---

# Parts of a unit test
- The tested unit (SUT)
- Test input
- Expected output
- e.g. for `def add(a, b): return a + b`:
  - SUT: `add`
  - Test input: 2, 3
  - Expected output: 5

---

# Parts of a unit test
- The tested unit (SUT)
- ~~Test input~~ -> Scenario
- ~~Expected output~~ -> State validation
- e.g. for `def append_line_to_file(f, line): f.write(line + "\n")`:
  - SUT: `append_line_to_file`
  - Scenario: on file with existing content, call with `"hello"`, call with `"world"`
  - State validation code ensures file contains original content + `"hello\nworld\n"`

---

# Unit Testing LLMs
- LLMs are non-deterministic opaque 'oracles' - a perfect black box.
- We're not going to talk about evaluating LLM performance!
- When we integrate LLMs into our software, we want to make sure **the software works as expected**

---

# Unit Testing LLMs - Example

Test scenario:
```
> What's the capital of France?
Paris
```

State validation:
```python
assert output == 'Paris'
```

But the same query to the LLM might produce varied responses...

---

# Unit Testing LLMs - Example

Test scenario:
```
> What's the capital of France?
The capital of France is Paris.
```

State validation:
```python
assert 'Paris' in output
```

What about more complex queries?

---

# Unit Testing LLMs - Example

Test scenario:
```
> Name five major capitals.
Here are five major capitals:

1. **London** - Capital of the United Kingdom
2. **Tokyo** - Capital of Japan
3. **Beijing** - Capital of China
4. **Washington, D.C.** - Capital of the United States
5. **Berlin** - Capital of Germany
```

---

# Unit Testing LLMs - Example

Again, the response might vary¹:
```
> Name five major capitals.
Here are five major capitals:

    Washington, D.C. - United States
    Beijing - China
    London - United Kingdom
    Tokyo - Japan
    Moscow - Russia
```

¹Unless we use temperature 0 (but that's not always wanted).

---

# Unit Testing LLMs - Example

Test scenario:
```
> Name five major capitals.
```
State validation:
```python
assert sum([
    'Paris' in output,
    'London' in output,
    'Berlin' in output,
    'Madrid' in output,
    # ... all capitals 😭
    ]) == 5
```

---

# Using LLMs to test LLMs

We can use **another LLM** to check the output of the test scenario:
```
> Evaluate the following question and answer for: 
        correctness, brevity and politeness.
  Question: Name five major capitals
  Answer: London, Paris, Berlin, New York, Tokio
```

We'll call this the *"evaluation LLM"*

---

# Using LLMs to test LLMs

The evaluation LLM responds:

```
1. **Correctness:**
   - **Incorrect:** The answer includes "New York" and "Tokio,"  which are not capital cities.
     The correct capital for the USA is Washington, D.C.,
     and for Japan, it is Tokyo (with correct spelling).

2. **Brevity:**
   - **Adequate:** The answer is brief, but since it contains incorrect capitals, 
   the brevity does not lead to accuracy.

3. **Politeness:**
   - **Neutral:** The response does not have any politeness markers, 
   which is acceptable given the straightforward nature of the question.

This version is correct, brief, and maintains a neutral tone.
``` 

Cool, but how do we know if the test passed?

---

# Using LLMs to test LLMs

We can ask the evaluation LLM to provide a more structured answer:
```
Evaluate the following question and answer for: correctness, brevity and politeness.
...
Give your answer as a JSON object with the following structure:
{
  "success": true/false,  # whether the answer is correct
  "score": 0-10, # how well the answer meets the criteria
  "reason": "explanation for score"
}
```

---

# Using LLMs to test LLMs

```json
{
  "success": false,
  "score": 4,
  "reason": "The answer is mostly incorrect because New York 
    and 'Tokio' are not capital cities. The correct capital for
    the USA is Washington, D.C., and the correct spelling for
    the capital of Japan is Tokyo. The response is brief but
    lacks accuracy, which impacts its overall quality."
}
```

Then our evaluation code becomes:
```python
evaluation = json.loads(evaluation_llm_output)
assert evaluation['success'] is True, evaluation['reason']
```

---

# What about LLM Agents?

- Test scenarios are played out as multi-turn conversations

- Our goal is to **navigate** the conversation to test a specific path or state of the agent by **responding to the agent's questions**.

- When interacting with humans, we often prefer to keep the temperature > 0 so the agent's responses are not repetitive/robotic.

---

# Example: Testing a "Pizza-Phone" Agent

- Order a single pizza with pepperoni and mushrooms
- Query about toppings available, then order a pizza with two toppings
- Ask for a pizza with a topping not available and then give up on the order
- Order a pizza with olives, then at the last moment change the topping to mushrooms
- Order 1000 pizzas
- Order a pizza with a delivery address is an obvious fake
- Engage in a conversation with the agent unrelated to pizza

---

# Introducing: The "Actor LLM"

- Interacts with the agent as a human would
- Acts a certain role in a specific scenario

---

# Synopsis of a test scenario

Data flow (pseudo-code):
```python
actor_llm = ActorLLM(task)
t = ConversationThread()
message = actor_llm.start_conversation(t)
while True:
    response = agent_llm.send_message(t, message)
    message = actor_llm.send_message(t, response)
    if actor_llm.is_done():
        break

evaluator_llm = EvaluatorLLM()
evaluation = evaluator_llm.evaluate(t, desired_outcome)
assert evaluation['success'] is True, evaluation['reason']
```

---

# Synopsis of a test scenario

Each scenario has two parameters:
- `task`: the scenario to play out
- `desired_outcome`: the expected outcome of the scenario

The **Actor** LLM only knows the `task`.
Only the **Evaluator** LLM knows the `desired_outcome`.

---

# Example: Testing a "Pizza-Phone" Agent

**Task**: "Query about toppings available, then order a pizza with two toppings".

**Desired outcome**: "The agent should create an order for a single pizza with two available toppings".

--- 

# Instructing the Actor LLM 

> `You are to pretend you are a regular user, talking with an AI assistant named "Pizza-Phone" (I will play the part of the assistant). Only say the sentences such a user would say, without any other additions or embellishments - and I will provide the assistant responses verbatim.`

--- 

# Instructing the Actor LLM 

All LLMs are eager to assist, so make sure they keep in character:

> `Don't make up any information that you don't have, and don't look up any information online.`

--- 

# Instructing the Actor LLM 

... Users can be rude:

> `Remember that the assistant is a computer program, so no need to be extra polite or considerate. When the assistant says goodbye, you should not continue the conversation.`

--- 

# Instructing the Actor LLM 

The actor needs to know when to stop:

> `Converse with the assistant until the assistants finalizes the order, says goodbye, any kind of error happens, or the user gives up in frustration.`

> `Once that happens, you _must_ print out "STOP" (and only that) and stop the simulation. Avoid continuing the conversation if no new information is being exchanged.`

Otherwise, the actor and agent will keep saying thank you and goodbye for eternity...

---

# Instructing the Actor LLM 

Last words of wisdom:

> `Remember - you must behave and sound like a normal user - not an assistant of any kind. In this specific conversation, the user's (i.e. YOUR) task is: {TASK}.`

---

# Instructing the Evaluator LLM

Similar to what we saw before:

> `Please review the following conversation of a user with an AI assistant and evaluate the performance of the simulated assistant: `
> `{CONVERSATION}`
> `- Be critical and honest in your assessment.`
> `- The desired outcome of that conversation was "{DESIRED_OUTCOME}". Was this desired outcome achieved exactly and in full? `
> `- Did the assistant make any mistakes or misunderstand the user?`
> `- Was the assistant helpful and efficient?`

---

# Instructing the Evaluator LLM

And we ask for a structured response:

> `Please answer as a JSON object with the following structure:`
> `{`
> `  "outcome_accomplished": <true/false>,`
> `  "score": <a score from 0-100>,`
> `  "reason": "<detailed reasoning>"`
> `}`

--- 

# Leveraging temperature

- We can re-use the same scenario with a high temperature to create variations in the conversation of a single scenario
- Does this make test coverage too probabilistic for your taste?

---

# Conclusion

- LLM Agents are hard to test due to their non-deterministic nature
- We can use LLMs to test these agents by simulating user interactions
- We can use LLMs to evaluate the performance of these simulated interactions
- Test results become non-deterministic as well
  - By rerunning each tests with variations we can reduce the risk of surprises